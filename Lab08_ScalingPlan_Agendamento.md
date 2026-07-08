# Lab 08 — Agendamento de startup/shutdown com Scaling Plan nativo do AVD

> **Disciplina:** Azure Virtual Desktop — Pós-Graduação em Arquitetura Avançada em Azure
> **Modalidade:** Passo a passo via Portal do Azure (portal-first).
> **Dependência:** Um host pool existente — recomenda-se o **`vdpool-avd-prd-cin-002`** (Lab 03), `vdpool-avd-prd-cin-003` (Lab 07 · RemoteApp) e/ou `vdpool-avd-prd-cin-001` (Lab 01).

---

<p align="center">
  <img src="https://img.shields.io/badge/Dificuldade-Intermedi%C3%A1rio-orange?style=for-the-badge" alt="Dificuldade">
  <img src="https://img.shields.io/badge/Tempo-45--60_min-blue?style=for-the-badge" alt="Tempo">
  <img src="https://img.shields.io/badge/Portal--first-Azure-0078D4?style=for-the-badge&logo=microsoftazure&logoColor=white" alt="Portal-first">
  <img src="https://img.shields.io/badge/Foco-Autoscale_%2F_FinOps-2F2F8F?style=for-the-badge" alt="Autoscale">
</p>

## 🗺️ Ciclo diário do Autoscale

```mermaid
flowchart LR
    A["🌅 07:00 · Ramp-up<br/>liga hosts (min 20%)"] --> B["☀️ 09:00 · Peak<br/>capacidade máxima"]
    B --> C["🌇 18:00 · Ramp-down<br/>logoff + deallocate"]
    C --> D["🌙 20:00 · Off-peak<br/>hosts desligados (min 0%)"]
    D -.->|"próximo dia útil"| A
    SP["⚙️ vdscaling-avd-prd-cin-001"] -.->|"controla as fases"| A
```

> **Leitura:** o scaling plan liga os hosts no início do expediente e os **desaloca** (sem custo de compute) à noite. Com expediente 07:00–18:00 em dias úteis: ~55 h/semana ligado vs 168 h se ficasse sempre on → **~67% de economia** no compute.

---

## 🧭 Ficha do laboratório

| Item | Detalhe |
|------|---------|
| **Dificuldade** | ★★ Intermediário |
| **Tempo estimado** | 45–60 min |
| **Objetivo** | Usar o **Autoscale / Scaling Plan** nativo do AVD para **ligar (start) e desligar/desalocar (deallocate)** os session hosts conforme horário, reduzindo custo de compute fora do expediente. |
| **Pré-requisitos** | Host pool com session hosts; papel **Owner** (para criar a custom role/atribuição que o Autoscale exige) ou Contributor + User Access Administrator. |
| **Recursos consumidos** | 1× Scaling Plan (sem custo direto), atribuição de função ao service principal do AVD. |
| **Entrega** | Scaling plan ativo com schedules de semana, ligando hosts no ramp-up e desalocando no off-peak. |

### Conceito — o que o Autoscale faz
O Autoscale liga e **desaloca** (stop + deallocate, sem cobrança de compute) os session hosts conforme **fases do dia**:
- **Ramp-up** — início do expediente, usuários começam a entrar.
- **Peak** — horário de pico.
- **Ramp-down** — fim do expediente, usuários saem.
- **Off-peak** — noite/madrugada, capacidade mínima.

Há **dois tipos** de scaling plan, e o comportamento difere:
| Tipo | Como decide ligar/desligar |
|------|----------------------------|
| **Pooled** | Por **capacidade/threshold** e % mínimo de hosts ligados por fase (balanceia carga entre hosts). |
| **Personal** | Por **estado da sessão** do usuário (signed-out / disconnected) — define a ação (deallocate/hibernate) por fase. |

> Como os labs usam host pools **Pooled** (`vdpool-avd-prd-cin-002`, `vdpool-avd-prd-cin-001`), este lab foca no **Scaling Plan Pooled**. A Parte F traz as diferenças para Personal.

### 🎓 Entendendo na prática (material de aula)

