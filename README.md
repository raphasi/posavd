# Laboratórios Azure Virtual Desktop (AVD)

Material prático da disciplina de **Azure Virtual Desktop** — Pós-Graduação em Arquitetura Avançada em Azure.
Laboratórios **portal-first** (passo a passo via Portal do Azure), com scripts apenas onde são obrigatórios.

> Nomenclatura no padrão **Cloud Adoption Framework (CAF)** — região **Central India (cin)**, ambiente **prd**. Ver [`00_Padrao_de_Nomenclatura_CAF.md`](00_Padrao_de_Nomenclatura_CAF.md).

## Trilha de laboratórios

| # | Lab | Foco | Depende de |
|---|-----|------|-----------|
| 01 | [Host Pool 2 VMs — Entra ID](Lab01_Hostpool_2VMs_EntraID.md) | Host pool cloud-native com ingresso no Microsoft Entra ID, SSO | — |
| 02 | [FSLogix — Entra ID](Lab02_FSLogix_EntraID.md) | Perfis FSLogix em Azure Files com Entra Kerberos | Lab 01 |
| 03 | [Host Pool 2 VMs — AD DS](Lab03_Hostpool_2VMs_ADDS.md) | Criação do AD DS + host pool domain-joined | — |
| 04 | [Imagem Windows 11 customizada](Lab04_Imagem_Windows11_Customizada.md) | Golden image pt-BR (idioma, teclado, fuso) + Compute Gallery + GPOs | Lab 03 |
| 05 | [FSLogix — AD DS + Private Endpoints](Lab05_FSLogix_ADDS_PrivateEndpoints.md) | Perfis FSLogix com AD DS e acesso privado | Lab 03 |
| 06 | [Scaling Plan — agendamento](Lab06_ScalingPlan_Agendamento.md) | Startup/shutdown automatico nativo do AVD | Host pool existente |

## Como usar

Cada lab traz uma ficha (dificuldade, tempo, pré-requisitos), o passo a passo no portal, critérios de sucesso e uma tabela de erros comuns. Os labs 01–02 formam o cenário **Entra ID**; os labs 03–06 formam o cenário **AD DS**.

## Ambiente base

- **Região:** Central India
- **Resource Group:** `rg-avd-prd-cin-001`
- **VNet:** `vnet-avd-prd-cin-001` (10.50.0.0/16) — sub-redes `snet-hosts-prd-cin-001`, `snet-fslogix-prd-cin-001`, `snet-adds-prd-cin-001`

---

*Professor: Raphael Andrade.*
