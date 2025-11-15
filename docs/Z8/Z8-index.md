# Z8 — Security & Trust Services

*(IAM/IdP/PAM • Vault/Segredos • KMS/HSM/PKI • Policy-as-Code • DLP/Tokenização • SIEM/UEBA/SOAR • Catálogo & Classificação • Compliance & Evidências)*

↩️ [Voltar ao README — Mapa Z0–Z9](../../README.md)

![Diagram](../../assets/Project_Diagram-Z8.drawio.svg)

> A Z8 provê **serviços de segurança compartilhados** que sustentam todas as zonas (Z1–Z7): identidade, chaves, segredos, políticas, proteção de dados, monitoração e governança. O formato, linguagem e profundidade seguem o padrão das páginas Z6/Z7.

---

## 1) Papel da Z8 no MLOps Security Lab

**Objetivo central:** oferecer **controles transversais** (identity, keys, secrets, policy, data protection, logging) que **fortalecem a borda e o runtime** em todo o pipeline (ingestão → serving), viabilizando **mínimo privilégio, rastreabilidade e conformidade**.
**Resultados-chave:**

* Identidades **humanas e de workload** federadas (OIDC/mTLS), **RBAC/ABAC** e **PAM** para acessos privilegiados.
* Segredos **não ficam no código**: são entregues em runtime via **Vault Agent/sidecar**, com **rotação**.
* Chaves criptográficas **geridas em KMS/HSM**, com **envelope encryption** e **políticas de uso**.
* **Políticas como código** (OPA/Rego) para gates (Z5) e autorização em APIs (Z1/Z6/Z7).
* **DLP/Tokenização/Mascaramento** para PII; contratos de **finalidade** (LGPD) visíveis no catálogo.
* **SIEM/UEBA/SOAR** centralizam logs/telemetria (→ **Z9**) com **trilhas de auditoria** e resposta coordenada.

---

## 2) Componentes da Z8

> Cada item pode ter um arquivo “**Z8-2.x.md**” de detalhamento, seguindo o mesmo padrão dos componentes em Z6/Z7.

### 2.1 Identity, Access & PAM (IdP/OIDC/IAM) — [Detalhamento](./Z8-2.1.md)

* **IdP (Keycloak/ADFS/…):** OIDC/OAuth2/SAML para humanos; **OIDC federation** para *workloads* (ex.: GitHub Actions → Vault).
* **IAM (RBAC/ABAC):** papéis e *claims* por tenant/rota/modelo/ambiente.
* **PAM/JIT:** acesso administrativo **just-in-time**, **session recording**, **break-glass** auditado.
* **Mínimo privilégio:** *scopes* finos para Z1/Z6/Z7 (autorização por rota/versão/tenant, como nos contratos de Z7).

### 2.2 Secrets Manager (Vault) — [Detalhamento](./Z8-2.2.md)

* **Entrega em runtime** via **Vault Agent/sidecar**; **nada de credencial hardcoded**.
* **AppRole/JWT/OIDC** para *workloads*; *leases* curtos, **rotação** e **revogação**.
* **Transit encryption** para *envelope* (dados selados no app, chaves no KMS/HSM).
* **Auditoria** dos acessos a segredos (eventos encaminhados ao **Z9**).

### 2.3 Key Management, KMS/HSM & PKI — [Detalhamento](./Z8-2.3.md)

* **KMS/HSM:** geração/armazenamento seguro; **SC-12/SC-13** (chaves e canais) na prática.
* **Policies de key-usage:** quem pode **criar/usar/assinar** (ex.: chaves de assinatura de modelos Z5/Z6).
* **PKI/TLS/mTLS:** *issuing* de certificados para serviços (Z1/Z6); **rotação** e CRL.
* **Assinaturas e *attestations***: base para **cosign/in-toto** do registry (Z5) e *startup verification* (Z6).

### 2.4 Policy-as-Code & Authorization (OPA/Rego) — [Detalhamento](./Z8-2.4.md)

