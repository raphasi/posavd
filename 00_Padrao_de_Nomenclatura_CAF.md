# Padrão de Nomenclatura — Disciplina AVD (modelo CAF)

> **Disciplina:** Azure Virtual Desktop — Pós-Graduação em Arquitetura Avançada em Azure
> **Objetivo:** padronizar os nomes de todos os recursos dos laboratórios seguindo o **Cloud Adoption Framework (CAF)** da Microsoft, para que o aluno aprenda a nomear recursos como em ambiente corporativo real.

---

## O modelo

```
<tipo>-<workload>-<ambiente>-<região>-<instância>
```

Exemplo: **`rg-avd-prd-cin-001`**

| Segmento | Valor neste projeto | Significado |
|----------|--------------------|-------------|
| **tipo** | `rg`, `vnet`, `vdpool`… | Abreviação do tipo de recurso (tabela abaixo) |
| **workload** | `avd` | A aplicação/carga de trabalho — aqui, Azure Virtual Desktop |
| **ambiente** | `prd` | Ambiente — usamos **produção** (`prd`) para simular cenário real |
| **região** | `cin` | Região do Azure — **Central India** |
| **instância** | `001`, `002`… | Número sequencial (diferencia recursos do mesmo tipo) |

### Códigos de ambiente (referência)
| Código | Ambiente |
|--------|----------|
| `prd` | Produção (**usado neste projeto**) |
| `dev` | Desenvolvimento |
| `tst` | Teste / QA |
| `lab` | Laboratório / sandbox |

### Códigos de região (referência)
| Código | Região Azure |
|--------|--------------|
| `cin` | **Central India** (usado neste projeto) |
| `ins` | South India |
| `brs` | Brazil South |
| `eus2` | East US 2 |
| `weu` | West Europe |

> A região de implantação de **todos os recursos** deste projeto é **Central India**. O *locale* do sistema operacional (idioma pt-BR, teclado ABNT2, fuso de Brasília) é configurado independentemente da região do datacenter, no Lab 04 — ou seja, os hosts ficam em Central India mas entregam experiência em português para o usuário brasileiro.

---

## Tabela de abreviações de tipo (CAF)

| Recurso | Abreviação | Exemplo neste projeto |
|---------|-----------|-----------------------|
| Resource Group | `rg` | `rg-avd-prd-cin-001` |
| Virtual Network | `vnet` | `vnet-avd-prd-cin-001` |
| Subnet | `snet` | `snet-hosts-prd-cin-001` |
| Network Security Group | `nsg` | `nsg-hosts-prd-cin-001` |
| Public IP | `pip` | `pip-ng-avd-prd-cin-001` |
| NAT Gateway | `ng` | `ng-avd-prd-cin-001` |
| AVD Host Pool | `vdpool` | `vdpool-avd-prd-cin-001` |
| AVD Workspace | `vdws` | `vdws-avd-prd-cin-001` |
| AVD Application Group | `vdag` | `vdag-avd-prd-cin-001` |
| AVD Scaling Plan | `vdscaling` | `vdscaling-avd-prd-cin-001` |
| Storage Account | `st` | `stavdfsxentracin001` (ver regra abaixo) |
| Azure Compute Gallery | `gal` | `galavdprdcin001` (ver regra abaixo) |
| Private Endpoint | `pep` | `pep-st-adds-prd-cin-001` |
| Log Analytics Workspace | `log` | `log-avd-prd-cin-001` |
| Virtual Machine | `vm` | `vm-adds-prd-cin` (ver regra abaixo) |

---

## Regras especiais (limites do Azure que quebram o padrão puro)

Alguns recursos **não aceitam** o formato com hífens ou têm limite de caracteres. Estes são os desvios oficiais:

**1. Storage Account** — só letras minúsculas e números, **sem hífen**, **3–24 caracteres**, único globalmente. Por isso comprimimos: `st` + `avd` + função + `cin` + instância → `stavdfsxentracin001`, `stavdfsxaddscin001`. (Ajuste se o nome já estiver em uso.)

