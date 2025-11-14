# Z7 — Consumers & Business Apps

*(Core/Risk/Fraud • Canais Internos • Contratos de API • Uso Responsável de Modelos • Privacidade por Padrão)*

![Diagram](../../assets/Project_Diagram-Z7.drawio.svg)

> A Z7 reúne **sistemas de negócio** (core banking, risco, fraude) e **canais internos** (portais, apps, chatbots) que **consomem resultados de modelos** com **governança**, **privacidade** e **contratos estáveis**. O estilo, profundidade e artefatos seguem o padrão de Z6. 

---

## 1) Papel da Z7 no MLOps Security Lab

**Objetivo central:** habilitar **decisões de negócio** com IA/ML **sem exponor storage bruto**, respeitando **contratos de API**, **minimização de dados**, **LGPD** e **trilhas de auditoria**.

**Resultados-chave:**

* Consumir **inferência on-line** **apenas** via **Inference Gateway / LLM Security Gateway (Z6)**.
* Consumir **saídas batch/online** **apenas** pela **Scored Output Access API (Z6-2.7)** — **nunca** direto do bucket.
* Aplicar **políticas de uso** (escopo, propósito, janelas, thresholds) e **UI privacy-by-default**.
* Produzir **logs/métricas** de consumo (latência, erro, custo, cobertura) para **Z9**.

---

## 2) Componentes da Z7

> Cada item pode ter um arquivo “Z7-2.x” de detalhamento, no mesmo padrão dos documentos Z6.

### 2.1 Core Banking / Risk / Fraud Engines — [Detalhamento](./Z7-2.1.md)

* Sistemas transacionais e motores de decisão que fazem **chamadas síncronas** a **Z6-2.1** (modelos clássicos) ou **Z6-2.2** (LLM), e **assíncronas** à **Scored Output Access API (Z6-2.7)** para recuperar **scores historizados**.
* **Boas práticas:** AuthN/AuthZ via OIDC/mTLS; **contratos de rotas**; *circuit breakers*; **não cachear PII** no cliente.

### 2.2 Canais & Apps Internos (Portais, Back-office, APIs de Produtos) — [Detalhamento](./Z7-2.2.md)

* Camada de **experiência** para times internos e operações.
* **UI privacy-by-default:** mascarar identificadores; redigir campos sensíveis; exibir **apenas** atributos necessários à decisão.

### 2.3 Chatbots/Assistentes Internos (GenAI) — [Detalhamento](./Z7-2.3.md)

* **Sempre via LLM Security Gateway (Z6-2.2)** com **input/output filtering**, **tooling allowlist** e **sem segredos em prompts**.
* **RAG** com ACL por documento/tenant; **quotas de custo** e **limites de contexto**.

### 2.4 Decisioning & Orquestração (BPM/Regras) — [Detalhamento](./Z7-2.4.md)

* Combina **regras de negócio** com **scores** (cut-offs, rejeição manual, *hold*).
* **Explainability de produção** (quando permitido) para auditoria humana.

### 2.5 Client Adapter / SDK (Contrato de Consumo) — [Detalhamento](./Z7-2.51.md)

* Bibliotecas internas padronizando **Auth**, **retries**, **idempotência**, **masking** e **telemetria** ao chamar Z6.
* Evita **acoplamento frágil** entre dezenas de sistemas e os contratos de inferência.

### 2.6 Catálogo & Data Contracts (Leitura) — [Detalhamento](./Z7-2.6.md)

* Catálogo apontando **quais rotas e campos** cada consumidor pode ler; **versionamento** (OpenAPI v1/v2) e **depreciação com janela**.
* **Propósito declarado** (finalidade LGPD) por rota/consumidor.

### 2.7 Segredos & Chaves (Consumidores) — [Detalhamento](./Z7-2.7.md)

* **Sem credenciais hardcoded**; usar **Vault Agent** ou *workload identity* (mTLS).
* **Rotação** e **escopo mínimo** (apenas as rotas/tenants necessários).

### 2.8 Observabilidade & Auditoria (Consumo) — [Detalhamento](./Z7-2.8.md)

* **Métricas por rota/tenant**: p95/p99, taxa de erro, custo, *hit-ratio* de “/last”.
* **Logs estruturados** sem PII; **traces end-to-end** (Z9).
* **Access reviews** periódicos (quem lê, o que e por quê).

---

## 3) Riscos × Controles × Frameworks (Z7)

| Risco de Consumo                                      | Controles em Z7                                                                                     | Referências (exemplos)                        |
| ----------------------------------------------------- | --------------------------------------------------------------------------------------------------- | --------------------------------------------- |
| **Leitura direta do bucket/tabela**                   | Consumo **apenas via APIs Z6-2.7**; buckets sem permissão de leitura para apps; *egress controls*   | NIST SC-7/SC-39; CSA DSI; OWASP ASVS          |
| **Exposição de PII/segredos em UI ou logs**           | **Masking/Redaction** na API; UI privacy-by-default; logs sem payload sensível; minimização         | LGPD; NIST SC-28/AU-; CSA DSI/LOG             |
| **Uso fora do propósito/escopo (LGPD Purpose Creep)** | **Data contracts** e **propósito declarado**; ABAC por rota; auditoria e *access review*            | LGPD; NIST AI RMF (Govern); CSA GRC           |
| **Automação cega com decisões injustas/errôneas**     | **Decisioning com thresholds**, *human-in-the-loop* quando aplicável, **explainability** controlada | NIST AI RMF (Map/Measure/Manage); MITRE ATLAS |
| **Prompt-Injection/Exfiltração em chatbots**          | **LLM Security Gateway** (input/output filtering, tooling allowlist, no-secrets-in-prompt)          | OWASP LLM/GenAI Top 10; CSA AICM              |
| **Enumeração/cross-tenant por APIs de leitura**       | **ABAC por tenant**; *row-level filtering*; limites de janela/paginação; quotas                     | OWASP API (API1/API5); NIST AC-*/IA-*         |
| **Acoplamento frágil a versões de rota/schema**       | **OpenAPI versionado**, SDK interno, *feature flags*, depreciação guiada                            | NIST CM-; OWASP ASVS                          |
| **Uso de score inválido/obsoleto**                    | “/last” com **commit atômico** (Z6-2.6), cursores `since_ts`, SLO e alarmes                         | NIST SA-; CSA IVS                             |