* **Gates de promoção (Z5):** *accept/reject* com critérios de segurança/compliance.
* **Autorização em APIs (Z6/Z7):** **ABAC por rota/tenant**; *deny by default*; *feature flags*; *deprecation windows*.
* **Catálogo de políticas** versionado (GitOps), *tests* e **evidence links** (auditoria de decisões).

### 2.5 Data Protection (DLP, Masking, Tokenização/Pseudonimização) — [Detalhamento](./Z8-2.5.md)

* **Discovery & classificação:** catálogo marca PII/sensibilidade/finalidade (LGPD).
* **DLP:** **egress controls**, *regex/smart classifiers*, prevenção de **logs sensíveis**.
* **Tokenização/mascaramento:** substitui identificadores; **purpose-based access** (só o necessário).

### 2.6 SIEM/UEBA/SOAR (Detecção & Resposta) — [Detalhamento](./Z8-2.6.md)

* **Ingestão de logs/métricas/traces** padronizados de Z1, Z6 e Z7 (→ **Z9**).
* **Detecção:** abuso de API, scraping, *prompt-injection* detectada, *drift* suspeito, uso fora de propósito.
* **Resposta:** *playbooks* (SOAR), *tickets* e **bloqueios automatizados** (WAF, IAM, chaves).

### 2.7 Data Catalog, Classification & Lineage — [Detalhamento](./Z8-2.7.md)

* **Domínio/tenant/finalidade** por dataset/campo; **quem pode usar e para quê**.
* **Lineage:** Z0→Z1→Z2→Z3→Z4→Z5→Z6/Z7 com *run_id*, *policy_digest*, *approvers*.
* **Contratos de leitura** (OpenAPI versionado) para Z7 consumir via APIs (**não** direto do bucket).

### 2.8 Compliance, Evidence & Audit Trails — [Detalhamento](./Z8-2.8.md)

* **Trilhas de auditoria** fim-a-fim (quem, o quê, quando, por quê).
* **Evidence store** (artefatos de build, SBOM, *attestations*, *approvals*).
* **Access reviews** e reconciliação de propósito (LGPD), com *evidence links*.

---

## 3) Riscos × Controles × Frameworks (Z8)

| Risco transversal                           | Controles em Z8                                                                            | Referências (exemplos)                  |
| ------------------------------------------- | ------------------------------------------------------------------------------------------ | --------------------------------------- |
| **Segredo exposto / credenciais hardcoded** | 2.2 Vault Agent, *leases* curtos, rotação, OIDC federation (GH→Vault)                      | NIST IA-5/AC-6; OWASP ASVS Secrets; CSA |
| **Uso indevido de chaves/assinaturas**      | 2.3 KMS/HSM, *key-usage policies*, CRL/rotação, assinatura/attestation verificadas (Z5/Z6) | NIST SC-12/SC-13; SSDF/SLSA; CSA EKM    |
| **Autorização fraca / *tenant-escape***     | 2.1 IAM com RBAC/ABAC; 2.4 OPA/Rego (*deny-by-default*; *claims*-based)                    | OWASP API (API1/API5); NIST AC-*/IA-*   |
| **Vazamento de PII por logs/saída**         | 2.5 DLP/masking/tokenização; *logging minimalista*; *redaction*                            | NIST SC-28/AU-*; LGPD; CSA DSI/LOG      |
| **Shadow access / bypass de gateway**       | 2.1 controles de rede + IAM; 2.6 SIEM/UEBA + SOAR bloqueando; IaC com *guardrails*         | NIST SC-7/CM-*; CSA IVS/SEF             |
| **Não conformidade / falta de evidências**  | 2.8 *Evidence store* + *audit trails*; *access reviews* periódicos                         | NIST AU-*; CSA GRC/STA                  |

> Observação: mantemos a mesma abordagem “Riscos × Controles × Frameworks” usada em Z6/Z7 para consistência de leitura.

---

## 4) Teoria ↔ Prática (o que você constrói no Lab)

