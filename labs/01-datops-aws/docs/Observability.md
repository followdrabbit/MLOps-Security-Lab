# 👁️ Observability — DatOps em AWS (Z0–Z3 + Z8/Z9)

## 1. Objetivo

Este documento define o **modelo de observabilidade** da pipeline DatOps (Z0–Z3) da plataforma **MLOps Security Lab**, cobrindo:

- O **que** observar (eventos, logs, métricas, traces);
- **Como** coletar, armazenar e consultar essas informações;
- Diferenças entre a versão **Lab (baixo custo)** e a visão de **Produção (full)**;
- Como usar essa observabilidade para:
  - troubleshooting,
  - monitoramento de saúde,
  - auditoria e segurança (em conjunto com SECURITY.md / SECURITY-prod.md).

> Em resumo: este documento responde à pergunta  
> **“Como eu enxergo o que está acontecendo no meu DatOps?”**

---

## 2. Escopo

A observabilidade aqui descrita se aplica à pipeline DatOps:

- **Z0 — Fontes de Dados**
- **Z1 — Ingestion & Security Gateway** (API Gateway + Lambda Ingestion)
- **Z2 — Raw Data Lake (S3 Raw)**
- **Z3 — Curated Data & Data Products (S3 Curated + Glue + Athena)**
- Camadas transversais de:
  - **Z8 — Security & Trust Services**
  - **Z9 — Monitoring, Observability & Audit**

### 2.1. Documentos relacionados

- `architecture-lab.md` — Arquitetura do lab (baixo custo);
- `architecture-prod.md` — Arquitetura de produção (full);
- `SECURITY.md` — Controles de segurança do lab;
- `SECURITY-prod.md` — Controles de segurança em produção;
- `Lab-vs-Prod.md` — Comparação entre lab e produção.

---

## 3. Conceitos de Observabilidade Utilizados

Neste contexto, vamos tratar observabilidade em quatro eixos principais:

1. **Logs**  
   Eventos detalhados de execução (Lambda, API Gateway, WAF, etc.).

2. **Métricas**  
   Valores agregados ao longo do tempo (latência, erros, volume de registros, etc.).

3. **Traces (opcional)**  
   Rastreamento de ponta a ponta (útil principalmente em Produção).

4. **Eventos de Auditoria**  
   Alterações de configuração, acessos a dados, uso de chaves, etc.

> No Lab, focamos em **logs + métricas básicas + auditoria mínima**.  
> Em Produção, expandimos para **traces, auditoria completa, Security Lake/SIEM, dashboards e playbooks de resposta**.

---

## 4. Observabilidade no Lab (Versão “Lite”)

### 4.1. O que é observado no Lab

No Lab, o objetivo é **ver o suficiente para aprender e debugar**, sem estourar custos:

- Logs de:
  - API Gateway (access logs);
  - Lambda de ingestão (`ingestion_lambda`);
  - Lambda de curadoria (`curated_lambda`).
- Métricas de:
  - Erros 4xx/5xx do API Gateway;
  - Invocações e erros das Lambdas;
  - Tempo de execução básico.
- Auditoria:
  - **CloudTrail (Management Events)** para mudanças de IAM, S3, Lambda, etc.

### 4.2. Logs — Lab

**Serviço principal:** Amazon CloudWatch Logs

- **Grupos de logs (exemplos):**
  - `/aws/lambda/mlops-datops-ingestion-lab`
  - `/aws/lambda/mlops-datops-curated-lab`
  - `/aws/apigateway/mlops-datops-http-api-lab`

**Boas práticas no lab:**

- Logs de Lambda em **formato JSON estruturado**, contendo:
  - `timestamp`
  - `request_id`
  - `source_id`
  - `schema_version`
  - `result` (`accepted` / `quarantine`)
  - `records_count`
  - `error_type` (quando aplicável)
- Evitar logar:
  - payloads completos com dados sensíveis,
  - segredos ou tokens.

---

### 4.3. Métricas & Alarmes — Lab

**Serviço principal:** Amazon CloudWatch Metrics & Alarms

Métricas mínimas recomendadas:

- **API Gateway**
  - `4XXError`, `5XXError`
  - `Count` (número de requisições)
  - `Latency` / `IntegrationLatency` (latência média/p95)

- **Lambda (ingest/curated)**
  - `Invocations`
  - `Errors`
  - `Duration` (p95/p99)
  - `Throttles` (se ocorrerem)

**Alarmes mínimos:**

- **Erro alto em ingestão**
  - Condição: `Errors` da `ingestion_lambda` acima de N em X minutos.
- **Erro alto em curadoria**
  - Condição: `Errors` da `curated_lambda` acima de N em X minutos.
- **Erro repetido no endpoint de ingestão**
  - Condição: taxa de `5XXError` do API Gateway acima de um limite.