**As 4 fases do Autoscale** — o que acontece com hosts, usuários e custo em cada etapa (simulação: 3 hosts, até 100 usuários) e o **algoritmo de balanceamento por fase** (Breadth-first no pico, Depth-first nas demais):

![Scaling Plan — as 4 fases do Autoscale](Scaling_Plan_1_Fases_Autoscale.svg)

**Quando cada host liga/desliga por nº de usuários** — os pontos de virada do *scale-out* / *scale-in* (com 40 sessões/host e Capacity threshold 75%):

![Autoscale — gatilhos por número de usuários](Scaling_Plan_2_Gatilhos_Autoscale.svg)

> 💡 **Fórmula do gatilho:** `(hosts ligados) × (sessões por host) × (Capacity threshold)`. Com 40 sessões/host e 75%: a **2ª** máquina liga no **31.º** usuário e a **3ª** no **61.º**; descendo, desliga quando as sessões cabem nos hosts restantes (**≤60 → 2 hosts, ≤30 → 1 host**), e a última é desalocada no **off-peak** quando esvazia. Mude o **Max session limit** ou o **threshold** e os números mudam.


---

## Parte A — Conceder a permissão de Autoscale ao service principal do AVD (obrigatório)

O serviço AVD precisa de permissão para ligar/desligar e gerenciar a energia das VMs. Cria-se (uma vez) uma **custom role** e atribui-se ao service principal **Azure Virtual Desktop**.

> Em muitos tenants já existe a função **"Desktop Virtualization Power On Off Contributor"** pronta — use-a diretamente e pule a criação da custom role.

1. Barra de busca → **Subscriptions → sua assinatura → Access control (IAM) → + Add → Add role assignment.**
2. **Role:** selecione **Desktop Virtualization Power On Off Contributor**.
   > Se não existir, crie uma custom role com as ações: `Microsoft.Compute/virtualMachines/start/action`, `.../deallocate/action`, `.../restart/action`, `.../read`, e ações de leitura/escrita em `Microsoft.DesktopVirtualization/hostpools` e `.../sessionhosts`. Atribua no escopo da assinatura.
3. **Members:** **Assign access to → User, group, or service principal** → **+ Select members** → busque **Azure Virtual Desktop** (o service principal de 1st party; App ID `9cdead84-a844-4324-93f2-b2e6bb768d07`) → selecione.
4. **Review + assign.**

> ⚠️ Sem esta atribuição, o scaling plan provisiona mas **não consegue** ligar/desligar as VMs.

---

## Parte B — Criar o Scaling Plan (Pooled)

1. **Azure Virtual Desktop → Scaling plans → + Create.**
2. **Basics:**
   - **Resource group:** `rg-avd-prd-cin-001`.
   - **Name:** `vdscaling-avd-prd-cin-001`.
   - **Location:** Central India (metadados).
   - **Time zone:** **(UTC-03:00) Brasilia** — define o relógio dos schedules.
   - **Host pool type:** **Pooled**.
   - **Exclusion tags** (opcional): tag para excluir hosts específicos do scaling (ex.: `noScale`).
3. Avance para **Schedules**.

---

## Parte C — Definir o Schedule de dias úteis

1. Na aba **Schedules → + Add schedule.**
2. **General:**
   - **Schedule name:** `weekdays`.
   - **Repeat on:** Monday a Friday.

### C.1 — Ramp-up (início do expediente)
- **Start time:** `07:00`.
- **Load balancing:** **Breadth-first**.
- **Minimum percentage of hosts (%):** `20` (garante pelo menos ~1 host ligado p/ os primeiros usuários).
- **Capacity threshold (%):** `60` (acima disso, liga mais hosts).

### C.2 — Peak (pico)
- **Start time:** `09:00`.
- **Load balancing:** **Depth-first** (concentra usuários em menos hosts; mais econômico) ou Breadth-first (mais performance).
- **Capacity threshold (%):** `80`.

### C.3 — Ramp-down (fim do expediente) — onde o **desligamento** começa
- **Start time:** `18:00`.
- **Load balancing:** **Depth-first**.
- **Minimum percentage of hosts (%):** `10`.
- **Capacity threshold (%):** `90`.
- **Force logoff users:** defina a política — recomendado **Yes**, com:
  - **Wait time (min):** `30` (avisa o usuário antes de deslogar).
  - **Notification message:** ex. "Sua sessão será encerrada em 30 minutos. Salve seu trabalho."
