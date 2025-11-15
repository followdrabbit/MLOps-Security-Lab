# 🛡️ SECURITY — Lab 01 DatOps em AWS

## 1. Objetivo

Este documento descreve a **visão de segurança** do **Lab 01 — DatOps em AWS** da plataforma MLOps Security Lab, incluindo:

- principais **ameaças** relacionadas ao fluxo de dados Z0–Z3;
- **controles efetivamente implementados** na versão de laboratório (custo mínimo);
- **lacunas conhecidas / riscos residuais** desta versão;
- **controles recomendados para produção**, detalhados em `architecture-prod.md`.

> Este documento complementa:
> - `architecture-lab.md` → foco em arquitetura lógica e componentes;
> - `architecture-prod.md` → visão de como essa arquitetura evolui para um ambiente produtivo com todos os controles de segurança ligados.

---

## 2. Escopo

O escopo deste `SECURITY.md` é o **Lab 01 — DatOps em AWS**, cobrindo:

- Z0 — Fontes de Dados (Simuladas)
- Z1 — Ingestion & Security Gateway (API Gateway + Lambda)
- Z2 — Data Lake Bruto (S3 Raw)
- Z3 — Dados Curados & Data Products (S3 Curated + Glue + Athena)
- Controle transversal “lite” de:
  - Z8 — Security & Trust Services
  - Z9 — Monitoring, Observability & Audit

### Fora do escopo (implementação real em Prod)

- Multi-conta / Landing Zone / organização AWS.
- WAF, GuardDuty, Security Hub, Macie, KMS com CMKs dedicadas, Config Rules, Security Lake/SIEM — **apenas citados como recomendação**.  
  Detalhes completos → `architecture-prod.md`.

---

## 3. Visão Geral de Segurança do Lab

Em alto nível, o Lab 01 foi desenhado para:

- **Simular um pipeline de dados coerente com boas práticas**, porém
- **Minimizar custos fixos** de serviços de segurança gerenciados da AWS.

A estratégia é:

1. **Implementar o “mínimo decente”** em segurança:
   - autenticação por API Key;
   - validação de contrato de dados;
   - S3 privado, criptografado, versionado;
   - IAM com privilégio mínimo;
   - CloudWatch Logs + CloudTrail (management events).
2. **Documentar claramente o “gap” em relação ao ambiente produtivo**:
   - quais serviços faltam (WAF, GuardDuty, Macie, etc.);
   - em que ponto eles entrariam no fluxo;
   - quais riscos continuam sem cobertura no lab.

---

## 4. Modelo de Ameaças (Resumo)

Abaixo, um resumo das principais ameaças consideradas por zona:

### 4.1. Z0 — Fontes de Dados

- **T1 — Data Poisoning / Dados inválidos**
  - Fontes enviando dados fora do contrato, quebrando modelos downstream.
- **T2 — Uso indevido de fontes**
  - Fonte enviando dados sensíveis para um endpoint que não deveria receber esse tipo de informação.

### 4.2. Z1 — Ingestion & Security Gateway

- **T3 — Abuso de endpoint / DoS lógico**
  - Parceiro ou script gerando grande volume de requisições.
- **T4 — Bypass de validações de negócio**
  - Tentativa de enviar payloads malformados ou com valores inconsistentes.
- **T5 — Ataques de aplicação (Prod)**
  - Em produção, ameaças OWASP (SQLi, XSS, etc.) se houvesse endpoints mais complexos.

### 4.3. Z2 — Data Lake Bruto (Raw)

- **T6 — Vazamento de dados**
  - Bucket mal configurado, exposição pública ou permissões excessivas.
- **T7 — Perda ou corrupção acidental de dados**
  - Deleções acidentais, sobrescrita ou alteração silenciosa.

### 4.4. Z3 — Dados Curados & Data Products

- **T8 — Exposição de PII / Dados sensíveis em camadas de consumo**
  - Dados pessoais que não deveriam aparecer em datasets analíticos.
- **T9 — Acesso não autorizado a dados curados**
  - Usuários lendo datasets além do necessário.

### 4.5. Z8/Z9 — Security & Observability

- **T10 — Falta de trilha de auditoria**
  - Dificuldade de reconstruir “quem fez o quê, quando e onde”.
- **T11 — Falta de detecção precoce**
  - Problemas de segurança ou falhas passando despercebidos.

---

## 5. Controles de Segurança Implementados (Versão Lab)

### 5.1. Z0 — Fontes de Dados (Simuladas)

- **Catálogo de fontes (`z0_data_sources.yaml`)**
  - `source_id` por fonte.
  - Tipo (`external`, `partner`, `internal`).
  - Contrato (JSON Schema) referenciado.
  - Classificação de dados (simples).
- **Contratos de dados (JSON Schema)**
  - `transactions_v1.json`, etc.
