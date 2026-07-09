# Tuning e Melhores Práticas — Azure Virtual Desktop (FSLogix · GPO · Antivírus · Imagem)

> **Disciplina:** Azure Virtual Desktop — Pós-Graduação em Arquitetura Avançada em Azure
> **Uso:** documento de referência transversal — aplique junto com o **Lab 05** (FSLogix + AD DS) e o **Lab 06** (imagem/GPO). Cada item traz **o que faz**, **como configurar** e o **impacto** de ligar/deixar errado.

---

## Como aplicar as configurações do FSLogix

O FSLogix é configurado por **chaves de registro**, e o jeito gerenciado de aplicá-las é por **GPO** usando os **ADMX do FSLogix** (`fslogix.admx`/`.adml`) importados no domínio (ver Lab 06 Parte E). Caminhos de registro:

| Área | Chave |
|------|-------|
| Serviços do FSLogix (App Services) | `HKLM\SOFTWARE\FSLogix\Apps` |
| **Profile Container** (perfil do usuário) | `HKLM\SOFTWARE\FSLogix\Profiles` |
| **Office Container** (ODFC) | `HKLM\SOFTWARE\Policies\FSLogix\ODFC` |

> No **Group Policy Management**, após importar os ADMX: *Computer Configuration → Policies → Administrative Templates → **FSLogix → Profile Containers***. Todos os valores abaixo têm o equivalente em política.

---

## 1) Não permitir perfil local / temporário (consistência do perfil)

O maior problema de suporte em AVD é o usuário **trabalhar num perfil local ou temporário** quando o FSLogix falha — ele acha que salvou algo e no próximo logon **perde tudo**, porque o container não foi montado. Estas três chaves fecham essa porta:

| Configuração (`...\FSLogix\Profiles`) | Valor | O que faz | Impacto |
|---|---|---|---|
| **DeleteLocalProfileWhenVHDShouldApply** | `1` | Quando o usuário deve ter container FSLogix **e existe um perfil local**, o FSLogix **apaga o perfil local**. | Evita o conflito "perfil local antigo x container". ⚠️ **Apaga permanentemente** o perfil local existente — em host **persistente**, garanta que não há dados só-locais importantes. |
| **PreventLoginWithTempProfile** | `1` | Se o Windows criar um **perfil temporário** (sinal de que o container não montou), o FSLogix **bloqueia a sessão** (tela do FRXShell → só "Sair"). | O usuário **não trabalha** num perfil que não salva. Troca "perder dados silenciosamente" por "não entrar e abrir chamado" — o comportamento correto em produção. |
| **PreventLoginWithFailure** | `1` | Se houver **falha ao anexar/usar** o container VHD(x), bloqueia a sessão com o mesmo aviso de suporte. | Igual acima, para falhas de anexação (storage indisponível, ACL errada). Evita sessão "meia-boca". |

**Complementos por GPO (Computer Configuration → Policies → Administrative Templates):**
- *System → User Profiles →* **"Delete cached copies of roaming profiles" = Enabled** — não deixa cópias locais de perfis se acumularem nos hosts pooled.
- *System → User Profiles →* **"Do not forcefully unload the users registry at user logoff" = Enabled** — evita corrupção/lentidão ao descarregar a hive.
- **Não configure Perfis Móveis (Roaming Profiles) do Windows** — quem roda o perfil é o FSLogix; misturar os dois quebra tudo.

---

## 2) Limitar o tamanho do perfil

