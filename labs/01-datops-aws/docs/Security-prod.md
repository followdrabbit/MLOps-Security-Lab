# 🛡️ SECURITY — Arquitetura de Produção DatOps em AWS

## 1. Objetivo

Este documento descreve a **visão de segurança em produção** da pipeline DatOps (Z0–Z3) da plataforma **MLOps Security Lab**, considerando:

- Ameaças relevantes para **ambientes produtivos**;
- **Controles de segurança completos** (WAF, GuardDuty, Security Hub, Macie, KMS/CMK, Config Rules, Security Lake/SIEM, etc.);
- Integração com serviços de **monitoramento, detecção e resposta**;
- Relação com o **lab** (versão de baixo custo) descrito em:
  - `architecture-lab.md`
  - `SECURITY.md` (versão lab)

O objetivo é servir como **estado-alvo de segurança** para quando a pipeline DatOps sair do contexto de laboratório e for usada com **dados reais**, **parceiros externos** e **requisitos de compliance**.

---

## 2. Escopo

Esta visão de segurança é aplicável à **arquitetura de produção** descrita em `architecture-prod.md`, cobrindo:

- **Z0 — Fontes de Dados (Prod)**
- **Z1 — Ingestion & Security Gateway**  
  (WAF + API Gateway + Lambda)
- **Z2 — Data Lake Bruto (Raw)**  
  (S3 Raw com SSE-KMS + Data Events)
- **Z3 — Dados Curados & Data Products**  
  (S3 Curated + Glue + Athena/Lake Formation)
- **Z8 — Security & Trust Services (Full)**
- **Z9 — Monitoring, Observability & Audit (Full)**

Fora de escopo:

- Detalhes de **multi-conta/landing zone** (tratado em arquitetura de organização AWS).
- Estratégias multi-região e DR avançado.
- Ferramentas de terceiros (DLP/CASB, WAF externos, etc.), embora possam complementar.

---

## 3. Visão Geral de Segurança em Produção

Em produção, a pipeline DatOps deixa de ser apenas “um lab seguro” e passa a:

- Processar **dados reais**, possivelmente contendo PII/sensíveis;
- Ser acessada por **parceiros externos**, sistemas legados e aplicações de negócio;
- Estar sujeita a **requisitos de compliance** (LGPD/GDPR, normas internas, auditorias).

Por isso, a arquitetura de produção adiciona:

- Camada de **proteção de borda** (AWS WAF);
- Camada de **detecção gerenciada** (GuardDuty, Macie, Inspector, etc., via Security Hub);
- **Criptografia avançada** com KMS e CMKs dedicadas;
- **Governança contínua de configuração** via AWS Config + Config Rules;
- **Centralização de logs e findings** em Security Lake/SIEM, habilitando resposta a incidentes.

---

## 4. Modelo de Ameaças (Produção)

Abaixo, um resumo das principais ameaças consideradas, agora com visão de produção.

### 4.1. Bordas & Ingestão (Z0–Z1)

- **P1 — Ataques de aplicação / OWASP (L7)**  
  SQLi, XSS, LFI/RFI, ataques de injection em geral contra o endpoint de ingestão.
- **P2 — Abuso de endpoint / DDoS lógico**  
  Tentando degradar serviço por alto volume de requisições.
- **P3 — Credenciais comprometidas / API Keys roubadas**  
  Uso indevido de credenciais de parceiros/clientes.

### 4.2. Armazenamento de Dados (Z2–Z3)

- **P4 — Vazamento de dados em S3**  
  Buckets públicos, permissões excessivas, chaves de acesso comprometidas.
- **P5 — Vazamento de PII / dados sensíveis**  
  Dados pessoais/sensíveis expostos em zonas indevidas (Raw/Curated).
- **P6 — Uso indevido de dados**  
  Acesso além da necessidade (privilégios excessivos, falta de segregação).

### 4.3. Governança, Configuração & Criptografia (Z8)

- **P7 — Configuração insegura ou “drift”**  
  Buckets que eram seguros se tornarem públicos; CloudTrail desligado; KMS mal configurado.
- **P8 — Gestão inadequada de segredos**  
  Segredos em código, em texto plano ou com falta de rotação.

### 4.4. Observabilidade, Detecção & Resposta (Z9)