---

## 4) Teoria ↔ Prática (o que você constrói no Lab)

* **Adapter/SDK Z7** (ex.: Python/Go) com:

  * OIDC/mTLS, **ABAC** (claims), **retries/backoff**, **idempotência**, **masking** no lado do cliente;
  * *helpers* para `GET /last` e `POST /search` (Z6-2.7).
* **Exemplo Core/Risk**:

  * Serviço que **decide** com base no score: cut-off + regras de negócio; escreve **reason codes** internos (sem PII).
* **Exemplo Chatbot Interno**:

  * Chama **LLM Security Gateway**; demonstra **input/output filtering** e **tool allowlist**.
* **Painéis Operacionais**:

  * p95/p99 por rota, taxa de erro, custo por consumidor, *score coverage* (quantas decisões usam o modelo correto).

---

## 5) Fluxos Resumidos

### 5.1 Decisão on-line (modelo clássico)

1. App/Core → **Inference Gateway (Z6-2.1)** com AuthN/AuthZ + rate limit.
2. **Serviço do modelo (Z6-2.3)** responde; App aplica **regras de negócio** e registra **audit**.
3. Opcional: persiste chave de correlação e decisão em store operacional.

### 5.2 Consulta operacional (“último score”)

1. Sistema chama **Scored Output Access API (Z6-2.7)** `GET /last`.
2. API aplica **masking + ABAC**; retorna **predição mais recente**.
3. Z7 não lê bucket diretamente; só via API.

### 5.3 Análise histórica / reconciliação

1. Time de risco chama `POST /search` (janela, modelo, tenant).
2. Paginação e agregações pré-definidas; **sem dados por entidade** quando não permitido.

### 5.4 Chatbot interno (LLM)

1. Operador → **LLM Security Gateway (Z6-2.2)** → provider/modelo → **Output filtering/redaction** → resposta.
2. **Custos/latência** monitorados por identidade e rota.

---

## 6) Políticas exemplares (Rego — trechos)

**6.1 Bloquear leitura direta**

```rego
package z7.read
default allow_bucket = false
# Somente APIs oficiais podem fornecer dados de score
```

**6.2 ABAC por rota/tenant**

```rego
package z7.api
default allow = false
allow {
  input.route == "/api/scored/v1/last"
  input.claims.tenant == input.query.tenant
  input.claims.scope[_] == "scored:last:read"
}
```

**6.3 UI privacy-by-default**

```rego
package z7.ui
mask[field] { field == "entity_key" }
mask[field] { field == "explanations"; input.claims.role != "risk-analyst" }
```

---

## 7) Checklists operacionais

**Antes do go-live**

* [ ] Sem **leitura direta** do storage — apenas **APIs Z6-2.7**.
* [ ] **Adapter/SDK** integrado (Auth, retries, idempotência, telemetria).
* [ ] **ABAC** e **masking** testados (positivo/negativo).
* [ ] **SLOs por rota** e **quotas** configurados por consumidor.
* [ ] **UI privacy-by-default** revisada; **mensagens de erro** sem dados sensíveis.

**Durante operação**

* [ ] Monitore p95/p99, taxa de erro, **custo por consumidor**.
* [ ] *Access reviews* regulares e reconciliação de **propósito/escopo** (LGPD).
* [ ] Alertas para **spikes** de consultas amplas e **enumeração**.
* [ ] Gestão de versões (OpenAPI/SDK) e depreciações com *changelog*.

---

## 8) Integrações

* **Z6 (Serving & Inference)**: porta única para inferência on-line e leitura governada de saídas (via **Z6-2.7**).
* **Z8 (IAM/Vault/KMS/DLP/Catálogo)**: identidade, segredos e classificação; **propósito** e **retentiva**.
* **Z9 (Monitoring & Audit)**: métricas, logs de acesso, *tracing* e reconciliações.

---

## 9) Frases prontas (entrevista)

* “Em Z7, **ninguém lê direto do bucket**: tudo passa pela **Scored Output Access API**, com **ABAC** e **masking**.”
* “Para **real-time**, chamamos o **Inference Gateway**; para histórico/operacional, usamos `/last` e `/search` com **contratos estáveis**.”
* “**UI privacy-by-default**, **propósito declarado** e **minimização** reduzem risco LGPD; **observabilidade por consumidor** controla custo e abuso.”
* “Chatbots internos passam pelo **LLM Security Gateway** para **input/output filtering** e **tooling seguro**; **sem segredos em prompts**.”