| Configuração (`...\FSLogix\Profiles`) | Valor | O que faz | Impacto |
|---|---|---|---|
| **SizeInMBs** | `30000` (≈30 GB) *(padrão)* | Tamanho **máximo** do container VHD(x). Você pode **aumentar** depois, mas **não diminuir**. | Segura o crescimento do perfil no storage. ⚠️ Se o perfil **estourar** o limite, o usuário recebe **erros** (não salva mais). Dimensione conforme Outlook cache/OneDrive: 30 GB é um bom começo; ambientes com OST grande podem precisar de mais. |
| **IsDynamic** | `1` *(padrão)* | VHD(x) **dinâmico**: ocupa só o espaço usado, cresce até `SizeInMBs`. | Economiza espaço em disco no storage. Não deixa o container passar de `SizeInMBs`. |
| **VolumeType** | `vhdx` | Cria containers no formato **VHDX** (recomendado — mais robusto e maior que VHD). | Padrão de fábrica é `vhd`; **defina `vhdx`** em ambientes novos. |
| **VHDCompactDisk** | `1` *(padrão · `...\FSLogix\Apps`)* | Tenta **compactar** o VHD(x) no logoff, reduzindo o "Size On Disk". | Menos consumo de storage ao longo do tempo, sem intervenção manual. |

> 💡 **Custo:** o tamanho do container impacta diretamente o **custo do storage** (Azure Files/ANF). O Autoscale economiza **compute**, não storage — controlar `SizeInMBs` é o que segura o custo de armazenamento.

---

## 3) Tempo e robustez para montar o perfil

Quando o container está **bloqueado** (aberto por outra sessão) ou o storage demora a responder, estes valores controlam **quanto o FSLogix espera/insiste** antes de falhar. Mexer neles ajuda em storage lento (SMB/ANF) ou logoff sujo.

| Configuração (`...\FSLogix\Profiles`, salvo indicado) | Padrão | O que faz | Impacto |
|---|---|---|---|
| **VolumeWaitTimeMS** | `20000` (20 s) | Tempo que o sistema espera o **volume aparecer** depois de anexar o VHD(x). | Muito baixo → falha de montagem em storage lento. Muito alto → logon demora mais quando há problema real. |
| **LockedRetryCount** | `12` | Nº de tentativas quando o VHD(x) está **bloqueado** (aberto por outra máquina/sessão). | Sobe se você vê "profile locked" (logoff sujo). Combinado com o intervalo abaixo define a janela total de espera. |
| **LockedRetryInterval** | `5` (s) | Segundos entre as tentativas de `LockedRetryCount`. | `12 × 5s = 60s` de espera por bloqueio, por padrão. |
| **ReAttachRetryCount** | `60` | Tentativas de **reanexar** o container se ele cair no meio da sessão. | Mantém a sessão viva em quedas rápidas de rede/storage. |
| **ReAttachIntervalSeconds** | `10` (s) | Segundos entre as tentativas de reanexação. | — |
| **CleanupInvalidSessions** (`...\FSLogix\Apps`) | `0` → use **`1`** | Limpa sessões que **terminaram abruptamente** e deixaram o VHD(x) preso, permitindo o próximo logon. | **Recomendado ligar (`1`)** — reduz drasticamente o clássico "não consigo logar porque o perfil ficou preso". |

---

## 4) Exclusões de antivírus (Microsoft Defender) — **crítico**

Antivírus varrendo o VHD(x) em **tempo real** é uma das **maiores causas de corrupção de container** e de **logon lento** — o AV tenta escanear todo o disco de perfil a cada anexação. **Exclua** os arquivos, pastas e processos do FSLogix em **todas as camadas** de segurança (AV do host, scanner de rede, DLP).

**Arquivos e pastas:**
```
%ProgramFiles%\FSLogix\Apps\frxdrv.sys
%ProgramFiles%\FSLogix\Apps\frxdrvvt.sys
%ProgramFiles%\FSLogix\Apps\frxccd.sys
%TEMP%\*.VHD
%TEMP%\*.VHDX
%Windir%\TEMP\*.VHD
%Windir%\TEMP\*.VHDX
\\<storage>.file.core.windows.net\<share>\*\*.VHD      ← raiz dos perfis (ajuste ao seu share)
\\<storage>.file.core.windows.net\<share>\*\*.VHDX
```

**Processos:**
```
%ProgramFiles%\FSLogix\Apps\frxccd.exe
%ProgramFiles%\FSLogix\Apps\frxccds.exe
%ProgramFiles%\FSLogix\Apps\frxsvc.exe
```

