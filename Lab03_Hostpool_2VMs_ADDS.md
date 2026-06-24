# Lab 03 — Host Pool com 2 VMs ingressadas em Active Directory Domain Services (AD DS)

> **Disciplina:** Azure Virtual Desktop — Pós-Graduação em Arquitetura Avançada em Azure
> **Modalidade:** Passo a passo via Portal do Azure (portal-first)
> **Resultado:** Cria-se um **AD DS próprio** (controlador de domínio em uma VM) sincronizado com o Entra ID, e um host pool com 2 hosts **domain-joined**. Esta estrutura é a base dos **Labs 04, 05 e 06**.

---

## Ficha do laboratório

| Item | Detalhe |
|------|---------|
| **Dificuldade** | ★★★ Avançado |
| **Tempo estimado** | 90–120 min |
| **Objetivo** | Construir um domínio AD DS (`avdlab.local`), sincronizá-lo com o Entra ID (Entra Connect), e provisionar um host pool com 2 session hosts ingressados no domínio. |
| **Pré-requisitos** | Lab 1.1 (VNet/sub-redes). Papel **Owner** ou (Contributor + User Access Administrator). |
| **Recursos consumidos** | 1× VM controlador de domínio (`Standard_B2s`/`D2s_v5`), 2× VM session host, Host Pool, Workspace, DAG. |
| **Entrega** | Domínio `avdlab.local` ativo, hosts domain-joined *Available* e usuário de domínio conectando ao AVD. |

### Cenário
Empresa com **Active Directory tradicional**. As VMs do AVD ingressam no domínio AD DS, e os usuários são sincronizados para o Entra ID (necessário porque a **autenticação do AVD sempre passa pelo Entra ID**). Por isso usamos **identidades híbridas**.

> 💡 **Atenção arquitetural:** o usuário que conecta ao AVD precisa existir nos **dois** lados — criado no AD DS e sincronizado para o Entra ID. Um usuário só-em-nuvem **não** consegue logar em host AD DS-joined.

### Convenção de nomes
| Recurso | Nome |
|---------|------|
| Domínio AD DS | `avdlab.local` |
| VM controladora de domínio | `vmdc-cin-01` (sub-rede `snet-adds-prd-cin-001`, 10.50.3.0/24, IP estático 10.50.3.4) |
| Host Pool | `vdpool-avd-prd-cin-002` |
| Workspace | `vdws-avd-prd-cin-001` (reutiliza, ou cria se não existir) |
| Desktop App Group | `vdag-avd-prd-cin-002` |
| Session hosts | `vmavda-cin` (gera `vmavda-cin-0`, `vmavda-cin-1`) |
| Unidade Organizacional alvo | `OU=AVD,DC=avdlab,DC=local` |

---

## Parte A — Provisionar a VM controladora de domínio

1. Barra de busca → **Virtual machines** → **+ Create → Azure virtual machine**.
2. Aba **Basics:**
   - **Resource group:** `rg-avd-prd-cin-001`.
   - **Virtual machine name:** `vmdc-cin-01`.
   - **Region:** Central India.
   - **Image:** **Windows Server 2022 Datacenter — x64 Gen2** (ou 2025).
   - **Size:** `Standard_B2s` (suficiente para lab) ou `Standard_D2s_v5`.
   - **Administrator account:** username `dcadmin` + senha forte (**anote**).
   - **Public inbound ports:** **None** (vamos usar Bastion ou RDP via rede interna; em lab você pode permitir 3389 temporariamente — não recomendado).
3. Aba **Networking:**
   - **Virtual network:** `vnet-avd-prd-cin-001`.
   - **Subnet:** `snet-adds-prd-cin-001` (10.50.3.0/24).
   - **Public IP:** crie um temporário só para o setup, ou use **Azure Bastion** (mais seguro).
4. **Review + create** → **Create**.

### A.1 — Fixar IP privado estático do DC (obrigatório)
Um DC **precisa** de IP fixo.
1. Aberta a VM → **Networking → Network settings** → clique na **NIC** → **IP configurations** → `ipconfig1`.
2. **Assignment: Static** → defina **10.50.3.4** → **Save**. A VM reiniciará a NIC.