* **01/03 – Vault & Vault Agent:** montar **AppRole/JWT/OIDC**, *templates* `.ctmpl`, *leases* e auditoria.
* **05 – OIDC & Segurança Avançada:** **IdP**, *workload identity*, RBAC/ABAC por rotas/tenants.
* **06 – CI/CD Seguro:** SBOM, assinaturas/attestations (**cosign/in-toto**), *policy checks* (OPA), segredos via OIDC→Vault.
* **02 – Core Infra:** PKI/mTLS, redes internas, *egress controls* para DLP.
* **Integração com Z5/Z6/Z7:** verificação de assinaturas (Z5), *startup verification* (Z6) e *access reviews* de consumo (Z7).

---

## 5) Fluxos Resumidos

### 5.1 Entrega de segredos (workload)

1. *Workload* autentica via **OIDC** no Vault → recebe *token* de curta duração.
2. **Vault Agent** injeta credenciais temporárias para chamar Z6/Z7.
3. Acesso auditado em Z8 → eventos no **Z9**.

### 5.2 Criptografia com envelope (KMS/HSM + Transit)

1. App gera chave de dados efêmera e **sela** via **Transit/KMS**.
2. Conteúdo trafega/repousa cifrado; chaves de envelope nunca saem do KMS/HSM.
3. Políticas de uso e rotação aplicadas.

### 5.3 Autorização por política (OPA/Rego)

1. API (Z6) chama **Policy Engine** com *input* (rota, tenant, *claims*, versão).
2. **Rego** decide *allow/deny*; logs → **SIEM**.

### 5.4 DLP/masking na saída

1. Serviço identifica PII → **masking/redaction/tokenização** antes de responder.
2. Logs **sem payload sensível**; métricas/traços → **Z9**.

---

## 6) Políticas exemplares (Rego — trechos)

**6.1 Uso de chave de assinatura (permit restrito)**

```rego
package z8.kms
default allow_sign = false
allow_sign {
  input.key_purpose == "model-sign"
  input.claims.role == "ml-sec"
  time.now_ns() - input.claims.session_start < 3600000000000 # 1h JIT
}
```

**6.2 ABAC por rota/tenant (API de inferência)**

```rego
package z8.authz
default allow = false
allow {
  input.route == "/api/churn/v2/predict"
  input.claims.tenant == input.request.tenant
  input.claims.scope[_] == "inference:churn:v2:invoke"
}
```

**6.3 DLP simples para respostas**

```rego
package z8.dlp
deny[msg] {
  some f in input.response_fields
  f.class == "PII"
  msg := sprintf("PII blocked in output: %v", [f.name])
}
```

---

## 7) Checklists operacionais

**Antes do go-live**

* [ ] **Federation OIDC** para *workloads*; **Vault Agent** em todos os serviços.
* [ ] **KMS/HSM** com *key policies* claras; CRL/rotação configuradas.
* [ ] **OPA/Rego** versionado com **tests** e *rollback*.
* [ ] **DLP/masking/tokenização** ativos nas rotas sensíveis.
* [ ] **SIEM/UEBA/SOAR** recebendo logs/traços; **painéis** e **alertas** prontos.
* [ ] Catálogo com **classificação/finalidade** por dataset/campo.

**Durante operação**

* [ ] **Rotação** de segredos/chaves periódica; *access reviews* mensais.
* [ ] *Drift* de políticas (o que está sendo permitido/negado) monitorado.
* [ ] **Auditoria** de assinaturas/attestations (Z5) e *startup checks* (Z6) contínua.
* [ ] Testes de **IR/SOAR** e *table-top* de vazamento de segredos/PII.

---

## 8) Frases prontas para a entrevista (Z8)

* “**Segredos não vivem no código**: entrego via **Vault Agent** com OIDC e rotação curta; cada serviço só recebe o que precisa.”
* “**Chaves e políticas** ficam no **KMS/HSM**; uso **assinaturas e attestations** para ligar Z5 (registry) e Z6 (*startup verification*).”
* “**Autorização é *policy-as-code***: OPA/Rego com **deny-by-default** e **ABAC por tenant/rota/versão**.”
* “**DLP/mascaramento/tokenização** garantem *privacy-by-default*; logs sem PII e *evidence store* para auditoria.”
* “**SIEM/UEBA/SOAR** fecha o ciclo com detecção e resposta — e tudo cai em **Z9** para observabilidade e auditoria centralizadas.”