A notificação pode ser algo simples, como:

- SNS Topic enviando e-mail para o autor do lab.

---

### 4.4. Auditoria — Lab

**Serviço principal:** AWS CloudTrail (Management Events)

No Lab:

- CloudTrail deve estar habilitado ao menos para:
  - `Write` e `Read` management events:
    - alterações de IAM;
    - criação/alteração de buckets;
    - deploy de Lambdas;
    - criação/alteração de API Gateway.

Objetivo:

- Permitir investigações básicas:
  - “Quem mudou tal policy?”
  - “Quando esse bucket foi criado/alterado?”
  - “Quem fez o deploy dessa Lambda?”

---

## 5. Observabilidade em Produção (Visão Full)

> A partir daqui, entramos na visão alvo de produção, descrita em `architecture-prod.md` e `SECURITY-prod.md`.

### 5.1. O que muda de Lab para Produção

Em Produção, além do que já existe no Lab, temos:

- Logs adicionais:
  - AWS WAF;
  - Amazon GuardDuty / Macie / Config / Security Hub (findings);
  - CloudTrail **Management + Data Events** (S3, etc.);
- Métricas adicionais:
  - blocos do WAF;
  - volume de findings de segurança;
  - métricas de ingestão por fonte/parceiro;
- Traces (opcional):
  - AWS X-Ray para rastrear fluxos complexos;
- Centralização:
  - **Amazon Security Lake** ou SIEM externo;
- Automação:
  - EventBridge + Lambda/Step Functions/SOAR para resposta a incidentes.

---

## 6. Design de Observabilidade por Zona (Produção)

### 6.1. Z0 — Fontes de Dados

- Observabilidade foca em:
  - **Quem está chamando o quê, quando e com que frequência**.
- Fontes de dados:
  - Logs de API Gateway/WAF — `source_id` e `api_key`/client identificando a fonte;
  - Métricas por fonte (requisições, erros, latência).

Possíveis dashboards:

- Tráfego por `source_id` ao longo do tempo.
- Taxa de erro por fonte.
- Distribuição de payloads inválidos (quarentena).

---

### 6.2. Z1 — Ingestion & Security Gateway

**Componentes observados:**

- AWS WAF
- API Gateway
- Lambda de Ingestão
- KMS (uso das chaves, indiretamente via CloudTrail)

**Logs:**

- **WAF Logs**:
  - Registra requisições bloqueadas, desafiadas ou permitidas com match em regras.
- **API Gateway**:
  - Access logs com:
    - `requestId`, `principalId`, `httpMethod`, `resourcePath`, `status`, `latency`.
- **Lambda**:
  - Logs estruturados dos eventos de ingestão.

**Métricas:**

- **WAF**:
  - requests bloqueados por regra / por IP / por geolocalização;
  - spikes de bloqueios (sinal de ataque).
- **API GW**:
  - tempo de resposta,
  - taxa de erro (4xx/5xx),
  - número de requisições por fonte.
- **Lambda**:
  - invocações, erros, duração, throttles.

**Alertas típicos:**

- aumento súbito de `5XXError` no endpoint de ingestão;
- aumento súbito de requisições bloqueadas pelo WAF;
- queda abrupta no volume de ingestão esperado.

---

### 6.3. Z2 — Raw & Z3 — Curated

**Componentes observados:**

- S3 Raw / Curated
- Lambda de Curadoria
- Glue / Athena
- KMS (uso das CMKs)
- Macie (findings)

**Logs & eventos:**

- **CloudTrail Data Events para S3**:
  - `GetObject`, `PutObject`, `DeleteObject` em buckets Raw/Curated;
- **CloudWatch Logs da Lambda de Curadoria**:
  - quantidade de registros processados, descartados, mascarados;
  - erros de leitura/escrita;
- **Macie Findings**:
  - PII detectada em locais não esperados;
  - buckets com alto risco de exposição.

**Métricas:**

- Volume de ingestão em Raw:
  - número de arquivos por dia,
  - tamanho total em bytes.
- Volume de dados curados em Z3:
  - quantidade de registros/partições;
- Taxa de erro de curadoria:
  - proporção de registros descartados.

**Alertas:**

- Queda no volume de dados em Z3 (problema de curadoria);
- Aumento anormal de registros em quarentena;
- Findings críticos em Macie (dados sensíveis em buckets errados).

---

### 6.4. Z8 — Security & Trust Services

**Componentes observados:**

- IAM
- Secrets Manager / SSM
- KMS
- GuardDuty
- Security Hub
- Config

**Logs & eventos:**

- **CloudTrail**:
  - alterações de IAM (policies, roles);
  - uso de CMKs (Encrypt/Decrypt/GenerateDataKey);
  - mudanças em Config Rules.