---

## Parte B — Instalar a função AD DS e promover o DC

1. Conecte na `vmdc-cin-01` (RDP / Bastion) como `dcadmin`.
2. Abra **Server Manager** → **Manage → Add Roles and Features**:
   - **Role-based or feature-based installation** → selecione o servidor local.
   - Marque **Active Directory Domain Services** → **Add Features** → avance → **Install**.
3. Após instalar, no Server Manager clique na **bandeira de notificação** (⚑) → **Promote this server to a domain controller**.
4. **Deployment Configuration:** **Add a new forest** → **Root domain name:** `avdlab.local` → **Next**.
5. **Domain Controller Options:** defina a **DSRM password** (anote) → **Next**.
6. Avance pelas telas (DNS, NetBIOS = `AVDLAB`, paths) aceitando padrões → **Install**. A VM reinicia ao final.

### B.1 — Criar OU e usuários de domínio
1. Reconecte como `AVDLAB\dcadmin`.
2. **Server Manager → Tools → Active Directory Users and Computers**.
3. Botão direito em `avdlab.local` → **New → Organizational Unit** → nome `AVD`.
4. Dentro da OU `AVD` → **New → User** → crie `joao.teste` (User logon name `joao.teste@avdlab.local`) → defina senha e **desmarque** "User must change password at next logon", marque "Password never expires" (lab) → **Finish**.

---

## Parte C — Apontar o DNS da VNet para o DC

Para que os hosts encontrem o domínio, a VNet deve resolver DNS pelo DC.

1. Barra de busca → **Virtual networks → `vnet-avd-prd-cin-001`** → **Settings → DNS servers**.
2. **Custom** → adicione **10.50.3.4** (o DC) → **Save**.
   > As VMs já criadas precisam reiniciar a NIC para pegar o novo DNS; as novas (hosts) já nascerão com ele.

---

## Parte D — Sincronizar identidades com o Entra ID (Entra Connect)

A autenticação do AVD passa pelo Entra ID, então os usuários do AD DS precisam ser sincronizados.

1. Na `vmdc-cin-01` (ou numa VM membro dedicada), baixe e instale o **Microsoft Entra Connect** (antigo Azure AD Connect): site oficial da Microsoft → "Microsoft Entra Connect".
2. Execute o assistente:
   - **Express Settings** (lab) ou **Customize**.
   - Informe credenciais de **Global Administrator** do tenant Entra.
   - Informe credenciais de **Enterprise Admin** do AD (`AVDLAB\dcadmin`).
   - Selecione a OU `AVD` para sincronizar (em **Domain and OU filtering**).
   - Conclua. Force a sync se quiser:
     ```powershell
     Import-Module ADSync
     Start-ADSyncSyncCycle -PolicyType Delta
     ```
3. Valide no portal: **Microsoft Entra ID → Users** → `joao.teste` deve aparecer com **On-premises sync enabled = Yes**.

> 💡 **Verificação de UPN:** o sufixo UPN do usuário no AD (`@avdlab.local`) idealmente deve corresponder a um domínio **verificado** no Entra ID. Em lab, se usar `.local`, o Entra Connect ajusta o UPN para `@seudominio.onmicrosoft.com`. Anote qual UPN o usuário terá no Entra — é com ele que fará login no AVD.

---

## Parte E — Criar o Host Pool com os 2 hosts ingressados no AD DS

1. **Azure Virtual Desktop → Host pools → + Create**.
2. **Basics:**
   - **Resource group:** `rg-avd-prd-cin-001`; **Host pool name:** `vdpool-avd-prd-cin-002`; **Location:** Central India.
   - **Host pool type:** **Pooled**; **Algorithm:** Breadth-first; **Max session limit:** `5`; **Preferred app group type:** Desktop.