- **Stop hosts when:** **deallocate** (desaloca a VM → para de cobrar compute) — esta é a ação que **desliga** as máquinas.

### C.4 — Off-peak (madrugada) — máquinas desligadas
- **Start time:** `20:00`.
- **Load balancing:** **Depth-first**.
- **Capacity threshold (%):** `90`.
- **Minimum percentage of hosts (%):** `0` (permite desligar **todos** os hosts se não houver sessão).

3. **Add** para salvar o schedule.

> 💡 **Como o start/stop acontece:** no **ramp-up (07:00)** o Autoscale **liga** os hosts necessários. No **ramp-down (18:00)** ele desloga usuários (respeitando o wait time) e **desaloca** hosts conforme a carga cai. No **off-peak**, com mínimo 0%, os hosts ficam **desligados/desalocados** durante a madrugada — sem custo de compute.

---

## Parte D — (Opcional) Schedule de fim de semana

1. **+ Add schedule** → **Name:** `weekend` → **Repeat on:** Saturday, Sunday.
2. Configure capacidade mínima baixa em todas as fases (ex.: Min 0%, threshold alto), mantendo hosts **desligados** salvo demanda. Ajuste conforme a necessidade da turma/empresa.

---

## Parte E — Associar o Scaling Plan ao(s) Host Pool(s) e habilitar

1. Ainda no assistente, aba **Host pool assignments** (ou após criar: **Scaling plans → `vdscaling-avd-prd-cin-001` → Host pool assignments**).
2. **+ Assign** → selecione **`vdpool-avd-prd-cin-002`** (e/ou outro host pool Pooled).
3. Marque **Enable autoscale** para o host pool.
4. **Review + create → Create** (ou **Save**).

> Um scaling plan só pode ser atribuído a host pools do **mesmo tipo** (Pooled com Pooled). Um host pool aceita **apenas um** scaling plan.

---

## Parte F — Caso Personal (se for agendar um host pool Personal)

Se você tiver um host pool **Personal** e quiser agendar desligamento por inatividade:
1. Crie outro scaling plan com **Host pool type: Personal**.
2. Nos schedules, em vez de threshold/capacidade, você define **ações por estado de sessão**:
   - **Ramp-up:** **Start VM on Connect** (liga sob demanda) + opção de ligar todos no início.
   - **Ramp-down/Off-peak:** ação quando o usuário **desconecta** ou **faz logoff** → **Deallocate** (desliga) após X minutos, ou **Hibernate** (se habilitado).
3. Atribua ao host pool Personal e **Enable autoscale**.

> Para Personal, combine com **Start VM on Connect** no host pool (**Host pool → Properties → Start VM on connect = Yes**) para que a VM religue automaticamente quando o usuário tentar conectar.

---

## Parte G — Validar

1. **Scaling plans → `vdscaling-avd-prd-cin-001` → Host pool assignments:** confirme o host pool com **Autoscale = Enabled**.
2. Para testar **sem esperar o horário**, ajuste temporariamente o **Start time** de uma fase para alguns minutos à frente (ex.: off-peak começando "agora + 10min" com Min 0%) e observe.
3. Acompanhe a ação:
   - **Host pools → `vdpool-avd-prd-cin-002` → Session hosts:** veja os hosts mudarem para **Shutdown/Unavailable** (desalocados) no off-peak, e voltarem a **Available** no ramp-up.
   - **Virtual machines:** o estado das VMs `vmavda-cin-0x` deve ir para **Stopped (deallocated)** quando desligadas.
4. Para auditar decisões do Autoscale, habilite **Diagnostic settings** no scaling plan enviando para o **Log Analytics** (`log-avd-prd-cin-001`) e consulte a tabela de Autoscale.

### Critérios de sucesso
- [ ] Service principal **Azure Virtual Desktop** com a função Power On/Off atribuída.
- [ ] Scaling plan `vdscaling-avd-prd-cin-001` criado com fuso **Brasília** e 4 fases.
- [ ] Host pool com **Autoscale = Enabled**.
- [ ] No off-peak, as VMs vão para **Stopped (deallocated)** (param de cobrar compute).
- [ ] No ramp-up, ao menos o % mínimo de hosts volta a **Available** automaticamente.

