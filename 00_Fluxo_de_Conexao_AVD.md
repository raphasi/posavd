# Fluxo de uma conexão AVD — diagrama de sequência

> **Disciplina:** Azure Virtual Desktop — Pós-Graduação em Arquitetura Avançada em Azure
> **Objetivo:** mostrar, passo a passo, **o que acontece "por baixo do capô"** quando um usuário conecta — da autenticação ao carregamento do perfil. Use como mapa mental ao depurar qualquer problema de conexão.

<p align="center">
  <img src="https://img.shields.io/badge/Tipo-Diagrama_de_sequência-2F2F8F?style=for-the-badge" alt="Sequência">
  <img src="https://img.shields.io/badge/Plano_de_Controle-Microsoft-0078D4?style=for-the-badge" alt="Control Plane">
  <img src="https://img.shields.io/badge/Identidade-Entra_ID-success?style=for-the-badge" alt="Entra">
</p>

---

## 🎬 O filme da conexão

```mermaid
sequenceDiagram
    actor U as 👤 Usuário
    participant C as Cliente AVD
    participant E as 🔐 Microsoft Entra ID
    participant W as Web Access
    participant B as Broker
    participant G as Gateway
    participant H as 🖥️ Session Host
    participant F as 💾 Azure Files · FSLogix
    participant L as 📊 Log Analytics

    U->>C: Abre o Windows App / Web client
    C->>E: 1 · Autentica (usuário + senha)
    E->>E: 2 · Conditional Access / MFA / Security Defaults
    E-->>C: 3 · Token de acesso
    C->>W: 4 · Solicita os recursos do workspace
    W-->>C: 5 · Lista desktops e apps publicados
    U->>C: Clica no desktop
    C->>B: 6 · Pede a conexão
    B->>B: 7 · Escolhe o host (load balancing)
    B-->>C: 8 · Host alvo + dados de conexão
    C->>G: 9 · Abre túnel TLS reverso na porta 443
    G->>H: 10 · Encaminha a sessão até o host
    H->>E: 11 · SSO valida o token no logon
    E-->>H: 12 · Autoriza o login no host
    H->>F: 13 · FSLogix monta o perfil (VHDX)
    F-->>H: 14 · Perfil do usuário carregado
    H-->>U: 15 · Área de trabalho entregue
    H->>L: 16 · Envia diagnóstico e telemetria
```

---

## 📝 Passo a passo explicado

| Passo | O que acontece | Onde depurar se falhar |
|:-----:|----------------|------------------------|
| 1–3 | Cliente autentica no **Entra ID**; Conditional Access/MFA/Security Defaults são avaliados; token é emitido | **Entra ID → Sign-in logs** |
| 4–5 | **Web Access** devolve os recursos que o usuário tem direito (atribuição no **Application Group**) | Atribuição do App Group |
| 6–8 | **Broker** faz o *load balancing* e escolhe um session host disponível | **Host pool → Session hosts** (estado *Available*) |
| 9–10 | **Gateway** abre um túnel **TLS reverso (443)** — sem expor RDP público — e leva a sessão ao host | Saída de rede do host (NAT Gateway) |
| 11–12 | O **SSO** valida o token no logon do Windows e autoriza a entrada | RBAC *VM User Login* (Entra) / domínio (AD DS) + saída p/ `login.microsoftonline.com` |
| 13–14 | O **FSLogix** monta o perfil do usuário a partir do `.vhdx` no Azure Files | Logs do FSLogix · `klist` · Azure Files |
| 15 | A **área de trabalho** é entregue ao usuário | — |
| 16 | Logs e métricas fluem para o **Log Analytics** (AVD Insights) | Diagnostic settings |

---

## 🔀 O que muda entre Entra ID e AD DS

O fluxo é **o mesmo**; muda apenas *como* dois passos são resolvidos:

| Passo | 🔐 Entra ID (cloud-native) | 🗄️ AD DS (híbrido) |
|:-----:|----------------------------|--------------------|
| **11–12 · Autorização do logon** | RBAC `Virtual Machine User Login` + SSO Entra | Autorização pelo domínio `avdlab.local` + SSO Entra |
| **13 · Montagem do perfil** | Azure Files via **Entra Kerberos** | Azure Files via **AD DS** (AzFilesHybrid), geralmente por **Private Endpoint** |

> 💡 **Por que isso é importante para depurar:** quase todo erro de conexão cai em **um** desses passos. "Sign in Failed" → passos 11–12 (SSO/MFA/saída de rede). Perfil temporário → passos 13–14 (FSLogix/Kerberos). Recurso não aparece no cliente → passos 4–5 (atribuição no App Group). Saber em qual passo o filme parou direciona a correção.

---

## 🔗 Relacionado

- [Guia de decisão — Entra ID vs AD DS](00_Guia_Decisao_Identidade_Entra_vs_ADDS.md)
- [Lab 01 — Host Pool com Entra ID](Lab01_Hostpool_2VMs_EntraID.md) · [Lab 03 — Host Pool com AD DS](Lab03_Hostpool_2VMs_ADDS.md)