**Cloud Cache (só se usar Cloud Cache):**
```
%ProgramData%\FSLogix\Cache\*.VHD
%ProgramData%\FSLogix\Cache\*.VHDX
%ProgramData%\FSLogix\Proxy\*.VHD
%ProgramData%\FSLogix\Proxy\*.VHDX
```

**Como aplicar por GPO (Defender):** *Computer Configuration → Policies → Administrative Templates → Windows Components → Microsoft Defender Antivirus → Exclusions* → **Path Exclusions** (arquivos/pastas) e **Process Exclusions** (processos). Ou por **Intune** (Antivirus policy).

> **Impacto de NÃO fazer:** logons de 30–60 s+ (o AV escaneia o container inteiro na montagem) e, no pior caso, **corrupção do perfil** exigindo recriar o container. Impacto de fazer: exclui **apenas** os arquivos do FSLogix — o resto do sistema continua protegido.

---

## 5) Reduzir I/O e "peso" do container

| Configuração | Valor | O que faz | Impacto |
|---|---|---|---|
| **SetTempToLocalPath** (`Profiles`) | `3` *(padrão)* | Redireciona **TEMP, TMP e INetCache** para o **disco local** do host (fora do container). | Menos escrita no VHD(x) → logon/logoff mais rápidos e menos corrupção. Como é disco local efêmero, o lixo some no reprovisionamento. |
| **OutlookCachedMode** (`Profiles`) | `1` *(padrão)* | Ativa o cache do Outlook **só enquanto o container está anexado**. | Evita baixar OST gigante se o FSLogix estiver desabilitado. Use com o **Office Container (ODFC)** para o OST viver junto do perfil. |
| **RedirXMLSourceFolder** (`Profiles`) | caminho do `redirections.xml` | Aponta um **redirections.xml** que exclui do container caches descartáveis (ex.: **Teams**, cache do Chrome/Edge). | Container menor e mais rápido — não carrega gigabytes de cache que se recriam sozinhos. |
| **RoamSearch** (`Profiles`) | `0` (deixe) | Roaming do índice de Busca do Windows. | **Não é mais necessário** no Windows 10/11 multi-session e Server 2019+. Deixe **desligado** (evita índice inchado no perfil). |

---

## 6) Otimização da imagem (golden image)

Aplique na **imagem** (Lab 06), antes do Sysprep:

- **VDOT — Virtual Desktop Optimization Tool** ([github.com/The-Virtual-Desktop-Team/Virtual-Desktop-Optimization-Tool](https://github.com/The-Virtual-Desktop-Team/Virtual-Desktop-Optimization-Tool)): desliga serviços, tarefas agendadas, telemetria e animações desnecessárias em VDI. **Impacto:** logon mais rápido, menos CPU/RAM por sessão → mais usuários por host.
- **Desativar Storage Sense** — senão ele pode **apagar cache do FSLogix**/arquivos temporários que o perfil espera encontrar.
- **Fuso horário** — via GPO (`Set-TimeZone` por startup script) ou manual em lab (ver **Lab 06 Parte E**). VMs do Azure nascem em **UTC**.
- **Não deixar o host dormir/hibernar** — session hosts sempre em **alto desempenho**; o desligamento é responsabilidade do **Scaling Plan** (Lab 08), não do power management do Windows.
- **Remover apps per-user/MSIX desnecessários** antes do Sysprep (ver Lab 06 C.1) — evita o erro `0x80073cf2`.

---

## 7) Limites de sessão (ajuda o Autoscale a desalocar)

Sessões **desconectadas** que ficam presas impedem o host de esvaziar — e, sem esvaziar, o **Autoscale não desaloca** (Lab 08). Configure por GPO: *Computer Configuration → Administrative Templates → Windows Components → Remote Desktop Services → Remote Desktop Session Host → **Session Time Limits***:

| Política | Sugestão | Por quê |
|---|---|---|
| **Set time limit for disconnected sessions** | 1–2 h | Encerra sessões desconectadas → host esvazia → Autoscale desaloca. |
| **Set time limit for active but idle sessions** | 2–3 h | Libera sessões ociosas. |
| **End session when time limits are reached** | Enabled | Garante o logoff (não só disconnect). |

> Combina com o **Force logoff** do ramp-down do Scaling Plan (Lab 08): os dois juntos garantem que os hosts realmente esvaziam para desligar.

---

## 8) Nomenclatura e localização das pastas de perfil

| Configuração (`Profiles`) | Valor | O que faz | Impacto |
|---|---|---|---|
| **FlipFlopProfileDirectoryName** | `1` | Cria a pasta como **`%username%_%sid%`** (em vez de `%sid%_%username%`). | Pastas ficam **ordenadas por nome de usuário** no share — muito mais fácil achar/gerenciar o perfil de alguém. |
| **VHDLocations** | caminho SMB | **Obrigatório** — onde ficam os VHD(x). Ex.: `\\<storage>.file.core.windows.net\<share>`. | É a base de tudo (ver Lab 05). |
| **Enabled** | `1` | **Obrigatório** — liga o Profile Container. | Sem isso o FSLogix não faz nada. |

---

## 9) Rede, storage e Kerberos (atenção 2026)

- **SMB/Kerberos AES:** a partir do **update de abril/2026** do Windows Server, o tipo de criptografia Kerberos padrão muda de **RC4 para AES-SHA1**. Shares que hospedam containers FSLogix e **não** estiverem em AES-SHA1 podem ter **problema de acesso** após o update. **Ação:** migrar para **AES-SHA1 antes** de aplicar o update. *(Ref.: blog FSLogix.)*
- **Permissões do share** (Lab 05): NTFS/ACL corretas para os usuários criarem seu próprio VHD(x); RBAC do storage (`Storage File Data SMB Share Contributor`).
- **Latência** do storage impacta o **tempo de logon** — prefira storage na **mesma região** dos hosts; para alta performance, **Azure NetApp Files**.

---

## Checklist rápido de tuning (produção)

- [ ] `Enabled=1`, `VHDLocations`, `VolumeType=vhdx`, `SizeInMBs` dimensionado, `IsDynamic=1`.
- [ ] `DeleteLocalProfileWhenVHDShouldApply=1`, `PreventLoginWithTempProfile=1`, `PreventLoginWithFailure=1`.
- [ ] `CleanupInvalidSessions=1`, `FlipFlopProfileDirectoryName=1`.
- [ ] **Exclusões do Defender** (arquivos + processos + share) aplicadas por GPO/Intune.
- [ ] `SetTempToLocalPath=3`, `redirections.xml` para caches (Teams/navegadores), `RoamSearch=0`.
- [ ] Imagem otimizada com **VDOT**, Storage Sense **off**, fuso correto.
- [ ] **Session Time Limits** (disconnect/idle) por GPO + **Force logoff** no ramp-down (Lab 08).
- [ ] Storage na **mesma região**, permissões OK, **Kerberos AES-SHA1** (antes de abr/2026).

---

## Referências (Microsoft Learn)
- [FSLogix — Configuration settings reference (todas as chaves)](https://learn.microsoft.com/en-us/fslogix/reference-configuration-settings)
- [FSLogix — Prerequisites (exclusões de antivírus)](https://learn.microsoft.com/en-us/fslogix/overview-prerequisites)
- [Defender — Exclusions on Windows Server](https://learn.microsoft.com/en-us/defender-endpoint/configure-server-exclusions-microsoft-defender-antivirus)
- [FSLogix — Configuration examples](https://learn.microsoft.com/en-us/fslogix/concepts-configuration-examples)
- [Virtual Desktop Optimization Tool (VDOT)](https://github.com/The-Virtual-Desktop-Team/Virtual-Desktop-Optimization-Tool)
- [Blog FSLogix — Kerberos hardening (RC4→AES) e SMB](https://techcommunity.microsoft.com/blog/fslogix-blog/action-required-windows-kerberos-hardening-rc4-may-affect-fslogix-profiles-on-sm/4506378)
