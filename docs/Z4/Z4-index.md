# Z4 — Model Factory (Dev/Train & Security by Design)

↩️ [Voltar ao README — Mapa Z0–Z9](../../README.md)

![Diagram](../../assets/Project_Diagram-Z4.drawio.svg)

> A Z4 é a **fábrica de modelos**: onde cientistas e engenheiros treinam, validam e **endurecem** modelos antes que qualquer artefato possa ser registrado e promovido. Mantém alinhamento total com a Z3 (dados curados & features) e entrega apenas **artefatos assinados** para a Z5. Estrutura, estilo e profundidade seguem o padrão dos documentos Z0–Z3. 

---

## 1. Papel da Z4 no MLOps Security Lab

A Z4 orquestra do **experimento ao candidato a produção**, impondo **controles de segurança, privacidade, reprodutibilidade e avaliação adversarial**.
Ela garante que **nenhum modelo** chegue à Z5 (Registry & Governance) sem:

* **Proveniência completa** (datasets/feature groups, versões de código, ambiente),
* **Higiene de dados/labels** e **testes de segurança** (poisoning, evasão, inferência),
* **Assinatura e SBOM** do artefato (container/wheel/onnx/pt/safetensors),
* **Model Card** e **evidências de avaliação** (métricas, fairness, riscos, salvaguardas),
* **Gates de aprovação** (políticas, compliance, LGPD, ética/responsible AI).

**Ideia central:** *Z4 é uma linha de montagem com “checkpoints de risco”*. Só sai do outro lado o que é **explicável, testado e íntegro**.

---

## 2. Componentes da Z4

Cada item tem papel funcional e controles explícitos.

### 2.1 Secure ML Workbench (Dev Environment Isolado) — [Detalhamento](./Z4-2.1.md)

Ambiente de desenvolvimento/experimentação com:

* **Isolamento** (namespaces, VMs/containers, quotas de CPU/GPU, egress control),
* **Acesso de leitura** somente a **Z3** (Curated/Feature Store) conforme RBAC/ABAC,
* **Segredos via Vault Agent**; nada de credencial fixa,
* **Hardening**: imagens base corporativas, patches, desabilitar execuções privilegiadas.

**Objetivo:** reduzir risco de **exfiltração**, **shadow data** e **dependências maliciosas**.

---

### 2.2 Dataset & Feature Access Broker (via Z3) — [Detalhamento](./Z4-2.2.md)

* Leitura **apenas** de datasets/feature groups **oficiais** (Z3),
* **Pseudonimização/mascaramento** por padrão quando viável,
* **Logs de acesso** (quem, o quê, quando, motivo) enviados à Z9,
* **Policies** por domínio/propósito (LGPD — *purpose limitation*).

**Objetivo:** prevenir **leakage** e uso indevido de PII em treino.

---

### 2.3 Training Pipelines (CI/CD) — Segurança & Reprodutibilidade — [Detalhamento](./Z4-2.3.md)

Pipelines versionados (Git + CI):

* **Pins & Lockfiles** (pip/conda/poetry), **SCA** (dependências), **SAST** (código),
* **SBOM** gerado no build (Syft/tern),
* **Imagens de container assinadas** (cosign/SLSA, keyless via OIDC),
* Persistência de **seeds, hyperparams, código, ambiente**, **hashes de dataset/feature** (MLflow/DVC/artefatos).

**Gates de build/test:** só prossegue se scanners e testes passarem.

---

### 2.4 Data & Label Hygiene (Pré-treino) — [Detalhamento](./Z4-2.4.md)

Checagens automáticas:

* **Deduplicação**, valores impossíveis, *target leakage* (temporal/causal),
* Distribuições vs. referência (PSI/KL), *holdout leak*,
* **Consentimento/legitimidade** para atributos sensíveis (LGPD).

**Se falhar:** bloqueio do job + evidência no tracking.

---

### 2.5 Adversarial & Security Testing (antes do registro!) — [Detalhamento](./Z4-2.5.md)

Conjunto de testes que **todo modelo candidato** deve passar:

* **Poisoning & Backdoor checks** (ex.: padrões “gatilho”, clusters suspeitos),
* **Evasion/Robustness** (perturbações pequenas → quanto a acurácia degrada),
* **Privacy attacks**: *membership inference*, *model inversion* (limiares aceitáveis),
* **Output integrity**: checagens de sanidade/limites do escore,
* **LLM-specific** (se aplicável): *prompt injection*, *jailbreak*, *data exfil*, *unsafe content*.

**Resultado:** relatório padronizado (anexado ao Model Card).
**Princípio:** *“Sem teste adversarial, sem registro.”*

---

### 2.6 Experiment Tracking & Proveniência — [Detalhamento](./Z4-2.6.md)

* **MLflow Tracking/Equivalente**: parâmetros, métricas, artefatos, datasets/feature refs,
* **Lineage** ligando **Z3 → Z4 → Z5** (run_id, git_sha, dataset_hash),
* **Reprodutibilidade**: *one-click* para refazer o treino com mesma versão do ambiente.

---

### 2.7 Model Packaging & Artifact Signing — [Detalhamento](./Z4-2.7.md)

* Empacotamento (wheel, onnx, torch, TF SavedModel, tokenizer/vocabulário),
* **Assinatura** do artefato e do container (cosign),
* **Verificação** obrigatória na Z5/Z6.

**Objetivo:** garantir **integridade e origem** (supply chain).

---

### 2.8 Model Evaluation, Bias & Responsible AI — [Detalhamento](./Z4-2.8.md)

* Métricas principais + **métricas por grupo** (fairness),
* **Model Card**: finalidade, dados, limitações, riscos, salvaguardas, *intended use*,
* **Checklist** de ética/privacidade (LGPD, vieses conhecidos, mitigação),
* **Evidências** anexadas ao artefato.