**2. Azure Compute Gallery** — **não aceita hífen** (só letras, números, `.` e `_`). Por isso: `galavdprdcin001`.

**3. Virtual Machine (nome de computador)** — o nome de host Windows tem **limite de 15 caracteres**. O formato CAF completo (`vm-avd-prd-cin-001` = 18) estoura. Por isso, **só para VMs**, usamos uma forma compacta que cabe em 15 e codifica função + região + instância:

| VM | Nome | Caracteres |
|----|------|-----------|
| Controlador de domínio | `vm-adds-prd-cin` | 15 |
| VM de build da imagem | `vmbld-cin-01` | 12 |
| Session hosts — cenário Entra ID (prefixo) | `vmavde-cin` → gera `vmavde-cin-0`, `vmavde-cin-1` | 12 |
| Session hosts — cenário AD DS (prefixo) | `vmavda-cin` → gera `vmavda-cin-0`, `vmavda-cin-1` | 12 |

> Nas VMs omitimos o segmento de ambiente (`prd`) para caber no limite de 15 — é o desvio aceito e documentado pelo próprio CAF para nomes de host Windows.

---

## Mapa de-para — nomes antigos → novos

Use esta tabela como referência rápida ao revisar os labs.

| Recurso | Nome antigo | Nome novo |
|---------|-------------|-----------|
| Resource Group | `rg-avd-lab` | `rg-avd-prd-cin-001` |
| Virtual Network | `vnet-avd-lab` | `vnet-avd-prd-cin-001` |
| Subnet hosts | `snet-hosts` | `snet-hosts-prd-cin-001` |
| Subnet FSLogix | `snet-fslogix` | `snet-fslogix-prd-cin-001` |
| Subnet AD DS | `snet-adds` | `snet-adds-prd-cin-001` |
| NAT Gateway | `nat-avd-lab` | `ng-avd-prd-cin-001` |
| Public IP (NAT) | `pip-nat-avd` | `pip-ng-avd-prd-cin-001` |
| Log Analytics | `law-avd-lab` | `log-avd-prd-cin-001` |
| Host Pool (Entra) | `hp-avd-entra-01` | `vdpool-avd-prd-cin-001` |
| Host Pool (AD DS) | `hp-avd-adds-01` | `vdpool-avd-prd-cin-002` |
| Workspace | `ws-avd-lab` | `vdws-avd-prd-cin-001` |
| App Group (Entra) | `dag-avd-entra-01` | `vdag-avd-prd-cin-001` |
| App Group (AD DS) | `dag-avd-adds-01` | `vdag-avd-prd-cin-002` |
| Session hosts (Entra) | `avdh-entra-0x` | `vmavde-cin-0x` |
| Session hosts (AD DS) | `avdh-adds-0x` | `vmavda-cin-0x` |
| VM de build | `avdh-build-01` | `vmbld-cin-01` |
| Controlador de domínio | `vm-dc-01` | `vm-adds-prd-cin` |
| Storage FSLogix (Entra) | `stavdfslogixentra` | `stavdfsxentracin001` |
| Storage FSLogix (AD DS) | `stavdfslogixadds` | `stavdfsxaddscin001` |
| Private Endpoint | `pe-stavdfslogixadds` | `pep-st-adds-prd-cin-001` |
| Compute Gallery | `acgavdlab` | `galavdprdcin001` |
| Image Definition | `win11-avd-ptbr` | `win11-avd-prd-cin` |
| Scaling Plan | `sp-avd-pooled-01` | `vdscaling-avd-prd-cin-001` |
| Região (todos) | Brazil South | **Central India** |

> **Instância 001 vs 002:** o `001` identifica o cenário **Entra ID** (Labs 01–02) e o `002` o cenário **AD DS** (Labs 03–06) para os recursos que existem nos dois (host pool e app group). RG, VNet, subnets e workspace são **compartilhados** e ficam todos em `001`.

> O domínio AD DS (`avdlab.local`) e o fuso/locale (pt-BR, Brasília) seguem suas próprias convenções e **não** entram no padrão de recursos Azure.
