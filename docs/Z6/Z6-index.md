# Z6 — Model Serving & Inference

↩️ [Voltar ao README — Mapa Z0–Z9](../../README.md)

![Diagram](../../assets/Project_Diagram-Z6.drawio.svg)

> A Z6 expõe **modelos aprovados** (vindos da Z5) para **inferência on-line** (baixa latência) e **batch** (alta vazão) com **portas de entrada controladas**, **políticas de segurança na borda e no runtime**, e **trilhas de auditoria completas**.
> O estilo, profundidade e artefatos seguem o padrão dos documentos anteriores (ex.: Z5 — Registry & Governance). 

---

## 1) Papel da Z6 no MLOps Security Lab

**Objetivo central:** entregar inferência **confiável, reproduzível e governada**.
**Resultados-chave:**

* Servir **apenas** modelos **Assinados/Verificados** do **Prod Model Registry** (Z5).
* **Revalidar** assinatura/attestation/policies **no startup/refresh** do serviço.
* **Proteger a borda** com API GW/WAF/AuthN/AuthZ + **mTLS**, **Rate Limiting** e **Audit**.
* Em LLM/GenAI, **filtrar entradas/saídas** (prompt-injection, exfiltração, PII) via **LLM Security Gateway**.
* Em batch, gravar **somente** em **Scored Output Store** governada; integração com Z7 **via APIs com contrato**.

---

## 2) Componentes da Z6

> Cada item abaixo possui um arquivo de detalhamento sugerido (mesmo padrão da Z5).

### 2.1 Inference Gateway (API GW + WAF + AuthN/AuthZ, Rate Limiting, mTLS, Audit) — [Detalhamento](./Z6-2.1.md)

*Porta única* para inferência on-line de modelos clássicos.
TLS/mTLS, WAF para APIs, **rate limiting/quotas**, OIDC/OAuth2, **políticas por rota/versão/tenant**, headers de segurança e **auditoria estruturada** (request_id, identidade, modelo/versão, latência, resultado).

### 2.2 LLM/GenAI Security Gateway (Input/Output Filtering, PII Redaction, Anti Prompt-Injection & Exfiltração) — [Detalhamento](./Z6-2.2.md)

Todo consumo de LLM **passa antes** por este gateway:
**Input filtering** (inj. direta/indireta, limites, RAG com ACL por documento/tenant), **Output filtering/redaction** (PII/segredos), **tooling seguro** (allowlist, validação), **no-secrets-in-prompt**, observabilidade e **limites de custo**.

### 2.3 Online Model Serving (Pods/Microservices; mTLS; NetworkPolicies; Somente Modelos Assinados) — [Detalhamento](./Z6-2.3.md)

Serviços de inferência dos modelos clássicos com **verificação de assinatura/digest** no startup, containers **não-root**, FS **read-only**, **seccomp**, **resource limits**, **NetworkPolicies (deny-by-default)**, e **schema enforcement** de entrada/saída.

### 2.4 Feature Retrieval Proxy (Opcional) — [Detalhamento](./Z6-2.4.md)

Proxy controlado para *features on-line*: **cache TTL**, ABAC por tenant, mTLS, **políticas de fallback** (degradar com segurança > responder errado).

### 2.5 Batch Inference Orchestrator (Service Accounts; Versão Fixada; Saída Governada) — [Detalhamento](./Z6-2.5.md)

Orquestra lotes (Airflow/cronjobs) com **service accounts** próprias, **fixa versão** do modelo por execução (reprodutibilidade), lê **apenas** datasets autorizados (Z3) e escreve **exclusivamente** em **Scored Output Store**.

### 2.6 Scored Output Store (Governado) — [Detalhamento](./Z6-2.6.md)

Repositório de **predições/resultados** (on-line e batch) com **classificação**, **criptografia em repouso**, **retenção/expurgo** e **contrato de consumo** pelos sistemas de negócio (Z7).

### 2.7 Segredos, Chaves e Config (Vault/KMS) — [Detalhamento](./Z6-2.7.md)

Segredos via **Vault Agent/Sidecar** (tokens, credenciais, chaves de provedores LLM); **rotação** e escopo mínimo; chaves de **assinatura/verificação** em **KMS/HSM**.

### 2.8 Observabilidade & Audit (Integração com Z9) — [Detalhamento](./Z6-2.8.md)

**Logs estruturados** (sem PII), **métricas** (p95/p99, erros por rota, saturação, custos GenAI), **traces** ponta-a-ponta e **sinais de risco** (abuso de API, scraping, jailbreak).

### 2.9 Startup/Download Verification (Assinatura/Policy) — [Detalhamento](./Z6-2.9.md)

No **startup/refresh**, o serviço só carrega modelo se:
`stage ∈ {Approved, Staging}` ∧ **assinatura válida** ∧ **attestation presente** ∧ `policy_digest` esperado ∧ **não revogado** (checar CRL/transparency-log).

---

## 3) Riscos × Controles × Frameworks (Z6)