- **Scripts de exemplo**
  - simulam chamadas HTTP seguindo o contrato.

**Riscos mitigados (parcialmente):**

- Dados fora do contrato (T1).
- Falta de rastreabilidade de origem (T2/T3).

---

### 5.2. Z1 — Ingestion & Security Gateway (API Gateway + Lambda)

**Controles principais:**

1. **Autenticação por API Key**
   - Cada fonte possui uma `x-api-key`.
   - API Gateway valida a chave antes de acionar a Lambda.

2. **Rate Limit / Quota via Usage Plans**
   - Previne abuso involuntário de endpoints.
   - Atenua DoS lógicos moderados (T3).

3. **Validação de contrato e regras de negócio na Lambda**
   - Estrutura: JSON Schema.
   - Regras de negócio simples:
     - `valor > 0`,
     - `data_hora` dentro de janelas razoáveis,
     - enums (canal/status) pré-definidos.

4. **Roteamento raw vs quarentena**
   - Payloads inválidos/suspeitos vão para prefixo `quarantine/`.
   - Mantém visibilidade dos erros sem contaminarem o raw “oficial”.

5. **Logging estruturado (CloudWatch Logs)**
   - `request_id`, `source_id`, `schema_version`, `result`, etc.
   - Suporte a correlação futura (investigações).

**O que *não* está implementado (mas é importante para Prod):**

- WAF (camada 7) na frente do API Gateway.
- JWT/Auth mais fortes (Cognito, IdP corporativo).
- Rate limit avançado (por IP, geolocalização, etc.).

---

### 5.3. Z2 — Data Lake Bruto (Raw)

**Controles principais (lab):**

1. **Bucket S3 Raw privado**
   - Block Public Access = ON.
   - Sem ACLs públicas.
   - Policies de acesso restritas a roles de Lambda e, eventualmente, usuários de troubleshooting.

2. **Criptografia SSE-S3**
   - S3 gerencia as chaves de criptografia.
   - Sem custo extra, reduz a chance de dados “em claro” em storage.

3. **Versionamento habilitado**
   - Permite recuperar versões antigas em caso de deleções acidentais ou corrupção.

4. **Separação de paths (raw vs quarantine)**
   - `raw/source=.../` para payloads aceitos.
   - `quarantine/source=.../` para payloads inválidos.

**Riscos mitigados:**

- Vazamento por exposição pública de bucket (T6) — mitigado via Block Public Access e IAM.
- Perda acidental de dados (T7) — mitigado via versionamento.

---

### 5.4. Z3 — Dados Curados & Data Products

**Controles principais (lab):**

1. **Bucket S3 Curated privado**
   - Mesmas configs de segurança do Raw (Block Public Access + SSE-S3 + versionamento).

2. **Lambda de Curadoria**
   - Aplicação de regras de Data Quality.
   - Normalização de tipos e formatos.
   - Remoção/mascaramento de PII onde aplicável.

3. **Glue Data Catalog + Athena**
   - Separação clara entre:
     - “dados para pipeline” (S3),
     - “como são expostos para consumo” (Glue/Athena).
   - Facilita futura aplicação de políticas de acesso em Prod (Lake Formation, row/column level security, etc.).

**Riscos mitigados:**

- Exposição indevida de PII em datasets curados (T8) — mitigado pelo código de curadoria (em nível de lab).
- Mistura de dados brutos/curados (separação de buckets e prefixes).

---

### 5.5. Z8 “Lite” — Security & Trust Services

**Controles principais (lab):**

1. **IAM com privilégio mínimo**
   - Roles dedicadas:
     - `mlops-datops-lambda-ingestion-role`
     - `mlops-datops-lambda-curation-role`
   - Policies pontuais:
     - `s3:PutObject`/`GetObject` apenas nos buckets/prefixes corretos.
     - Permissões mínimas de CloudWatch Logs.

2. **Segredos no SSM Parameter Store (Standard)**
   - API keys e outros parâmetros sensíveis armazenados fora do código.
   - Acesso via IAM (e não string hardcoded).

3. **Criptografia SSE-S3**
   - Todos os buckets (Raw/Curated) com SSE-S3 habilitado.

**Riscos mitigados:**

- Chaves/segredos em código fonte.
- Acesso excessivo de IAM.

---

### 5.6. Z9 “Lite” — Observabilidade & Auditoria

**Controles principais (lab):**

1. **CloudWatch Logs**
   - Habilitado para:
     - API Gateway (access logs),
     - Lambdas (ingest/curated).

2. **Métricas + Alarmes básicos**
   - Erros 4xx/5xx.
   - Falhas em Lambda.

3. **CloudTrail (management events)**
   - Trilhas de:
     - alterações de IAM,
     - criação/alteração de buckets,
     - deploy de Lambdas.