- **GuardDuty Findings**:
  - comportamento suspeito de IAM, instâncias, rede.
- **Security Hub**:
  - findings agregados (GuardDuty, Macie, Config, etc.).

**Visão de observabilidade:**

- Painel de postura de segurança:
  - quantos findings abertos por severidade;
  - quantos recursos não conformes (Config);
  - histórico de uso de chaves KMS.

---

### 6.5. Z9 — Monitoring, Observability & Audit

Aqui é onde tudo se encontra.

**Componentes:**

- CloudWatch Logs & Metrics
- CloudTrail
- Amazon Security Lake ou SIEM externo
- EventBridge + Lambda/Step Functions (ou SOAR)

**Fluxo típico:**

1. Logs (WAF, API GW, Lambda, S3, etc.) vão para:
   - CloudWatch Logs e/ou S3.
2. CloudTrail (Mgmt + Data) envia para:
   - S3 e Security Lake.
3. Findings de segurança (GuardDuty, Macie, Config, WAF, etc.) chegam ao:
   - Security Hub → Security Lake → SIEM.
4. Dashboards em:
   - CloudWatch Dashboards;
   - Console do Security Hub;
   - Ferramenta SIEM (Kibana, Grafana, Splunk, etc.).

**Automação:**

- EventBridge Rules escutando eventos de:
  - Security Hub (finding novo);
  - Config (recurso não conforme);
  - GuardDuty (detecção crítica).
- Ações:
  - acionar Lambda de remediação (ex.: fechar bucket público, revogar chave);
  - abrir ticket (Jira/ServiceNow);
  - notificar times (Slack/Teams/Email).

---

## 7. Tabela Resumo — Observabilidade Lab vs Prod

| Dimensão        | Lab                                                     | Prod                                                                                  |
|-----------------|---------------------------------------------------------|---------------------------------------------------------------------------------------|
| Logs            | Lambda + API GW em CloudWatch                           | + WAF, + GuardDuty/Macie/Config, + logs enviados a Security Lake/SIEM                |
| Métricas        | Básicas (invocations, errors, 4xx/5xx)                  | Métricas detalhadas, p95/p99, taxas por fonte, blocos WAF, volume de findings        |
| Traces          | Não utilizado (opcional)                                | Opcional com X-Ray para fluxos complexos                                             |
| Auditoria       | CloudTrail Management Events                            | CloudTrail Mgmt + Data Events (S3, Lambdas, etc.)                                    |
| Centralização   | Não há (consulta direto em CloudWatch/CloudTrail)       | Security Lake / SIEM agregando logs + findings                                       |
| Automação IR    | Não há (análise manual)                                 | EventBridge + Lambda/Step Functions/SOAR executando playbooks de resposta            |
| Dashboards      | CloudWatch simples (manual)                             | Dashboards de negócios, operação e segurança (CloudWatch + SIEM + Security Hub)      |

---

## 8. Boas Práticas de Observabilidade

1. **Logar contexto, não segredos**
   - IDs, estados, métricas do fluxo.
   - Nunca tokens, senhas, PII em claro.

2. **Padronizar formato de logs**
   - JSON estruturado com campos consistentes:
     - `trace_id`, `request_id`, `source_id`, `dataset`, `result`, etc.

3. **Ligar métricas a SLOs**
   - Ex.: SLO de “Taxa de erro de ingestão < X% por dia”.
   - Ex.: SLO de “Latência p95 < Y ms”.

4. **Criar dashboards pairando dev + ops + security**
   - Não é só dashboard “bonito”, mas útil:
     - para squads;
     - para time de segurança;
     - para time de dados.

5. **Usar o Lab como playground de observabilidade**
   - Testar formatos de log;
   - Testar métricas e alarmes;
   - Testar consultas Athena sobre CloudTrail/CloudWatch exportados.

---

## 9. Próximos Passos

- No **Lab**:
  - Garantir que as Lambdas gerem logs estruturados;
  - Habilitar access logs de API Gateway;
  - Criar pelo menos 2–3 alarmes críticos.

- Na **visão de Produção**:
  - Definir conjunto mínimo de **dashboards**:
    - saúde da ingestão;
    - saúde da curadoria;
    - postura de segurança;
  - Escolher a estratégia de **centralização de logs** (Security Lake vs SIEM externo);
  - Documentar **playbooks de resposta**:
    - anexo a `RUNBOOK-IR.md` ou documento similar.

---

Este `Observability.md` deve ser lido em conjunto com:

- `architecture-lab.md` e `SECURITY.md` (para entender a base do lab);
- `architecture-prod.md` e `SECURITY-prod.md` (para entender o estado alvo em produção);
- `Lab-vs-Prod.md` (para enxergar exatamente o que falta evoluir entre um e outro.