| Risco                                                   | Controles Z6                                                                                                         | Referências (exemplos)                             |
| ------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------- |
| **DoS/Abuso de API**                                    | 2.1 WAF + rate limit/quotas + circuit breakers; 2.8 métricas/alertas                                                 | OWASP API (API4), NIST SC-5/6, CSA IVS/MOS         |
| **Broken Auth/Access (tenant-escape, rota indevida)**   | 2.1 AuthN/AuthZ forte; 2.3 rotas/versões por escopo; 2.4 ABAC por tenant; 2.7 segredos rotacionados                  | OWASP API (API2/API5), NIST AC-*/IA-*, CSA IAM/AAC |
| **Modelo não aprovado/alterado (supply chain)**         | 2.3 só **modelos assinados**; 2.9 verificação em startup; 2.7 chaves em KMS/HSM; deploy canário/rollback             | NIST SI-7/CM-*, SLSA/SSDF (cadeia), CSA SEF        |
| **Prompt-Injection / Exfiltração (LLM)**                | 2.2 input/output filtering, PII redaction, RAG-ACL, tool allowlist, **no-secrets-in-prompt**                         | OWASP LLM/GenAI Top 10, CSA AICM, NIST AI RMF      |
| **Vazamento de PII/segredos em respostas**              | 2.2 output redaction; 2.6 store governado; 2.8 logs sem PII                                                          | NIST SC-28/AU-*, CSA DSI/LOG; LGPD                 |
| **Evasão/Adversarial Examples**                         | 2.1 validações de entrada + WAF; 2.3 schema enforcement; 2.8 monitorar anomalias; testes adversariais exigidos em Z5 | MITRE ATLAS, NIST AI RMF                           |
| **Model Extraction/Scraping (roubo de modelo via API)** | 2.1 throttling/quotas; respostas menos detalhadas; 2.8 detecção de padrão de consulta sistemática                    | OWASP API, MITRE ATLAS (model theft)               |
| **Bypass de governança (batch escreve direto no Core)** | 2.5 **escrita apenas em Scored Output**; integração com Z7 via **APIs com contrato**; 2.8 auditoria                  | NIST SC-7/CM-*, CSA GRC/IVS                        |
| **Logs/erros vazando dados sensíveis**                  | 2.8 **logs mínimos** de payload e **máscara de PII**; templates de erro padronizados                                 | OWASP Logging, NIST AU-*                           |

---

## 4) Teoria ↔ Prática (o que você constrói no Lab)

* **04-Serviços MLOps (Airflow/MLflow/FastAPI)**
  FastAPI como **serviço de inferência** (2.3) verificando **assinatura** do artefato do MLflow Registry; validação de schema; **Traefik/Nginx** como **Inference Gateway** (2.1) com OIDC/mTLS, WAF e Rate Limit.
* **05-OIDC & Segurança Avançada**
  Keycloak/IdP, **RBAC/ABAC**, perfis por aplicação/tenant, tokens de curta duração, *service-to-service* com mTLS.
* **03-Vault Agent & Integração**
  Segredos **dinâmicos** para provedores LLM/DB/Feature-store; rotação/escopo mínimo; chaves em **KMS/HSM**.
* **06-CI/CD & GitHub OIDC**
  Deploy **canário/blue-green**, verificação de **assinatura/attestation**, *version pin* de modelo, scan de imagem.
* **Z9**
  Dashboards de **latência/erros/custos** (GenAI), alertas de abuso e **trilhas por identidade/modelo/versão**.

---

## 5) Fluxos Resumidos

### 5.1 On-line (modelo clássico)

1. App → **Inference Gateway** (AuthN/AuthZ, WAF, rate limit).
2. Gateway → **serviço do modelo Vn** via **mTLS**.
3. Serviço valida **schema** → (opcional) **Feature Proxy** → predição.
4. Resposta padronizada (sem PII); **audit** + métricas → Z9.

### 5.2 GenAI/LLM

1. App → **LLM Security Gateway** (input filtering, limites).
2. Gateway → provedor/modelo (com **segredos via Vault** e restrições de região/política).
3. **Output filtering/redaction** → resposta → **audit** + custos.

### 5.3 Batch

1. Orquestrador fixa **modelo/versão** do registry.
2. Lê entradas autorizadas (Z3), processa lotes.
3. Escreve em **Scored Output Store**; **NÃO** publica direto no Core.
4. Z7 consome via **APIs com contrato**.

---

## 6) Políticas exemplares (Rego — trechos)

**6.1 Download/Startup do modelo (permit)**

```rego
package serving.startup
default permit = false
permit {
  input.stage == "Approved"  # (ou "Staging" sob regra)
  input.signatures.model_valid
  input.attestation.present
  input.revoke == false
  input.policy_digest == input.runtime.policy_digest
}
```

**6.2 Enforce de versão/rota por aplicação**

```rego
package serving.routes
default allow = false
allow {
  input.app in {"core-banking","risk-engine"}
  input.model.name == "churn_v2"
  input.model.version == "2.3.1"
  input.route == "/v2/predict"
}
```

**6.3 LLM Output Filter (redação simples de PII)**

```rego
package llm.output
default allow = true
deny_reason[msg] {
  some p in input.tokens
  p.class == "PII"
  msg := sprintf("PII token blocked: %v", [p.value])
}
```

---

## 7) Frases prontas para a entrevista

* “**Z6 só serve modelos do Registry (Z5)** e **revalida** assinatura/attestation/policies em cada **startup/refresh**; versão **revogada não carrega**.”
* “**Toda entrada passa por gateways**: API GW/WAF/AuthZ/mTLS; em GenAI, **LLM Security Gateway** faz **input/output filtering** e previne **prompt-injection/exfiltração**.”
* “**Batch não fura a governança**: escreve em **Scored Output** e integra com Core (Z7) só via **APIs com contrato**.”
* “Trato modelo em produção como **software crítico**: **supply chain verificado**, **runtime hardening**, **auditoria forte** e **rollback** claro.”