**Riscos mitigados:**

- Falta de trilha de auditoria básica (T10).
- Falta de visibilidade sobre falhas óbvias (T11).

---

## 6. Tabela Resumida — Ameaças x Controles do Lab

| ID Ameaça | Descrição Resumida                               | Zona(s)          | Controles do Lab                                                   |
|----------:|--------------------------------------------------|------------------|---------------------------------------------------------------------|
| T1        | Data poisoning / dados fora do contrato          | Z0, Z1           | Catálogo de fontes, JSON Schema, validação na Lambda               |
| T2        | Fonte enviando dado sensível indevido            | Z0, Z1, Z2       | Classificação em Z0, curadoria em Z3, buckets privados             |
| T3        | Abuso de endpoint / DoS lógico leve              | Z1               | API Key + Usage Plans (rate limit / quota)                         |
| T4        | Payload malformado / regras de negócio quebradas | Z1               | Validação de schema + regras na Lambda                             |
| T6        | Exposição de dados S3                            | Z2, Z3           | Block Public Access, IAM mínimo, SSE-S3, versionamento             |
| T7        | Perda / corrupção de dados                       | Z2, Z3           | Versionamento S3                                                   |
| T8        | PII vazando em camadas curadas                   | Z3               | Curadoria com mascaramento/anonimização                            |
| T10       | Falta de auditoria                               | Z8/Z9            | CloudWatch Logs, CloudTrail (management events)                    |
| T11       | Falta de detecção de falhas                      | Z8/Z9            | Métricas + alarmes básicos em CloudWatch                           |

> Para ameaças mais avançadas (ex.: ataques OWASP na borda, exfiltração sofisticada, detecção gerenciada, etc.), ver seção 7 e `architecture-prod.md`.

---

## 7. Limitações do Lab & Controles Recomendados para Produção

Apesar de ter um baseline “decente” para estudo e prática, o Lab 01 possui **limitações importantes** se comparado a um ambiente produtivo.

### 7.1. Limitações principais

- **Sem WAF** protegendo o endpoint de ingestão.
- **Sem GuardDuty / Security Hub / Macie / Config Rules**, ou seja:
  - não há detecção gerenciada de ameaças;
  - não há visão integrada de postura de segurança;
  - não há descoberta automática de PII em S3;
  - não há validação contínua de compliance de configuração.
- **Criptografia “básica” (SSE-S3)**:
  - sem CMKs dedicadas, sem controle fino de quem pode usar as chaves.
- **Observabilidade “lite”**:
  - sem centralização em Security Lake / SIEM;
  - sem playbooks automatizados de resposta a incidentes.

### 7.2. Controles recomendados para produção

Todos os controles abaixo estão descritos em detalhes em `architecture-prod.md`:

- **AWS WAF** na frente do API Gateway.
- **Amazon GuardDuty** para detecção gerenciada.
- **AWS Security Hub** como painel central de posture & findings.
- **Amazon Macie** para descoberta de dados sensíveis em S3.
- **AWS KMS (CMKs dedicadas)** para Raw/Curated.
- **AWS Config + Config Rules** para governança contínua.
- **Security Lake / SIEM** para correlação e resposta avançada.

---

## 8. Boas Práticas de Uso Seguro do Lab

Mesmo sendo um laboratório, algumas boas práticas são recomendadas:

1. **Não usar dados reais de clientes ou produção**
   - Sempre trabalhar com dados sintéticos ou anonimizados.

2. **Manter o repositório livre de segredos**
   - Não commitar chaves de API, tokens, credenciais em texto puro.
   - Usar SSM Parameter Store ou arquivos locais ignorados (`.gitignore`).

3. **Revisar permissões de IAM periodicamente**
   - Validar se as roles das Lambdas realmente têm apenas o necessário.

4. **Monitorar custos e logs**
   - Verificar se não há abuso de chamadas no API Gateway.
   - Conferir CloudWatch Logs em caso de comportamentos inesperados.

5. **Usar o lab como base para discussão de riscos**
   - Tratar o lab como um “esqueleto seguro mínimo”.
   - Discutir sempre: “o que faltaria aqui para virar Prod?” → `architecture-prod.md`.

---

## 9. Referências (Conceituais)

*(Sugestão de leituras, não obrigatório para rodar o lab)*

- OWASP Top 10 / OWASP API Security Top 10.
- NIST SP 800-53 (famílias AC, AU, SC, SI, etc.).
- AWS Well-Architected Framework — Security Pillar.
- AWS Security Reference Architecture (SRA).
- Documentação oficial:
  - Amazon S3 Security.
  - AWS Lambda Security.
  - Amazon API Gateway Security.

---

Este `SECURITY.md` deve ser lido em conjunto com o documento `architecture-lab.md` → visão técnica da arquitetura do lab.