- **P9 — Falta de detecção precoce de comprometimentos**  
  Atividades maliciosas não são identificadas a tempo.
- **P10 — Falta de visibilidade cross-conta / cross-serviço**  
  Dificuldade de correlacionar eventos entre múltiplos serviços.
- **P11 — Incidentes sem resposta coordenada**  
  Sem playbooks, sem automação de resposta, sem processo de IR.

---

## 5. Controles de Segurança por Camada (Produção)

### 5.1. Proteção de Borda & Ingestão (WAF + API Gateway + Lambda)

**Componentes principais:**

- AWS WAF (Web ACL)
- API Gateway (REST/HTTP API)
- Lambda `ingestion_lambda`
- Autenticação forte (API Key + JWT/mTLS/Cognito, conforme caso)

**Controles:**

1. **AWS WAF na frente do API Gateway**
   - Regras gerenciadas:
     - AWSManagedRulesCommonRuleSet
     - SQLi, XSS, LFI/RFI, NoSQLi, etc.
   - Regras customizadas:
     - Rate limiting por IP/faixa/ASN;
     - Bloqueio por geolocalização;
     - Allowlist/denylist de IPs/ASN/partners.
   - Logs do WAF enviados para:
     - CloudWatch Logs e/ou
     - S3 + Security Lake.

2. **API Gateway (EndPoints REGIONAL ou via CloudFront)**
   - TLS 1.2+ obrigatório;
   - Autenticação:
     - API Keys + Usage Plans **e**/ou
     - JWT (Cognito/IdP corporativo) ou mTLS;
   - Rate limit & quotas:
     - Restrições por parceiro/canal;
     - Proteção contra abuso lógico (P2).

3. **Lambda `ingestion_lambda` endurecida**
   - Validação *estrita* de schema e regras de negócio;
   - Defesa contra inputs maliciosos (sanitize + validações adicionais);
   - Logging estruturado (sem dados sensíveis em logs);
   - Dependências gerenciadas via pipeline com SCA/assinatura (supply chain).

**Ameaças mitigadas:**

- P1 (ataques OWASP usando WAF + validações);
- P2 (abuso via rate limit WAF + API Gateway);
- P3 (API Keys comprometidas → monitoramento & rotação; mitigação parcial, não total).

---

### 5.2. Armazenamento de Dados (S3 Raw & Curated com SSE-KMS)

**Componentes:**

- Buckets S3 Raw e Curated (prod)
- AWS KMS (CMKs dedicadas)
- CloudTrail (Management + Data Events)
- Amazon Macie

**Controles:**

1. **Criptografia SSE-KMS com CMKs dedicadas**
   - CMKs específicas, ex.:
     - `cmk-datalake-raw-prod`
     - `cmk-datalake-curated-prod`
   - Políticas de chave:
     - Apenas roles de Lambda e serviços autorizados podem usar a CMK;
     - Restrições por conta/região/VPC endpoint quando possível.
   - Logs de uso das chaves visíveis em CloudTrail.

2. **Buckets 100% privados**
   - Block Public Access = ON;
   - Policies rígidas, sem wildcard permissivo;
   - ACLs desativadas/limitadas.

3. **CloudTrail Data Events para S3**
   - Tracking de `GetObject`, `PutObject`, `DeleteObject`;
   - Exportados para S3 + Security Lake/SIEM.

4. **Amazon Macie para discovery de dados sensíveis**
   - Scans regulares de:
     - buckets Raw;
     - buckets Curated;
   - Findings enviados a Security Hub;
   - Uso para:
     - identificar PII mal posicionada;
     - validar se tratamentos de curadoria estão corretos.

5. **Versionamento e políticas de retenção**
   - Versionamento ON;
   - Políticas de lifecycle para:
     - retenção legal;
     - limpeza segura de dados expirados.

**Ameaças mitigadas:**

- P4 (vazamento por S3 mal configurado);
- P5 (PII exposta sem controle);
- P6 (uso indevido — mitigado em conjunto com IAM/Lake Formation).

---

### 5.3. Governança de Acesso a Dados (Glue, Athena, Lake Formation)

**Componentes:**

- AWS Glue Data Catalog
- Amazon Athena
- AWS Lake Formation (recomendado)
- IAM + Identity Center (SSO/IdP)

**Controles:**

