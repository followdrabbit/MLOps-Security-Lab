# 🔁 Lab vs Prod — DatOps em AWS (Z0–Z3 + Z8/Z9)

## 1. Objetivo

Este documento compara, lado a lado, a arquitetura e os controles de segurança do:

- **Lab 01 — DatOps em AWS**  
  (`architecture-lab.md` + `SECURITY.md`)
- **Ambiente de Produção — DatOps em AWS**  
  (`architecture-prod.md` + `SECURITY-prod.md`)

Foco:

- Mostrar **o que existe de fato no lab**;
- Mostrar **o que muda em produção**;
- Sugerir um **caminho de evolução (roadmap de hardening)**.

---

## 2. Visão Resumida — Lab vs Prod

### 2.1. Tabela resumo por camada

| Camada                     | Lab (Baixo custo)                                             | Prod (Completa)                                                                 |
|---------------------------|--------------------------------------------------------------|---------------------------------------------------------------------------------|
| Z0 – Fontes               | Fontes simuladas + YAML + JSON Schema                        | Fontes reais + contratos formais + classificação + base legal                  |
| Z1 – Ingestion            | HTTP API + API Key + validação + Lambda                      | WAF + API Gateway (REST/HTTP) + Auth forte (API Key/JWT/mTLS) + Lambda endurecida |
| Z2 – Raw                  | S3 privado + SSE-S3 + versionamento                          | S3 privado + SSE-KMS (CMK) + Data Events + Macie + Config Rules                |
| Z3 – Curated              | S3 privado + SSE-S3 + Lambda DQ + Glue + Athena              | S3 SSE-KMS + Lake Formation (row/column) + Macie + acesso refinado             |
| Z8 – Security “Trust”     | IAM mínimo + SSM Parameter Store + SSE-S3                    | IAM forte + Secrets Manager + SSM SecureString + CMKs + Security Hub + GuardDuty + Config |
| Z9 – Observability & Audit| CloudWatch Logs + métricas básicas + CloudTrail Mgmt events  | Logs full + métricas avançadas + CloudTrail (Mgmt + Data) + Security Lake/SIEM + automação de resposta |

---

## 3. Comparação Detalhada por Componente

### 3.1. Z0 — Fontes de Dados

| Item                     | Lab                                                           | Prod                                                                                   |
|--------------------------|---------------------------------------------------------------|----------------------------------------------------------------------------------------|
| Tipo de fonte            | Simulada (scripts HTTP)                                       | Aplicações reais, parceiros, sistemas internos                                        |
| Catálogo de fontes       | `z0_data_sources.yaml` com `source_id`, tipo, schema, etc.    | Catálogo formal (Git/CMDB/Data Catalog) com owner, classificação, base legal, etc.    |
| Contratos                | JSON Schema local                                             | JSON Schema / Avro / Protobuf versionados, mapeados a acordos contratuais             |
| Classificação            | Campo simples (`public`, `internal`, `sensitive`…)            | Classificação alinhada a políticas corporativas + LGPD/GDPR                           |
| Uso principal            | Simular ingestão com dados sintéticos                         | Sustentar fluxo de negócio, modelos e relatórios com dados reais                      |

---

### 3.2. Z1 — Ingestion & Security Gateway

| Item                          | Lab                                                                          | Prod                                                                                                 |
|-------------------------------|------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------|
| Entrada HTTP                  | API Gateway **HTTP API**                                                     | API Gateway HTTP/REST (REGIONAL ou via CloudFront)                                                  |
| Proteção de borda            | **Sem WAF**                                                                  | **AWS WAF** (managed + custom rules) na frente do API Gateway                                       |
| Autenticação                  | API Key simples (`x-api-key`)                                               | API Key **+** JWT (Cognito/IdP) e/ou mTLS, conforme o tipo de cliente                               |
| Rate limit / quota            | Usage Plans básicos                                                          | Rate limit avançado (WAF + API GW), quotas por parceiro/canal                                       |
| Validação de payload          | JSON Schema + regras básicas na Lambda                                      | JSON Schema + regras fortes + sanitização, com preocupação de ataques de input                      |
| Logs                          | CloudWatch Logs (API GW + Lambda)                                           | CloudWatch Logs + export para Security Lake/SIEM + WAF logs                                         |
| Detecção de abuso/ataques     | Manual via CloudWatch                                                        | WAF + GuardDuty + Security Hub + dashboards e alarmes                                               |

---

### 3.3. Z2 — Data Lake Bruto (Raw)

