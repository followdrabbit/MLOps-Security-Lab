# 🧨 Threat Model — DatOps em AWS (Z0–Z3 + Z8/Z9)

## 1. Objetivo

Este documento descreve o **modelo de ameaças** da pipeline DatOps (Z0–Z3) da plataforma **MLOps Security Lab**, com foco em:

- Identificar **ativos críticos** e **fronteiras de confiança**;
- Mapear ameaças usando **STRIDE**;
- Relacionar ameaças aos **controles existentes no Lab** e aos **controles recomendados para Produção**;
- Servir de base para:
  - decisões de arquitetura,
  - priorização de controles,
  - criação de baselines de segurança.

> Este documento complementa:
> - `architecture-lab.md` e `SECURITY.md`
> - `architecture-prod.md` e `SECURITY-prod.md`
> - `Lab-vs-Prod.md` e `Observability.md`

---

## 2. Escopo

O threat model cobre:

- **Z0 — Fontes de Dados**
- **Z1 — Ingestion & Security Gateway** (API Gateway + Lambda)
- **Z2 — Raw Data Lake (S3 Raw)**
- **Z3 — Curated Data & Data Products (S3 Curated + Glue + Athena)**
- **Z8 — Security & Trust Services**
- **Z9 — Monitoring, Observability & Audit**

Ele considera dois contextos:

- **Lab** → ambiente de estudo, baixo custo, dados sintéticos;
- **Produção** → ambiente real, com dados sensíveis e requisitos de compliance.

---

## 3. Metodologia de Threat Modeling

### 3.1. Abordagem utilizada

A abordagem utilizada combina:

- **Modelagem por zonas (Z0–Z3 + Z8/Z9)**  
  Cada zona é analisada em termos de:
  - ativos que processa ou protege;
  - ameaças relevantes;
  - controles existentes / recomendados.

- **STRIDE** como taxonomia de ameaças:
  - **S**poofing (falsificação de identidade)
  - **T**ampering (modificação indevida de dados)
  - **R**epudiation (negação de ações)
  - **I**nformation Disclosure (vazamento de informação)
  - **D**enial of Service (indisponibilidade)
  - **E**levation of Privilege (elevação de privilégios)

### 3.2. Níveis de análise

- **Nível 1 — Alto nível (este documento)**  
  Principalmente para:
  - visão de riscos;
  - priorização de controles;
  - comunicação com times de arquitetura, segurança e negócio.

- **Nível 2 — Específico por componente**  
  Pode ser detalhado futuramente em documentos separados, ex.:
  - threat model específico da **Lambda de Ingestão**,
  - threat model do **Data Lake (S3)**,
  - threat model da **exposição de dados via Athena**.

---

## 4. Ativos e Fronteiras de Confiança

### 4.1. Ativos principais

- **Dados em trânsito**
  - Requisições HTTP(S) de ingestão (Z0 → Z1).
- **Dados em repouso**
  - Arquivos brutos em S3 Raw (Z2).
  - Dados curados em S3 Curated (Z3).
- **Metadados**
  - Catálogo de fontes (`z0_data_sources.yaml`).
  - JSON Schemas.
  - Glue Data Catalog (tabelas, schemas).
- **Segredos & Credenciais**
  - API Keys de fontes.
  - Segredos de integração armazenados em SSM/Secrets Manager.
- **Configurações de segurança**
  - Policies de IAM.
  - Policies KMS/CMK.
  - Regras de WAF.
  - Config Rules.

### 4.2. Fronteiras de confiança

1. **Fronteira Externa → AWS (Z0 → Z1)**
   - Chamadas de parceiros, sistemas internos e apps externos para o endpoint de ingestão.
   - Em produção, passa por **WAF + API Gateway**.

2. **Fronteira Z1 → Z2 (Ingestão → Data Lake Raw)**
   - Lambda grava dados validados/quarentenados em S3 Raw.

3. **Fronteira Z2 → Z3 (Raw → Curated)**
   - Lambda de curadoria lê dados brutos, aplica DQ e anonimização e grava em Z3.

4. **Fronteira Z3 → Consumidores**
   - Acesso a dados via Athena (Lab) e Lake Formation/consumidores (Produção).

5. **Fronteira de Gestão & Segurança (Z8/Z9)**
   - IAM, KMS, Secrets Manager, GuardDuty, Security Hub, Config, Logging/Auditoria.

---

## 5. Ameaças por Zona (STRIDE)

Abaixo, um resumo das principais ameaças por zona, com IDs reutilizáveis.

### 5.1. Z0 — Fontes de Dados