1. **Catálogo centralizado (Glue)**
   - Tabelas de Z3 devidamente classificadas (tags de domínio, sensibilidade, owner).

2. **Lake Formation (row/column level security)**
   - Políticas definindo:
     - Quem pode ver qual tabela;
     - Quem pode ver quais colunas (masking) — ex.: mascarar CPF;
     - Opcionalmente, filtros de linha (por tenant, região, etc.).

3. **Integração com IdP / IAM Identity Center**
   - Controle de acesso baseado em grupos/funções:
     - Squads de produto;
     - Times de risco/compliance;
     - Equipe de segurança.

**Ameaças mitigadas:**

- P5 (PII vazando em relatórios);
- P6 (uso indevido / excesso de acesso).

---

### 5.4. Z8 — Security & Trust Services (Full)

**Componentes:**

- IAM (roles/policies)
- AWS Secrets Manager / SSM Parameter Store (SecureString)
- AWS KMS (CMKs)
- WAF, GuardDuty, Security Hub, Macie, Config

**Controles:**

1. **IAM com privilégio mínimo e separação de funções**
   - Roles específicas:
     - ingestão, curadoria, consulta, operação, segurança;
   - Policies explícitas e restritivas;
   - Uso de conditions (ex.: `aws:SourceVpce`, `aws:PrincipalOrgID`).

2. **Gerenciamento de segredos**
   - **AWS Secrets Manager**:
     - credenciais de bancos, APIs sensíveis, etc.;
     - rotação automática configurada;  
   - **SSM Parameter Store (SecureString)**:
     - configs sensíveis de menor criticidade;
   - Nenhum segredo em:
     - código-fonte,
     - Terraform/CloudFormation em texto puro,
     - variáveis de ambiente sem criptografia.

3. **Integração com Security Hub**
   - Security Hub como **agregador de findings** de:
     - GuardDuty;
     - Macie;
     - Config Rules;
     - Inspector (se usado);
     - WAF (via integradores).

4. **Amazon GuardDuty**
   - Habilitado na organização/conta;
   - Analisa:
     - CloudTrail;
     - VPC Flow Logs;
     - DNS Logs;
   - Detecta atividades suspeitas (ex.: exfiltração, comportamento anômalo, credenciais comprometidas).

5. **AWS Config + Config Rules**
   - Regras pelo menos para:
     - S3 público → não conforme;
     - buckets sem criptografia → não conforme;
     - CloudTrail desativado → não conforme;
     - KMS sem rotação → não conforme;
     - IAM com policies muito amplas → alerta.
   - Integração com:
     - Security Hub (alertas);
     - EventBridge + Lambda (remediação automática em alguns casos).

**Ameaças mitigadas/endisadas:**

- P4 (configuração insegura de S3);
- P7 (drift de configuração);
- P8 (segredos mal geridos);
- P3 (credenciais comprometidas — mitigação com detecção + rotação).

---

### 5.5. Z9 — Monitoring, Observability & Audit (Full)

**Componentes:**

- CloudWatch Logs, Metrics & Alarms
- CloudTrail (Management + Data Events)
- Amazon Security Lake ou SIEM externo (Splunk, Elastic, etc.)
- EventBridge + Lambda / SOAR para automação de resposta

**Controles:**

1. **CloudWatch Logs & Metrics**
   - Logs de:
     - API Gateway;
     - Lambdas;
     - WAF;
     - outros serviços relevantes.
   - Métricas e dashboards:
     - latência;
     - taxa de erro por endpoint;
     - volume de dados processados;
     - volume de registros em quarentena;
     - número de bloqueios no WAF.

2. **CloudTrail Full**
   - Management Events para toda a conta;
   - Data Events para:
     - S3 Raw/Curated;
     - outras fontes críticas;
   - Exportação para S3 + Security Lake.

3. **Security Lake / SIEM**
   - Centralização de:
     - CloudTrail;
     - CloudWatch Logs;
     - WAF logs;
     - Findings do Security Hub;
     - VPC Flow Logs.
   - Uso em:
     - threat hunting;
     - correlação de incidentes;
     - compliance reporting.