---

## Erros comuns

| Sintoma | Causa | Correção |
|---------|-------|----------|
| Hosts não ligam/desligam no horário | Permissão Power On/Off ausente no service principal | Refaça a Parte A |
| VM "parada" mas ainda cobrando | Host foi **Stopped** e não **Deallocated** | Garanta a ação **deallocate** no ramp-down/off-peak |
| Usuários deslogados sem aviso | Force logoff sem wait time/mensagem | Ajuste **Wait time** e **Notification** no ramp-down |
| Não consigo atribuir o plano | Tipo do plano ≠ tipo do host pool | Crie plano do mesmo tipo (Pooled/Personal) |
| Schedule disparou no horário errado | Time zone do plano incorreto | Ajuste **Time zone** para (UTC-03:00) Brasília |

---

## Parte H — Monitoramento e diagnóstico do Autoscale

### H.1 — Acompanhar as decisões do Autoscale
1. **Scaling plans → `vdscaling-avd-prd-cin-001` → Diagnostic settings → + Add diagnostic setting** → envie os logs para o **Log Analytics** `log-avd-prd-cin-001`.
2. Em **Log Analytics → Logs**, consulte a tabela `WVDAutoscaleEvaluationPooled` para ver, por avaliação, quantos hosts foram ligados/desligados e **por quê**.
3. **Host pool → Insights** (workbook nativo do AVD) mostra sessões, hosts disponíveis e uso ao longo do dia.

### H.2 — Onde buscar logs quando "não liga/desliga"
| Sintoma | Onde olhar | O que procurar |
|---------|-----------|----------------|
| Hosts não ligam/desligam | **Subscription → IAM** | Função **Power On/Off Contributor** no SP **Azure Virtual Desktop** (Parte A) |
| Plano não age | **Scaling plan → Host pool assignments** | `Autoscale = Enabled` no host pool certo |
| Decisões inesperadas | **Log Analytics →** `WVDAutoscaleEvaluationPooled` | Thresholds / min % por fase e horário avaliado |
| Disparo em horário errado | **Scaling plan → Properties** | **Time zone** = (UTC-03:00) Brasília |
| Usuário deslogado sem aviso | Configuração do **Ramp-down** | Wait time + Notification message |

---

## Discussão para a aula — economia
Com expediente 07:00–18:00 em dias úteis, hosts ligados ~11h/dia × 5 dias ≈ **55h/semana** vs **168h** se ficassem sempre ligados — **~67% de redução** no custo de compute dos session hosts. Reforce: o **storage (FSLogix)** continua sendo cobrado; o Autoscale economiza **compute**, não armazenamento.

---

## 🧪 Anexo — Teste do Autoscale em laboratório SEM usuários (host pool com 3 hosts)

Objetivo: **ver os hosts ligando (start) e desalocando (deallocate)** dirigidos pelo **schedule** e pelo **% mínimo de hosts**, sem precisar de usuários reais conectados. Ótimo para validar/demonstrar em aula em ~20 min.

> **O que dá e o que não dá para testar sem usuários:**
> - ✅ **Ligar/desligar por horário + Min hosts %** (dirigido pelo schedule) — funciona **sem ninguém conectado**.
> - ⚠️ **Scale-out por carga** (2ª host no 31.º usuário, 3ª no 61.º — ver desenho *"gatilhos por usuários"*) — **exige sessões reais**; sem usuários não dispara. Veja o item **opcional** no fim para simular com 1–2 testers.

> 📌 **Terminologia:** aqui é **1 host pool com 3 hosts (máquinas)** — o Autoscale escala o **nº de hosts dentro de um pool**, não pools separados.

### Pré-requisitos do teste
- Um host pool **Pooled** com **3 session hosts** (ex.: adicione um 3º host ao `vdpool-avd-prd-cin-002`, **ou** crie um pool de teste `vdpool-test-cin-001` com 3 VMs pequenas `Standard_B2s`).
- **Role de Autoscale** atribuída ao SP *Azure Virtual Desktop* (**Parte A**) e um **Scaling Plan** criado (**Parte B**).
- **Time zone do Scaling Plan = (UTC-03:00) Brasília** (senão os horários não batem com o seu relógio).