| Item                            | Lab                                                      | Prod                                                                                   |
|---------------------------------|----------------------------------------------------------|----------------------------------------------------------------------------------------|
| Bucket S3 Raw                   | Privado + Block Public Access                            | Privado + Block Public Access                                                          |
| Criptografia                    | **SSE-S3** (chaves gerenciadas pelo S3)                  | **SSE-KMS (CMK dedicada)**                                                             |
| Versionamento                   | Ativado                                                  | Ativado                                                                                |
| Logging                         | Opcional / simples                                       | Logging de acesso S3 + CloudTrail Data Events                                          |
| Descoberta de dados sensíveis   | Não há ferramenta gerenciada                             | **Amazon Macie** escaneando Raw                                                        |
| Governança de config            | Manual                                                    | **AWS Config + Config Rules** monitorando S3, KMS, CloudTrail, IAM, etc.              |

---

### 3.4. Z3 — Dados Curados & Data Products

| Item                            | Lab                                                              | Prod                                                                                           |
|---------------------------------|------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Bucket S3 Curated               | Privado + SSE-S3 + versionamento                                | Privado + SSE-KMS + versionamento + Data Events                                               |
| Curadoria / DQ                  | Lambda simples (limpeza + mascaramento básico)                  | Lambda/Step Functions com regras fortes de DQ + mascaramento/anonimização robusta             |
| Catálogo                        | Glue Data Catalog + Athena                                      | Glue + Athena **+ Lake Formation** para acesso refinado (tabela/coluna/linha)                |
| Controle de acesso a dados      | IAM simples (em nível de bucket/tabela)                         | Lake Formation + IAM/SSO + políticas baseadas em domínio, sensibilidade, função de negócio   |
| Descoberta de PII               | Via lógica de Lambda (explícita)                                | Macie + validação automática dos buckets Curated                                              |

---

### 3.5. Z8 — Security & Trust Services

| Item                         | Lab                                                                 | Prod                                                                                                    |
|------------------------------|---------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| IAM                          | Roles mínimo por Lambda                                             | IAM desenhado por função (ingest, curadoria, consumo, operações, segurança) + uso de conditions        |
| Segredos                     | SSM Parameter Store (Standard)                                     | **AWS Secrets Manager** + SSM SecureString, com rotação automática para segredos críticos              |
| Criptografia                 | SSE-S3 em buckets                                                   | SSE-KMS com **CMKs dedicadas** (por domínio/ambiente)                                                  |
| Detecção gerenciada          | Não há                                                              | **GuardDuty** habilitado (CloudTrail, VPC Flow Logs, DNS Logs)                                         |
| Postura de segurança         | Manual                                                              | **Security Hub** agregando findings (GuardDuty, Macie, Config, Inspector, WAF, etc.)                   |
| Governança de config         | Manual                                                              | **AWS Config + Config Rules** + integração com Security Hub                                            |

---

### 3.6. Z9 — Observabilidade, Auditoria & Resposta

| Item                        | Lab                                                             | Prod                                                                                                         |
|-----------------------------|------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Logs                        | CloudWatch Logs (API GW + Lambdas)                              | CloudWatch Logs + WAF logs + export p/ S3/Security Lake/SIEM                                                |
| Métricas                    | Métricas básicas & alarmes simples                              | Métricas detalhadas, dashboards por domínio, alarmes contextuais                                           |
| CloudTrail                  | Management Events habilitados                                   | **Management + Data Events** (S3, etc.) + organização multi-conta (org trail)                              |
| Centralização de logs       | Não há                                                            | **Amazon Security Lake ou SIEM externo** para correlação/hunting/compliance                               |
| Resposta a incidentes       | Manual, baseada em CloudWatch / CloudTrail                      | Playbooks (EventBridge + Lambda/Step Functions/SOAR) + runbooks de IR                                      |

---

## 4. Mapeamento Controles — Lab vs Prod

### 4.1. Tabela “Controle → Lab → Prod”