4. **Automação de resposta (EventBridge + Lambdas / SOAR)**
   - Exemplo de playbooks:
     - Encontrou bucket S3 público → remover permissão e alertar;
     - Encontrou chave comprometida (GuardDuty) → revogar e notificar o time;
     - Alto volume de bloqueios no WAF → aumentar sensibilidade/regra específica.

**Ameaças mitigadas:**

- P9 (falta de detecção precoce);
- P10 (falta de visibilidade cross-serviço);
- P11 (falta de resposta coordenada).

---

## 6. Tabela Resumida — Ameaças x Controles em Produção

| ID | Ameaça                                         | Camadas principais                   | Controles em Produção                                             |
|----|-----------------------------------------------|--------------------------------------|--------------------------------------------------------------------|
| P1 | Ataques OWASP (SQLi, XSS, etc.)               | Z1 (WAF + API GW + Lambda)          | AWS WAF (managed/custom rules), validação forte na Lambda         |
| P2 | Abuso de endpoint / DDoS lógico               | Z1                                  | Rate limit WAF + quotas API GW + monitoração CloudWatch           |
| P3 | Credenciais/API Keys comprometidas            | Z1, Z8                              | Rotação de segredos, GuardDuty, Security Hub, IAM/least privilege |
| P4 | Vazamento de dados em S3                      | Z2, Z3, Z8                          | SSE-KMS, Block Public Access, Config Rules, Macie, CloudTrail     |
| P5 | Exposição de PII/dados sensíveis              | Z2, Z3, Z8                          | Macie, curadoria com masking, Lake Formation / column-level       |
| P6 | Uso indevido de dados (acesso excessivo)      | Z3, Z8                              | Lake Formation, IAM, Identity Center, políticas de mínimo acesso  |
| P7 | Drift de configuração                         | Z8                                  | AWS Config + Config Rules, Security Hub, remediação automática    |
| P8 | Gestão inadequada de segredos                 | Z8                                  | Secrets Manager, SSM SecureString, rotação automática             |
| P9 | Falta de detecção precoce                     | Z9                                  | GuardDuty, Macie, Security Hub, CloudWatch Alarms                 |
| P10| Falta de visibilidade integrada               | Z9                                  | Security Lake / SIEM centralizando logs & findings                |
| P11| Incidentes sem resposta coordenada            | Z9                                  | Playbooks (EventBridge + Lambda/SOAR), runbooks de IR             |

---

## 7. Relação com o Lab (Lab vs Prod)

A pipeline de produção é, conceitualmente, a **evolução direta do lab**, com:

- **Mesma topologia Z0–Z3**,  
- Mas com **Z8/Z9 “full”** ao invés de “lite”.

Resumo:

- Tudo que existe no **lab** (API Key, validação de schema, S3 privado + SSE-S3, logs, IAM mínimo)  
  ⇒ é **mantido** em produção.
- A produção **adiciona**:
  - WAF, GuardDuty, Security Hub, Macie, Config,
  - SSE-KMS com CMKs dedicadas,
  - CloudTrail Data Events,
  - Security Lake/SIEM,
  - Lake Formation para acesso refinado a dados.

O `SECURITY.md` (lab) pode ser lido como:

> “O que eu tenho hoje no ambiente de estudo.”

O `SECURITY-prod.md` (este documento) responde:

> “Como esse lab precisa evoluir para ficar com cara de **arquitetura enterprise de produção**.”

---

## 8. Próximos Passos

- Definir **IaC (Terraform/CDK/CloudFormation)** para:
  - WAF, GuardDuty, Security Hub, Macie;
  - KMS/CMKs, Config Rules;
  - Security Lake/SIEM.
- Criar **ADRs** para:
  - estratégia de WAF (REST direto vs HTTP API via CloudFront);
  - granularidade de CMKs (por domínio, ambiente, tenant);
  - uso de Lake Formation e níveis de segurança de dados.
- Integrar esta visão com:
  - `RUNBOOK-IR.md` (runbooks de resposta a incidentes);
  - `COMPLIANCE.md` (mapeamento para LGPD/GDPR, NIST, CIS, etc., se aplicável).

---

Este `SECURITY-prod.md` deve ser mantido em sincronia com:

- `architecture-prod.md` (arquitetura de produção);  
- `architecture-lab.md` e `SECURITY.md` (versão lab).

Ele representa o **estado de segurança desejado** para quando a pipeline DatOps estiver em **produção real**.