### Passo 1 — Anote a hora e o estado inicial
- No pool → **Session hosts** e em **Virtual machines** veja o **power state** dos 3 hosts.
- Anote a **hora atual** (ex.: **14:00**) — vamos programar as fases minutos à frente.

### Passo 2 — Configure um schedule COMPRIMIDO (fases minutos à frente)
No Scaling Plan → **Schedules → + Add** (ou edite), para o **dia da semana atual**, com ~**5 min entre fases**:

| Fase | Início (ex.) | Config para o teste |
|------|--------------|---------------------|
| **Ramp-up** | 14:05 | **Min hosts = 100%** · Capacity threshold 75% · Load balancing *Depth-first* |
| **Peak** | 14:10 | Capacity threshold 75% · *Breadth-first* |
| **Ramp-down** | 14:15 | **Min hosts = 0%** · **Force logoff = Yes**, aguardar **1 min** · *Depth-first* |
| **Off-peak** | 14:20 | Capacity threshold 75% · *Depth-first* |

> **Por que `Min hosts = 100%` no ramp-up?** Sem usuários, o que **liga** os hosts é o **% mínimo**. Com 100% você vê os **3 hosts ligarem**. Quer ver ligar **parcial**? Use **34% ≈ 1 host** ou **67% ≈ 2 hosts**.

### Passo 3 — Associe o plano ao pool e habilite
- Scaling Plan → **Host pool assignments → + Assign** → escolha o pool de teste → **Enable autoscale = Yes**.

### Passo 4 — Assista ao vivo (sem usuários)
Deixe abertos **Virtual machines** (power state) e **Host pool → Session hosts**:
- **14:05 (ramp-up):** os **3 hosts ligam** (Running / Available) — puxados pelo Min 100%.
- **14:15 (ramp-down):** com Min 0% + force logoff, o Autoscale começa a **desalocar**.
- **14:20 (off-peak):** os **3 hosts ficam Stopped (deallocated)** → custo de compute zero.

> ⏱️ O Autoscale avalia em **ciclos de alguns minutos** — deixe ~5 min entre fases e espere alguns minutos após cada transição. Para **repetir**, empurre os horários do schedule à frente de novo.

### Passo 5 — Comprovar que foi o Autoscale
- **VM → Activity log** (ou **Host pool → Session hosts**): as operações **Start Virtual Machine** / **Deallocate Virtual Machine** aparecem **iniciadas pelo *Azure Virtual Desktop*** (service principal) — prova de que o schedule agiu.
- Se ativou o diagnóstico (**H.1**): `WVDAutoscaleEvaluationPooled` no Log Analytics mostra a decisão de cada avaliação.

### (Opcional) Testar o scale-out por CARGA com poucos testers
O scale-out por **nº de usuários** precisa de **sessões**. Para forçar com pouca gente:
- No host pool, baixe o **Max session limit para `1`** (cada usuário "enche" um host).
- Conecte **2 usuários de teste** → o 2º força o Autoscale a **ligar o 2º host**; um 3º força o 3º. É a régua do desenho *"gatilhos por usuários"*, só que com limite **1** em vez de 40.

### Limpeza do teste
- **Desabilite** o autoscale no pool (ou desassocie o plano) e **deallocate** os hosts de teste para não gerar custo.
- Se criou o pool de teste dedicado (`vdpool-test-cin-001`), **exclua-o** (pool + VMs + discos).

---

## 🧪 Anexo 2 — Demonstração guiada com 3 usuários (ligar → 2º host no login → pico → desligar)

Cenário exato: o pool **liga no horário** com 1 host; ao **logar o 2º usuário sobe o 2º host**; no **pico** (com o 3º usuário) sobem os **3**; no **horário de desligamento** as sessões são derrubadas e os hosts desalocam.