| ID       | STRIDE | Descrição                                                                                  | Impacto típico                            |
|----------|--------|--------------------------------------------------------------------------------------------|-------------------------------------------|
| Z0-S1    | S      | Fonte se passando por outra (uso indevido de API Key ou credencial)                       | Dados maliciosos ou não autorizados na pipeline |
| Z0-T1    | T      | Manipulação de payload para burlar validações fracas                                      | Data poisoning, corrupção de dataset      |
| Z0-R1    | R      | Fonte nega ter enviado dados, sem trilha adequada                                         | Dificuldade de atribuição e forense       |
| Z0-I1    | I      | Envio de dados mais sensíveis do que o permitido pelo contrato                            | Vazamento indireto via pipeline           |

---

### 5.2. Z1 — Ingestion & Security Gateway

| ID       | STRIDE | Descrição                                                                                | Impacto típico                             |
|----------|--------|------------------------------------------------------------------------------------------|--------------------------------------------|
| Z1-S1    | S      | Uso de API Keys comprometidas                                                            | Acesso indevido à ingestão                 |
| Z1-T1    | T      | Payload malicioso buscando explorar vulnerabilidades na Lambda/API                       | Execução inesperada, bypass de regras      |
| Z1-R1    | R      | Ausência de logs confiáveis para requests críticos                                      | Dificuldade em investigar incidentes       |
| Z1-I1    | I      | Respostas de erro expondo detalhes internos (stack traces, mensagens sensíveis)         | Vazamento de informação técnica            |
| Z1-D1    | D      | Ataques de volumetria (DoS lógico) contra o endpoint de ingestão                        | Indisponibilidade da pipeline              |
| Z1-E1    | E      | Falha na autenticação/autorização permitindo uso de endpoint sem controles necessários  | Uso indevido, ingestão por atores não-autorizados |

---

### 5.3. Z2 — Raw Data Lake (S3 Raw)

| ID       | STRIDE | Descrição                                                                                 | Impacto típico                          |
|----------|--------|-------------------------------------------------------------------------------------------|-----------------------------------------|
| Z2-S1    | S      | Uso de credenciais IAM comprometidas para ler dados em Raw                                | Vazamento de dados brutos               |
| Z2-T1    | T      | Alteração ou deleção maliciosa de objetos em Raw                                         | Corrupção de histórico, perda de trilha |
| Z2-R1    | R      | Falta de CloudTrail Data Events para operações sensíveis                                 | Dificuldade de auditoria                |
| Z2-I1    | I      | Bucket exposto (por erro de config)                                                      | Exposição pública de dados              |
| Z2-D1    | D      | Carga excessiva de escrita/leitura causando degradação de performance e custos elevados  | Impacto financeiro e de disponibilidade |

---

### 5.4. Z3 — Curated Data & Data Products

| ID       | STRIDE | Descrição                                                                                  | Impacto típico                                |
|----------|--------|--------------------------------------------------------------------------------------------|-----------------------------------------------|
| Z3-S1    | S      | Uso de credenciais de consumo de dados por usuários não autorizados                       | Acesso indevido a dados curados               |
| Z3-T1    | T      | Manipulação de datasets curados (sem versionamento/controle adequado)                     | Insights/modelos baseados em dados incorretos |
| Z3-I1    | I      | Exposição de PII em tabelas acessíveis amplamente (sem mascaramento)                      | Risco LGPD/GDPR e danos à privacidade         |
| Z3-D1    | D      | Consultas pesadas/abuso de Athena causando aumento de custo e degradação                   | Impacto financeiro/operacional                |
| Z3-E1    | E      | Falhas em controles de Lake Formation (Prod) permitindo privilégios maiores que o necessário | Violação de least privilege, vazamento de dados |

---

### 5.5. Z8 — Security & Trust Services

| ID       | STRIDE | Descrição                                                                                  | Impacto típico                             |
|----------|--------|--------------------------------------------------------------------------------------------|--------------------------------------------|
| Z8-S1    | S      | Comprometimento de credenciais de administração (IAM, KMS)                                | Controle total ou parcial do ambiente       |
| Z8-T1    | T      | Alteração maliciosa de policies (IAM, KMS, S3)                                            | Bypass de controles, criação de backdoors   |
| Z8-R1    | R      | Falta de logs confiáveis para mudanças de segurança                                        | Dificuldade para auditoria e IR            |
| Z8-I1    | I      | Exposição indevida de segredos (Secrets Manager/SSM)                                      | Vazamento de credenciais                   |
| Z8-E1    | E      | Privilégios excessivos (IAM overprivileged, falta de separação de funções)                | Elevação de privilégio e movimentos laterais |

---

### 5.6. Z9 — Monitoring, Observability & Audit

| ID       | STRIDE | Descrição                                                                                  | Impacto típico                                     |
|----------|--------|--------------------------------------------------------------------------------------------|----------------------------------------------------|
| Z9-T1    | T      | Alteração de logs/auditoria (log tampering)                                               | Distorção de evidências                            |
| Z9-R1    | R      | Lacunas em logging/auditoria                         | Atividades críticas sem rastreabilidade           |
| Z9-I1    | I      | Logs contendo dados sensíveis sem mascaramento                                            | Vazamento de informação via logs                   |
| Z9-D1    | D      | Falha em alarmes (não configurados, ruído excessivo ou silenciados)                       | Incidentes não detectados a tempo                  |

