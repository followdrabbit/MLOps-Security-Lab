# Z5 — Model Registry & Governance (Fonte Única de Verdade)

![Diagram](../../assets/Project_Diagram-Z5.drawio.svg)

> A Z5 transforma **modelos candidatos** vindos da Z4 em **versões governadas** para consumo seguro na Z6. É o **catálogo imutável** de artefatos assinados, políticas, evidências e decisões — onde acontecem **gates de aceite, promoção, revogação** e todo o vínculo com **Model Card, SCDR/ADR, SBOM, proveniência** e **mudança controlada**.
> A estrutura, estilo e profundidade seguem os documentos Z0–Z4; Z5 verifica as evidências, assinaturas e attestations produzidas na Z4 antes de aceitar qualquer versão. 

---

## 1) Papel da Z5 no MLOps Security Lab

**Objetivo central:** tornar o **Registry** a *única fonte de verdade* para modelos — **imutável**, **assinado**, **auditável** e com **governança** de ciclo de vida.
**Resultados-chave:**

* Rejeitar tudo que **não** tenha **proveniência**, **teste adversarial PASS**, **SBOM**, **assinatura** e **attestation**.
* Registrar **metadados de negócio/risco** (Model Card, SCDR/ADR).
* Controlar **estados** (`Candidate` → `Staging` → `Approved` → `Revoked`) e **promoções**.
* Servir políticas e referências que a Z6 **revalida** no *startup*/**download**.

---

## 2) Componentes da Z5

### 2.1 Registry Core (catálogo de modelos) — [Detalhamento](./Z5.2.1.md)

* Armazena **versões imutáveis** com `name:version` + `digest`.
* Mantém **tags/attrs** (run_id, dataset@snapshot, container_digest, sbom_ref, policy_digest, approvers).
* **No lab:** MLflow Model Registry (mas o desenho vale para qualquer registry corporativo).

### 2.2 Verificação de Assinaturas & Attestations — [Detalhamento](./Z5-2.2.md)

* **Entrada (upload)**: validar **cosign sig** (artefato/imagem), **in-toto/SLSA** (proveniência), **checksums** (manifest).
* **Saída (download/consumo)**: **reverificar** assinatura/attestation e checar **revogação** (CRL/rekor/estado).
* Bloqueia submissões **sem** evidências ou com **cadeia inválida**.

### 2.3 Policy Engine & Gates (Policy-as-Code) — [Detalhamento](./Z5-2.3.md)

* Políticas OPA/Rego determinam **accept/reject** e **promoção** (`Candidate→Staging→Approved`).
* Gates típicos exigem: `security_decision=PASS`, **SBOM sem CVEs bloqueadoras**, **Model Card completo**, **attestation** coerente, **fairness** e **SLO profile**.
* Integra **LGPD** (finalidade/base legal) via *tags* do catálogo de dados.

### 2.4 Metadados de Governança (Model Card, SCDR, ADR) — [Detalhamento](./Z5-2.4.md)

* **Model Card** final (finalidade, limites, riscos, salvaguardas, público-alvo).
* **SCDR/ADR** linkados ao `run_id` e à versão (decisões de segurança/arquitetura, risco residual, condicionantes).
* Tornam a promoção **auditável** e **explicável**.

### 2.5 Versionamento, Estados & Imutabilidade — [Detalhamento](./Z5-2.5.md)

* Versões são **imutáveis**; alterações ⇒ **nova versão**.
* Estados e transições:

  * `Candidate` (aceito no catálogo, sem servir)
  * `Staging` (teste integrado/produção-like)
  * `Approved` (liberado para Z6)
  * `Revoked` (bloqueado para carga/uso)
* Transições **somente via pipeline** com verificação de assinatura/políticas.

### 2.6 Acesso & Tenancy (IAM/ABAC) — [Detalhamento](./Z5-2.6.md)

* **RBAC/ABAC** para quem **lê**, **promove**, **revoga** e **assina**.
* Segrega **times/dominios/parceiros** (multi-tenant) e aplica **SoD** (segregação de funções).

### 2.7 Promoção & Change Management — [Detalhamento](./Z5-2.7.md)

* **Promotion gates** com *four-eyes* (Segurança + Produto) e *change ticket* (janela, rollback, impacto).
* Publica **release artifacts**: `release_manifest.json`, `policy_snapshot/`, `release_notes.md`, `dashboards_links.json`.

### 2.8 Revogação, Quarentena & Rollback — [Detalhamento](./Z5-2.8.md)

* `Revoked` invalida consumo pela Z6; **purga de cache** no serving é obrigatória.
* Quarentena para versões sob investigação; **runbook** de rollback para versão **Approved** anterior.

### 2.9 Integridade de Download (para Z6) — [Detalhamento](./Z5-2.9.md)

* Z6 só carrega `stage in {"Staging","Approved"}` **com**:
  **assinatura válida**, **attestation presente**, **policy_digest** correspondente, **não revogado**.
* Falha na verificação ⇒ **bloqueia startup**.

### 2.10 Integrações (Z6, Z8, Z9) — [Detalhamento](./Z5-2.10.md)

* **Z6**: *startup check* de assinatura/attestation/políticas.
* **Z8**: Vault/KMS (chaves/rotação), IAM (papéis), DLP (marcação de modelos sensíveis).
* **Z9**: eventos “release_published”, “promotion”, “revocation”, dashboards *per-version*.

---

## 3) Riscos × Controles × Frameworks (Z5)

| Risco                                            | Controles Z5                                                                                                  | Referências                                |
| ------------------------------------------------ | ------------------------------------------------------------------------------------------------------------- | ------------------------------------------ |
| **Supply chain** (artefato trocado/interceptado) | Verificação de **assinaturas** (cosign/KMS), **attestation** (SLSA/in-toto), **checksums** em upload/download | NIST SSDF, SLSA, NIST SP 800-53 SA-*, CM-* |
| **Promoção sem evidências**                      | Gates OPA: `security_decision=PASS`, SBOM sem CVEs bloqueadoras, Model Card/SCDR/ADR obrigatórios             | NIST AI RMF (Measure/Manage), OWASP ML/LLM |
| **Uso de versão revogada**                       | Estado `Revoked`, CRL/transparency log, verificação no **download** e *startup* do Z6                         | NIST SI-4, CSA AICM (SEF/LOG)              |
| **Governança opaca**                             | Model Card + SCDR/ADR linkados, release notes, *change ticket*                                                | ISO/IEC 23894, LGPD/GDPR                   |
| **Quebra de conformidade (LGPD/finalidade)**     | *Tags* de finalidade/base legal exigidas; OPA rejeita inconsistências                                         | LGPD Art. 6/7, CSA CCM (DSI/IAM)           |
| **Tenancy/abuso de privilégios**                 | RBAC/ABAC, SoD, auditoria por ação                                                                            | NIST AC-*, AU-*                            |

---

## 4) Teoria ↔ Prática (o que você constrói no Lab)

* **Registry**: MLflow Model Registry com **webhooks** para OPA/Rego (accept/promotion), **cosign verify**, **attestation check**.
* **Pipelines**: passos “`submit→candidate`” e “`promote→staging/approved`” com gates automáticos + *four-eyes*.
* **Release artifacts**: geração de `release_manifest.json`, `policy_snapshot/`, `dashboards_links.json` anexados à versão.
* **Revogação**: comando/pipeline “`revoke`” que marca a versão, limpa cache na Z6 e abre *incident/SCDR*.

---

## 5) Fluxo Resumido (Z4 → Z5 → Z6)

1. **Z4** envia pacote **assinado** + **attestation** + evidências → **submit** (vira `Candidate`). 
2. **OPA** valida políticas (**PASS**) → `Staging` → testes integrados.
3. **Promotion** com *four-eyes* → `Approved` + **release artifacts**.
4. **Z6** consome **somente** `Approved/Staging` **revalidando assinatura/attestation/policies**.
5. **Revogação** marca versão como `Revoked` e bloqueia consumo.

---

## 6) Políticas exemplares (Rego — trechos)

**6.1 Accept (entrada no catálogo)**

```rego
package registry.accept
default allow = false
allow {
  input.security_decision == "PASS"
  input.signatures.model_valid
  input.attestation.present
  not input.sbom.has_blocking_cves
  input.model_card.present
  input.dataset.snapshot_present
}
```

**6.2 Promotion (Staging → Approved)**

```rego
package registry.promote
default allow = false
allow {
  input.stage_from == "Staging"
  input.stage_to   == "Approved"
  input.canary.result == "PROCEED"
  input.policies.bound == true
  input.signatures.model_valid
  input.attestation.present
}
```

**6.3 Download (Z6 startup/refresh)**

```rego
package registry.download
default permit = false
permit {
  input.stage in {"Staging","Approved"}
  input.signatures.model_valid
  not input.revoke
  input.policy_digest == input.runtime.policy_digest
}
```

---

## 7) Frases prontas para a entrevista

* “**Z5 é o guardião de integridade e governança**: só entra o que vem **assinado**, com **attestation**, **SBOM**, **Model Card** e **gates PASS**. Promoção é **policy-as-code** + *four-eyes*; revogação bloqueia consumo e limpa caches do serving.”
* “Em produção, a Z6 **revalida** assinatura/attestation/políticas a cada **startup/download**. Se qualquer coisa não bate (**policy_digest**, CRL, estado `Revoked`), **o modelo não carrega**.”
* “Trato modelo como **software crítico**: **cadeia de suprimentos** verificada (SLSA/in-toto), **governança viva** (Model Card/SCDR/ADR) e **change management** com *rollback* claro.”