> 🔑 **O truque dos números:** com **Max session limit = 2** por host e **Capacity threshold = 60%**, o gatilho fica redondo para 3 usuários (fórmula: *sobe +1 host quando as sessões passam de `hosts × 2 × 60%`*):
> - **1 usuário** → 1 sessão em 1 host = 50% → **fica 1 host**.
> - **2 usuários** → 2 sessões / 1 host = 100% (> 60%) → **sobe o 2º host**.
> - **3 usuários** → 3 sessões / 2 hosts = 75% (> 60%) → **sobe o 3º host**.

### Pré-requisitos
- Host pool **Pooled** com **3 hosts** (todos **deallocated** no início) e **Max session limit = 2** (Host pool → Properties).
- **3 usuários de teste** atribuídos ao app group do pool (ex.: `user1/2/3` em `grp-avd-usuarios`), cada um com seu Windows App/credencial.
- **Role de Autoscale** no SP *Azure Virtual Desktop* (Parte A) + **Scaling Plan** (Parte B) + **Time zone = (UTC-03:00) Brasília**.

### Passo 1 — Schedule comprimido (ajuste os horários para minutos à frente do seu relógio)
No Scaling Plan → **Schedules → + Add** para o dia atual:

| Fase | Início (ex.) | Config |
|------|--------------|--------|
| **Ramp-up** | 14:05 | **Min hosts = 33%** (≈**1** de 3; se subir 2, use 20%) · **Capacity threshold = 60%** · Load balancing *Depth-first* |
| **Peak** | 14:12 | **Capacity threshold = 60%** · *Breadth-first* |
| **Ramp-down** | 14:28 | **Min hosts = 0%** · **Force logoff = Yes**, aguardar **1 min** · *Depth-first* |
| **Off-peak** | 14:33 | **Capacity threshold = 60%** · *Depth-first* |

### Passo 2 — Associar o plano e habilitar
- Scaling Plan → **Host pool assignments → + Assign** → o pool de teste → **Enable autoscale = Yes**.

### Passo 3 — Roteiro ao vivo (acompanhe em **Host pool → Session hosts** e **Virtual machines**)
1. **14:05 — Ramp-up:** o pool **liga 1 host** (Min 33%). Aguarde ficar **Available**.
2. **Conecte o usuário 1** (Windows App) → a sessão cai no **host 1** (1/2 = 50%) → **continua 1 host**.
3. **Conecte o usuário 2** → uso passa de **60%** → em 1 avaliação (~min) o Autoscale **liga o 2º host**.
4. **14:12 — Peak** · **conecte o usuário 3** → uso passa de 60% novamente → **liga o 3º host**. **3 hosts ligados** (no pico o Autoscale mantém — não desliga).
5. **14:28 — Ramp-down:** com **Force logoff**, as **sessões são derrubadas** (aviso + 1 min); com **Min 0%**, os hosts **desalocam** conforme esvaziam.
6. **14:33 — Off-peak:** os **3 hosts** ficam **Stopped (deallocated)** → custo de compute zero.

> ⚠️ **Sobre "o pico sobe o 3º host":** o Peak **não** tem "% mínimo" — quem sobe o 3º host é a **carga** (o 3º usuário). Por isso conecte o 3º usuário **no início do Peak**: fica visualmente "no pico subiu o 3º". Se ninguém logar, o Peak **não** força os 3.
>
> ⏱️ O Autoscale avalia em **ciclos de alguns minutos** — dê intervalo entre cada login e entre as fases; a subida/queda de host não é instantânea.

### Passo 4 — Comprovar
- **Host pool → Session hosts:** nº de hosts disponíveis subindo 1 → 2 → 3 e depois caindo.
- **VM → Activity log:** operações **Start / Deallocate** iniciadas por **Azure Virtual Desktop** (o SP), provando que foi o Autoscale.
- (se ativou **H.1**) `WVDAutoscaleEvaluationPooled` mostra, por avaliação, o motivo de cada decisão.

### Limpeza
- **Desabilite** o autoscale no pool (ou desassocie o plano), **deallocate** os hosts e, se foi pool dedicado de teste, **exclua-o**.

---

## Fim da trilha
Você construiu, via portal: host pools Entra ID e AD DS, FSLogix nos dois modelos de identidade, imagem customizada pt-BR e agendamento de energia. Ao encerrar, **desaloque ou exclua** os recursos para evitar cobrança (ver seção de limpeza em cada lab).