| Controle / Tema                      | Lab (Implementado?)                               | Prod (Implementado?)                                   | Comentário / Evolução                                      |
|-------------------------------------|---------------------------------------------------|--------------------------------------------------------|------------------------------------------------------------|
| Catálogo de fontes (Z0)            | ✅ YAML + schemas                                  | ✅ Catálogo formal                                     | Evoluir de arquivo para repositório + governança          |
| API Key por fonte (Z1)             | ✅                                                | ✅ (complementado por JWT/mTLS)                       | Em Prod, API Key vira **só um pedaço** da autenticação    |
| WAF na frente do API Gateway       | ❌                                                | ✅ AWS WAF (managed + custom rules)                    | Feature típica “Prod only”                                 |
| Validação de schema e regras       | ✅ Lambda                                          | ✅ Lambda (mais rígida e com sanitização)             | Lab já prepara, Prod endurece                              |
| S3 privado                         | ✅ Block Public Access                             | ✅ idem                                                | Igual, mas com Config Rules validando em Prod             |
| Criptografia buckets               | ✅ SSE-S3                                          | ✅ SSE-KMS (CMK)                                       | Troca SSE-S3 → SSE-KMS quando sair do lab                 |
| Versionamento em S3                | ✅                                                | ✅                                                    | Mantido                                                    |
| Macie (descoberta de PII)          | ❌                                                | ✅                                                    | Rodar scans regulares em Raw/Curated                      |
| GuardDuty                          | ❌                                                | ✅                                                    | Detecção gerenciada de ameaças                            |
| Security Hub                       | ❌                                                | ✅                                                    | Painel central de posture e findings                      |
| AWS Config + Config Rules          | ❌                                                | ✅                                                    | Evita “drift” de config em Prod                           |
| Secrets Manager                    | ❌ (uso de SSM Standard)                          | ✅ + SSM SecureString                                  | Segredos críticos vão para Secrets Manager                |
| CloudTrail (Mgmt events)           | ✅                                                | ✅ Mgmt + Data Events                                 | Em Prod, Data Events são importantes para forense         |
| SIEM / Security Lake               | ❌                                                | ✅                                                    | Opcional no lab, altamente recomendado em Prod            |
| Lake Formation (governança dados)  | ❌                                                | ✅                                                    | Controle refinado sobre acesso a dados curados            |

---

## 5. Roadmap de Evolução: do Lab para Prod

Uma forma prática de migrar do lab para produção é seguir **fases de hardening**.

### 5.1. Fase 1 — “Foundation” (Baseline de Segurança)

- Garantir que tudo que já existe no lab está **corretamente implementado**:
  - API GW + Lambda + S3 Raw/Curated + IAM mínimo;
  - CloudWatch Logs + CloudTrail (Mgmt);
  - SSE-S3 e buckets privados.
- Introduzir:
  - **SSE-KMS** nos buckets mais sensíveis (começar pela Z3);
  - Configuração básica do **GuardDuty** e **Security Hub**.

### 5.2. Fase 2 — Borda & Dados Sensíveis

- Colocar **AWS WAF** na frente do API Gateway.
- Ajustar autenticação:
  - API Key + Cognito/JWT ou mTLS para parceiros mais críticos.
- Habilitar **Macie**:
  - primeiro em buckets Curated;
  - depois em Raw, conforme necessidade/custo.

### 5.3. Fase 3 — Governança & Compliance

- Ativar e calibrar **AWS Config + Config Rules**:
  - S3 público → não conforme;
  - buckets sem criptografia → não conforme;
  - CloudTrail desativado → não conforme.
- Expandir uso de **SSE-KMS (CMKs)**:
  - separar por domínio / ambiente / sensibilidade.
- Introduzir **Lake Formation** para acesso refinado a dados.

### 5.4. Fase 4 — Observabilidade, Hunting & Resposta

- Implantar **Security Lake ou SIEM**:
  - centralizar CloudTrail, WAF logs, GuardDuty, Macie, Security Hub, etc.
- Criar **playbooks de resposta a incidentes** (EventBridge + Lambda / SOAR).
- Produzir **dashboards executivos e operacionais**:
  - postura de segurança;
  - tendências de findings;
  - volumetria de ingestão x incidentes.

---

## 6. Quando usar cada documento

- Use `architecture-lab.md` + `SECURITY.md` quando:
  - estiver rodando o lab em conta pessoal;
  - quiser explicar o conceito para estudo;
  - precisar de um ambiente barato para experimentar.

- Use `architecture-prod.md` + `SECURITY-prod.md` quando:
  - for desenhar o **alvo de produção**;
  - estiver em revisão com áreas de segurança, risco, arquitetura corporativa;
  - precisar justificar uso de serviços pagos (WAF, GuardDuty, Macie, etc.).

- Use `Lab-vs-Prod.md` (este arquivo) quando:
  - quiser explicar “o que falta para virar Prod”;
  - montar um **roadmap de hardening**;
  - preparar apresentações para gestão / stakeholders.

---

Este `Lab-vs-Prod.md` é um documento de **ponte**: ele existe para deixar explícito que o lab não é um “ambiente tosco”, mas sim um **subconjunto controlado e barato** de uma arquitetura de produção muito mais robusta.
