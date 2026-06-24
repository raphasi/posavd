# Guia de decisão — Microsoft Entra ID vs AD DS no AVD

> **Disciplina:** Azure Virtual Desktop — Pós-Graduação em Arquitetura Avançada em Azure
> **Objetivo:** ajudar o aluno a escolher **qual modelo de identidade** usar para os session hosts antes de iniciar os laboratórios — e, com isso, qual trilha seguir.

<p align="center">
  <img src="https://img.shields.io/badge/Tipo-Guia_de_decisão-2F2F8F?style=for-the-badge" alt="Guia">
  <img src="https://img.shields.io/badge/Trilha_Entra_ID-Labs_01_·_02-0078D4?style=for-the-badge" alt="Entra">
  <img src="https://img.shields.io/badge/Trilha_AD_DS-Labs_03_a_06-orange?style=for-the-badge" alt="AD DS">
</p>

---

## 🧭 Fluxograma de decisão

```mermaid
flowchart TD
    S(["🏁 Qual identidade usar nos session hosts?"]) --> Q1{"Já existe Active Directory<br/>on-prem (ou apps que exigem<br/>Kerberos/LDAP clássico:<br/>file server, ERP legado)?"}
    Q1 -->|"Sim"| ADDS
    Q1 -->|"Não"| Q2{"Precisa gerenciar os hosts<br/>com GPO tradicional<br/>(em vez de Intune)?"}
    Q2 -->|"Sim, GPO obrigatória"| ADDS
    Q2 -->|"Não, Intune resolve"| Q3{"Os perfis/arquivos precisam<br/>ficar em rede privada E<br/>autenticados pelo domínio<br/>(Private Endpoint + AD)?"}
    Q3 -->|"Sim"| ADDS
    Q3 -->|"Não"| Q4{"Quer o caminho mais simples,<br/>sem servidor para manter?"}
    Q4 -->|"Sim"| ENTRA
    Q4 -->|"Tanto faz"| ENTRA
    ADDS["🗄️ AD DS (híbrido)<br/>━━━━━━━━━━<br/>Trilha Labs 03 → 04 → 05 → 06<br/>DC + Entra Connect + GPO"]
    ENTRA["🔐 Microsoft Entra ID (cloud-native)<br/>━━━━━━━━━━<br/>Trilha Labs 01 → 02<br/>sem servidor · Intune · SSO"]
```

> **Regra de bolso:** comece sempre se perguntando *"existe alguma dependência que **obriga** o domínio clássico?"*. Se a resposta for **não** em todas as perguntas, **Entra ID** é o caminho mais simples, barato e moderno. O AD DS entra quando há uma **âncora no legado** (AD existente, GPO obrigatória, app que fala Kerberos clássico, ou storage que precisa de identidade de domínio).

---

## 📊 Comparativo lado a lado

| Dimensão | 🔐 Microsoft Entra ID (cloud-native) | 🗄️ AD DS (híbrido) |
|----------|--------------------------------------|--------------------|
| **Servidor a manter** | Nenhum (sem DC) | Controlador de domínio (VM) + patching/backup |
| **Identidade do usuário** | Só nuvem ou sincronizada | **Tem de ser híbrida** (criada no AD + Entra Connect) |
| **Ingresso dos hosts** | Microsoft Entra ID join | Domain join (`avdlab.local`) |
| **Gerência de políticas** | **Intune** (Settings Catalog) | **GPO** (Group Policy) |
| **Login no host** | RBAC `Virtual Machine User Login` + SSO | Autorização via AD + SSO Entra |
| **FSLogix / Azure Files** | **Entra Kerberos** (cloud) | **AD DS** (AzFilesHybrid) |
| **Rede privada p/ perfis** | Possível, menos comum | **Private Endpoint** (Lab 05) |
| **Apps Kerberos/LDAP clássicos** | ❌ Não atende | ✅ Atende |
| **Complexidade / custo** | Menor | Maior (DC, sync, GPO) |
| **Laboratórios** | **Labs 01 → 02** | **Labs 03 → 04 → 05 → 06** |

---

## 🎯 Recomendação por cenário

| Cenário | Escolha | Por quê |
|---------|---------|---------|
| Startup/empresa **100% nuvem**, sem AD | 🔐 **Entra ID** | Nada de legado; caminho mais simples e barato |
| Empresa com **AD on-prem** já em uso | 🗄️ **AD DS** | Reusa identidades, GPOs e apps de domínio existentes |
| Precisa de **GPO** detalhada nos hosts | 🗄️ **AD DS** | GPO clássica (ou avaliar Intune como equivalente) |
| App legado que fala **Kerberos/LDAP** | 🗄️ **AD DS** | Entra ID puro não entrega Kerberos clássico |
| Perfis FSLogix em **rede privada + domínio** | 🗄️ **AD DS** | Private Endpoint + Azure Files com AD (Lab 05) |
| POC rápida / aula introdutória | 🔐 **Entra ID** | Sobe em minutos, sem servidor para manter |

> 💡 **Importante:** em **ambos** os modelos a **autenticação do AVD passa sempre pelo Microsoft Entra ID** — a diferença está em como os *session hosts* e o *storage de perfis* são ingressados e gerenciados. Por isso o cenário AD DS exige **identidades híbridas** (sincronizadas para o Entra ID via Entra Connect).

---

## ➡️ Próximo passo

- Escolheu **Entra ID**? Comece pelo **[Lab 01 — Host Pool com Entra ID](Lab01_Hostpool_2VMs_EntraID.md)**.
- Escolheu **AD DS**? Comece pelo **[Lab 03 — Host Pool com AD DS](Lab03_Hostpool_2VMs_ADDS.md)**.
