<h1 align="center">🖥️ Laboratórios Azure Virtual Desktop (AVD)</h1>

<p align="center">
  <em>Material prático da disciplina de Azure Virtual Desktop — Pós-Graduação em Arquitetura Avançada em Azure</em>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Azure-Virtual_Desktop-0078D4?style=for-the-badge&logo=microsoftazure&logoColor=white" alt="AVD">
  <img src="https://img.shields.io/badge/Modalidade-Portal--first-2F2F8F?style=for-the-badge" alt="Portal-first">
  <img src="https://img.shields.io/badge/Laboratórios-6-success?style=for-the-badge" alt="6 labs">
  <img src="https://img.shields.io/badge/Nomenclatura-CAF-blue?style=for-the-badge" alt="CAF">
  <img src="https://img.shields.io/badge/Região-Central_India-orange?style=for-the-badge" alt="Central India">
</p>

---

## 📚 Sobre

Seis laboratórios **passo a passo via Portal do Azure** (portal-first), com scripts apenas onde são obrigatórios. Cada lab traz ficha resumida, diagrama de arquitetura, critérios de sucesso e uma tabela de erros comuns.

> 🏷️ Nomenclatura no padrão **Cloud Adoption Framework (CAF)** — ambiente **prd**, região **Central India (cin)**. Detalhes em [`00_Padrao_de_Nomenclatura_CAF.md`](00_Padrao_de_Nomenclatura_CAF.md).

> 🧭 **Não sabe qual trilha seguir?** Comece pelo [**Guia de decisão — Entra ID vs AD DS**](00_Guia_Decisao_Identidade_Entra_vs_ADDS.md).

## 🧭 Trilha de laboratórios

| # | Lab | Foco | Nível | Depende de |
|:-:|-----|------|:-----:|:----------:|
| 01 | [🔐 Host Pool 2 VMs — Entra ID](Lab01_Hostpool_2VMs_EntraID.md) | Host pool cloud-native + SSO | ★★ | — |
| 02 | [💾 FSLogix — Entra ID](Lab02_FSLogix_EntraID.md) | Perfis em Azure Files (Entra Kerberos) | ★★★ | Lab 01 |
| 03 | [🗄️ Host Pool 2 VMs — AD DS](Lab03_Hostpool_2VMs_ADDS.md) | Criação do AD DS + hosts domain-joined | ★★★ | — |
| 04 | [🖼️ Imagem Windows 11 customizada](Lab04_Imagem_Windows11_Customizada.md) | Golden image pt-BR + Gallery + GPOs | ★★★ | Lab 03 |
| 05 | [🔌 FSLogix — AD DS + Private Endpoints](Lab05_FSLogix_ADDS_PrivateEndpoints.md) | Perfis com AD DS e acesso privado | ★★★ | Lab 03 |
| 06 | [⚙️ Scaling Plan — agendamento](Lab06_ScalingPlan_Agendamento.md) | Startup/shutdown automático (FinOps) | ★★ | Host pool |

> 🟦 **Trilha Entra ID:** Labs 01 → 02  ·  🟨 **Trilha AD DS:** Labs 03 → 04 → 05 → 06

## 🗺️ Visão geral do ambiente

```mermaid
flowchart TB
    U["👤 Usuários"] --> E["🔐 Microsoft Entra ID<br/>identidade · SSO · MFA"]
    E --> CP["☁️ AVD Control Plane (Microsoft)<br/>Broker · Gateway · Web Access"]
    subgraph SUB["📦 rg-avd-prd-cin-001 · Central India"]
      direction TB
      subgraph VNET["🌐 vnet-avd-prd-cin-001 · 10.50.0.0/16"]
        subgraph SH["snet-hosts"]
          P1["🔐 vdpool-...-001<br/>hosts Entra-joined<br/>(Lab 01)"]
          P2["🗄️ vdpool-...-002<br/>hosts AD DS<br/>(Lab 03)"]
        end
        subgraph SF["snet-fslogix"]
          PE["🔌 Private Endpoint<br/>(Lab 05)"]
        end
        subgraph SA["snet-adds"]
          DC["🗄️ vmdc-cin-01<br/>AD DS (Lab 03)"]
        end
        NAT["🚪 NAT Gateway"]
      end
      ST["💾 Azure Files<br/>perfis FSLogix<br/>(Labs 02 · 05)"]
      GAL["🖼️ Compute Gallery<br/>imagem pt-BR (Lab 04)"]
      SCALE["⚙️ Scaling Plan<br/>(Lab 06)"]
    end
    CP --> P1
    CP --> P2
    P1 -.-> ST
    PE --> ST
    DC -.-> E
    GAL -.-> P2
    SCALE -.-> P1
    SCALE -.-> P2
```

## 🚀 Como usar

1. Comece pela trilha que interessa (**Entra ID** ou **AD DS**).
2. Siga os labs na ordem da coluna **Depende de**.
3. Reutilize o ambiente base (RG, VNet, sub-redes) entre os labs.

### 🧱 Ambiente base

| Recurso | Nome | Detalhe |
|---------|------|---------|
| 🌍 Região | Central India | `cin` |
| 📦 Resource Group | `rg-avd-prd-cin-001` | agrupa tudo |
| 🌐 VNet | `vnet-avd-prd-cin-001` | `10.50.0.0/16` |
| 🔹 Sub-redes | `snet-hosts` · `snet-fslogix` · `snet-adds` | `.1` · `.2` · `.3` /24 |

---

<p align="center">
  <sub>👨‍🏫 Professor: Raphael Andrade · Pós-Graduação em Arquitetura Avançada em Azure</sub>
</p>