---

### 2.9 Secrets, Keys & Credentials — [Detalhamento](./Z4-2.9.md)

* **Vault/KMS** para chaves (assinatura, criptografia, HMAC),
* **AppRole/OIDC/JWT** para jobs; **nada** de segredos em env estático,
* **Rotação** e **escopo mínimo** por pipeline.

---

### 2.10 Compute Policy & Network Guardrails — [Detalhamento](./Z4-2.10.md)

* **Quotas** de GPU/CPU/memória, **limites de custo**,
* **NetworkPolicies**: sem acesso direto a Z2/externo sem autorização,
* **mTLS** interno, egress via proxy com allowlist.

---

### 2.11 Observabilidade do Treino — [Detalhamento](./Z4-2.11.md)

* Logs estruturados e métricas (tempo, custo, iterações, falhas),
* Alertas de **instabilidade numérica**, *NaN*, saturação de GPU,
* Exportação para Z9 e vínculo com run_id/artefatos.

---

## 3. Riscos × Controles × Frameworks (Z4)

| Risco (exemplos)                                     | Controles Z4                                                                 | Frameworks/Referências                                             |
| ---------------------------------------------------- | ---------------------------------------------------------------------------- | ------------------------------------------------------------------ |
| **ML Supply Chain** (lib maliciosa, imagem alterada) | 2.3 (SCA, SAST, SBOM), 2.7 (assinatura), 2.10 (hardening)                    | OWASP ML Top 10 (ML06), NIST SP 800-53 SA-*, NIST AI RMF, CSA AICM |
| **Poisoning/Backdoors**                              | 2.4 (higiene), 2.5 (adversarial), 2.6 (proveniência)                         | OWASP ML (ML02/ML10)                                               |
| **Membership/Model Inversion**                       | 2.5 (privacy attacks), 2.2 (pseudonimização Z3), 2.8 (limites/Model Card)    | OWASP ML (ML03/ML04), LGPD                                         |
| **Leakage/Target Leakage**                           | 2.4 (temporal/causal checks), 2.6 (lineage), 2.8 (documentação)              | NIST AI RMF (Measure/Manage)                                       |
| **Uso indevido de PII**                              | 2.2 (ABAC + purpose), 2.8 (Model Card), 2.11 (auditoria)                     | LGPD/GDPR, CSA CCM (DSI/IAM)                                       |
| **Skew treino→produção**                             | 2.6 (artefatos/versionamento), integração Z3-2.6/ Z6, testes de consistência | OWASP ML (ML08), NIST AI RMF                                       |
| **Exfiltração/Shadow data**                          | 2.1/2.2 (isolamento & broker), 2.10 (rede), 2.11 (logs)                      | NIST SP 800-53 AC/SC, OWASP API                                    |

---

## 4. Teoria ↔ Prática (o que você constrói no Lab)

* **04-Serviços-MLOps (Airflow/MLflow/FastAPI)**

  * DAGs de treino lendo **Z3** (Curated/Feature Store) com **Vault Agent**,
  * Tracking de runs + artefatos,
  * **Endpoint interno** para testes adversariais automatizados.

* **06-CI/CD & GitHub Actions + OIDC**

  * Build com **SBOM**, **scanners** e **assinatura de imagem/artefato**,
  * **Gates**: só “PASS” promove para `candidate/`.

* **Integração com Z5 (Registry & Governance)**

  * *Hook* pós-teste: se **2.5** aprovado → **assina** e **submete** ao Registry,
  * Anexa **Model Card**, relatórios e evidências.

* **Integração com Z8/Z9**

  * Policies (ABAC, roles por domínio),
  * Logs/métricas de treino e testes adversariais no SIEM/monitoramento.

---

## 5. Fluxo Resumido (Z3 → Z4 → Z5)

1. **Ler dados/features oficiais** (Z3) via broker controlado.
2. **Higiene & checagens** (2.4) → bloqueia leakage/poisoning básico.
3. **Treinar** com ambiente fixado (2.3) e **track completo** (2.6).
4. **Testar adversarial/privacy** (2.5).
5. **Empacotar & assinar** (2.7) + **Model Card** (2.8).
6. **Aprovação** (gates de política/compliance) → **submeter à Z5**.
7. **Sem aprovação**, **nada** vai para Serving (Z6).

---

## 6. Alinhamento com Frameworks

* **OWASP ML Security Top 10**: ML02, ML03, ML04, ML06, ML08, ML10.
* **OWASP GenAI/LLM Top 10** (se LLM): prompt injection, data exfil, plugin/supply chain.
* **NIST SP 800-53 Rev.5**: AC, AU, CM, SA, SC, SI aplicados ao ciclo de treino e artefatos.
* **NIST AI RMF**: *Govern → Map → Measure → Manage* (testes, riscos, documentação).
* **CSA CCM / AI Controls Matrix (AICM)**: AIS, DSI, IAM, LOG, SEF.
* **LGPD/GDPR**: minimização, base legal, documentação de finalidade (Model Card).

---

## 7. Frases prontas para a entrevista

* “**Na Z4, nenhum modelo é registrado sem teste adversarial e assinatura do artefato.** A fábrica registra proveniência (dados, código, ambiente), executa scanners/SBOM, checa leakage e ataques de privacidade, e só então libera para a Z5 com Model Card e evidências.”
* “**Treino só lê Z3 oficial via broker com ABAC.** Isso impede shadow pipelines, reduz risco de PII indevida e facilita auditoria.”
* “**Supply chain de ML é tratada como software crítico**: imagens e pacotes assinados, dependências auditadas, CI com gates.”