---

## 6. Matriz Ameaça x Controles (Lab vs Prod)

Resumo simplificado dos principais riscos e como eles são tratados no **Lab** e na **Produção**.

| ID Ameaça | Zona | Risco (resumo)                                     | Lab — Controles atuais                                  | Prod — Controles adicionais                                 |
|-----------|------|-----------------------------------------------------|---------------------------------------------------------|-------------------------------------------------------------|
| Z1-S1     | Z1   | API Key comprometida                               | API Key + logs + IAM mínimo                             | WAF, Auth forte (JWT/mTLS), GuardDuty, rotação de segredos  |
| Z1-D1     | Z1   | DoS lógico na ingestão                              | Usage Plans básicos no API GW                           | WAF rate-based rules + monitoramento avançado              |
| Z2-I1     | Z2   | Exposição de dados em S3                            | S3 privado + SSE-S3 + Block Public Access               | SSE-KMS, Config Rules, Macie, CloudTrail Data Events       |
| Z3-I1     | Z3   | PII em datasets amplamente acessíveis               | Lambda de curadoria com masking básico                  | Lake Formation, Macie, controle fino de acesso             |
| Z8-I1     | Z8   | Vazamento de segredos                               | SSM Parameter Store (Standard), sem segredos no código  | Secrets Manager, SecureString, rotação automática          |
| Z8-E1     | Z8   | IAM com privilégios excessivos                      | IAM mínimo por Lambda (lab)                             | IAM refinado, separação de funções, revisão contínua       |
| Z9-R1     | Z9   | Falta de logs/auditoria confiáveis                  | CloudWatch Logs + CloudTrail Mgmt                       | CloudTrail Data Events, Security Lake/SIEM, playbooks IR   |

---

## 7. Riscos Residuais (Lab)

No **Lab**, mesmo com os controles implementados, permanecem alguns riscos residuais importantes:

1. **Ausência de WAF / detecção gerenciada de ameaças**
   - A ingestão é protegida por API Key e validações básicas, mas não há:
     - proteção contra ataques OWASP de forma gerenciada;
     - detecção de comportamentos suspeitos de rede/conta.

2. **Criptografia básica (SSE-S3)**
   - Sem CMKs dedicadas.
   - Risco residual em casos de requisitos de compliance mais rígidos.

3. **Observabilidade “lite”**
   - Logs e métricas existem, mas:
     - sem correlação centralizada;
     - sem automação de resposta;
     - sem visão consolidada de eventos de segurança.

4. **Controle de acesso a dados curados ainda simplificado**
   - IAM mais simples.
   - Lake Formation não implementado no lab.

> Esses riscos são **aceitáveis no contexto de lab com dados sintéticos**, mas **não aceitáveis em produção**.

---

## 8. Riscos Residuais (Produção)

Mesmo em Produção, após aplicar todos os controles de `SECURITY-prod.md`, alguns riscos residuais ainda existem (como em qualquer sistema):

- **Zero-day em serviços gerenciados da AWS ou libs usadas nas Lambdas**  
  → Mitigação: monitorar boletins, aplicar patching, usar scanners de dependências.

- **Uso indevido legítimo** (usuário com acesso autorizado, mas uso antiético dos dados)  
  → Mitigação: políticas de uso, segregação de funções, monitoramento comportamental (UEBA).

- **Configuração inadequada de regras de WAF / Config / Lake Formation**  
  → Mitigação: revisões periódicas, pentests, revisões de arquitetura e auditorias de segurança.

---

## 9. Próximos Passos

1. **Integrar este threat model com o padrão de identificação de controles**  
   - Associar cada ameaça (ex.: `Z1-D1`) aos controles numerados do baseline (ex.: `AWS.API-GW.WAF.2025.r1`).

2. **Criar ADRs específicos para decisões de segurança**
   - Ex.: “Por que SSE-KMS vs SSE-S3?”,
   - “Por que WAF na frente do API Gateway e não via CloudFront?”.

3. **Evoluir para threat models específicos por componente**
   - Lambda de ingestão (incluindo supply chain de libs),
   - Data Lake S3 + Lake Formation,
   - Exposição de dados via Athena.

4. **Usar o Lab como playground de threat modeling**
   - Ajustar código e configurações com base nas ameaças descritas aqui;
   - Validar se os controles de `SECURITY.md` estão de fato cobrindo o que se espera.

---

Este `threat-model.md` deve ser mantido alinhado com:

- `architecture-lab.md` / `architecture-prod.md`;
- `SECURITY.md` / `SECURITY-prod.md`;
- `Lab-vs-Prod.md` e `Observability.md`.

Ele funciona como **guia central de riscos** para o DatOps, servindo de base para hardening contínuo, priorização de iniciativas e criação de novos controles de segurança.