3. **Virtual Machines:** **Add virtual machines = Yes**:
   - **Name prefix:** `vmavda-cin`.
   - **Image:** Windows 11 Enterprise multi-session 24H2 (+ M365 opcional).
   - **Size:** `Standard_D2s_v5`; **Number of VMs:** `2`.
   - **Network:** `vnet-avd-prd-cin-001` / `snet-hosts-prd-cin-001`.
   - **Domain to join:** **Active Directory** (não "Microsoft Entra ID").
     - **AD domain join UPN:** uma conta com permissão de join no domínio, ex. `dcadmin@avdlab.local` (ou conta de serviço dedicada).
     - **Password:** senha da conta.
     - **Specify domain or unit:** marque e informe a OU alvo **`OU=AVD,DC=avdlab,DC=local`** (assim os hosts caem na OU certa, importante para GPO no Lab 04).
   - **Virtual Machine Administrator account:** `localadmin` + senha.
4. **Workspace:** **Register desktop app group = Yes** → workspace `vdws-avd-prd-cin-001` (criar se não existir).
5. **Review + create** → **Create**.

> O agente de join usa a conta informada para ingressar as VMs no `avdlab.local`, dentro da OU `AVD`.

---

## Parte F — Atribuir acesso ao usuário

1. **RBAC de login no SO:** ao contrário do cenário Entra-join, hosts AD DS-joined **não** exigem o papel "Virtual Machine User Login" para o login no SO (a autorização é via AD). Mesmo assim, o usuário precisa ter direito de logon (membro de *Remote Desktop Users*, que o AVD configura).
2. **Atribuir ao Application Group (obrigatório para o recurso aparecer):**
   - **Azure Virtual Desktop → Application groups → `vdag-avd-prd-cin-002`** → **Assignments → + Add** → selecione `joao.teste` (a identidade **sincronizada** no Entra) → **Select**.

---

## Parte G — Conectar e validar

1. **Host pools → `vdpool-avd-prd-cin-002` → Session hosts:** os 2 hosts devem ficar **Available**.
2. Abra o **Windows App** / web client → login com o **UPN do Entra** de `joao.teste` (ver nota da Parte D).
3. Abra o desktop publicado.
4. Dentro da sessão, valide o ingresso no domínio:
   ```cmd
   systeminfo | findstr /B /C:"Domain"
   whoami
   ```
   `Domain` deve ser `avdlab.local`; `whoami` deve retornar `avdlab\joao.teste`.

### Critérios de sucesso
- [ ] `vmdc-cin-01` promovida a DC do domínio `avdlab.local` com IP 10.50.3.4.
- [ ] VNet `vnet-avd-prd-cin-001` usa DNS 10.50.3.4.
- [ ] `joao.teste` sincronizado para o Entra ID.
- [ ] Os 2 hosts `vmavda-cin-0x` aparecem **Available** e na OU `AVD`.
- [ ] Login no AVD com a identidade híbrida; `whoami` = `avdlab\joao.teste`.

---

## Erros comuns

| Sintoma | Causa | Correção |
|---------|-------|----------|
| Host falha ao ingressar | DNS da VNet não aponta para o DC, ou credencial/UPN de join errada | Refaça Parte C; verifique a conta de join e a OU |
| Usuário não loga ("conta não existe") | Usuário não sincronizado para o Entra | Force `Start-ADSyncSyncCycle -PolicyType Delta` (Parte D) |
| Recurso não aparece no cliente | Falta atribuição no App Group | Refaça Parte F |
| Login pede senha mas falha | UPN usado no login difere do UPN no Entra | Use o UPN exato mostrado em Entra ID → Users |

---

## Importante — não destrua esta estrutura
Os **Labs 04 (imagem), 05 (FSLogix + private endpoints) e 06 (scaling plan)** reutilizam este domínio e host pool. Mantenha `vmdc-cin-01` e `vdpool-avd-prd-cin-002` ativos.

---

## Próximo lab
➡️ **Lab 04 — Criar imagem customizada de Windows 11** (idioma, teclado, fuso, GPOs) e publicá-la no Compute Gallery para reuso nesta estrutura AD DS.
