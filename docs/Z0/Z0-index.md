# Z0 — Data Sources (External, Partner & Internal Producers)

![Diagram](../../assets/Project_Diagram-Z0.drawio.svg)

## 1. Papel da Z0 no MLOps Security Lab

**Z0 é o ponto de partida lógico de todos os dados** que podem vir a alimentar o pipeline de IA/ML/LLM.

No contexto do **MLOps Security Lab**:

* Representa **fontes externas, parceiras e internas**, antes de cruzarem qualquer controle do lab.
* É tratada como **zona de risco**, não como zona de confiança.
* Conecta diretamente com a linha do README:

> “Z0 — Fontes externas, parceiras e internas → Riscos: data poisoning, arquivos malformados → Controles: allowlist, schema/MIME, validação de origem.”

**Princípio chave do lab:**
Nenhum dado de Z0 é considerado confiável por default. A confiança só começa a ser construída em **Z1 (Ingestion & Security Gateway)**.

---

## 2. Componentes da Z0

Dividimos Z0 em dois grandes grupos, ambos vistos sob a ótica **zero trust / risk-based**.

### 2.1. External & Partner Sources (Untrusted)

Usados para simular o mundo real no lab:

* **Clientes / Apps Externos**

  * Requisições HTTP / APIs públicas simuladas.
  * Uploads de arquivos (ex.: PDF, CSV) e eventos de uso.
* **Parceiros / Bureaus / Open Finance**

  * Feeds simulados de bureau, open finance, listas de sanções, etc.
  * Tratados como *terceiros* no modelo de risco.
* **Fontes Públicas / OSINT / Open Data**

  * Datasets públicos (governo, mercado, etc.) usados como mock.
  * Potencialmente ruidosos, desatualizados ou manipuláveis.

**No lab:** representados por scripts, arquivos de exemplo, requisições geradas artificialmente — mas sempre documentados como **não confiáveis**.

---

### 2.2. Internal Producers (Conditionally Trusted)

Fontes internas do próprio ambiente, que no mundo real viriam de sistemas legados/bancários:

* **Core Banking / Risk / Fraud / CRM / Canais**

  * Eventos de transação, scores existentes, alertas, histórico.
* **Data Warehouse / Data Marts / Relatórios**

  * Bases consolidadas usadas como entrada para modelos.
* **Security & Ops Telemetry**

  * Logs, métricas, eventos de segurança.

**No lab:** podem ser simulados via:

* Bancos de dados locais (Postgres),
* Arquivos CSV/Parquet,
* Tasks de Airflow/ETL que geram dados sintéticos.

Mesmo sendo “internos”, **não têm passe livre**: qualquer fluxo em direção ao pipeline deve seguir as mesmas regras de borda (passar por Z1).

---

## 3. Riscos em Z0

>Nota: Z0 não executa controle técnico ainda, mas define **de onde vem o risco**. Aqui você mostra maturidade de threat modeling focado em dados.

### 3.1 Data poisoning (externo, parceiro, interno)

**O que é**

Inserção intencional ou acidental de dados manipulados que:

* Enviesam o treino,
* Enfraquecem detecções de fraude,
* Criam backdoors comportamentais em modelos.

**Cenários**

* Cliente malicioso alimentando cadastro/transações “limpas” para normalizar fraude.
* Parceiro enviando dataset corrompido (seja erro ou comprometimento de segurança).
* Sistema interno com bug começando a gerar campos com valores errados (ex: score invertido, datas trocadas).

**Impacto no lab**

* Qualquer CSV/JSON que você “considerar verdade” sem passar por Z1/Z2 pode virar veneno do modelo.

> Z0 deixa claro: **toda fonte é candidata a poisoning até ser validada**.

---

### 3.2 Arquivos malformados / conteúdo malicioso

**O que é**

Uploads ou feeds com:

* Formatos quebrados (CSV truncado, JSON inválido),
* Payload inesperado (binário onde deveria ser texto),
* Malware embutido (macro, scripts, PDFs maliciosos).

**Cenários**

* Upload de documento por cliente.
* Parceiro mandando lote com anexos.
* Dump interno “rápido” copiado pra área compartilhada (ctrl-c + ctrl-v com dados).

**Por que importa**

* Pode quebrar pipelines,
* Pode explorar parsers (ferramentas que manipulam estes dados) e libs (também utilizadas para manipular os dados),
* Pode virar vetor lateral dentro da infra de dados.

> Z0 assume que todo arquivo que chega **pode estar errado ou hostil**; Z1 é obrigado a tratar.

---

### 3.3 Dados sensíveis, ilegais ou fora de propósito

**O que é**

Dados que:

* Não deveriam ser usados no contexto de IA/ML,
* Excedem o mínimo necessário (violando minimização),
* Misturam PII + contexto sensível sem base legal.

**Cenários**

* Parceiro inclui CPF completo onde só precisava hash.
* Export interno envia colunas de senha/segredo, token, etc.
* Fonte pública “contaminada” com dado pessoal scrapado.

**Risco**

* Violação LGPD/regulação bancária,
* Bbrigação de apagar/re-treinar,
* Risco reputacional e legal.

---

### 3.4 Falta de proveniência e rastreabilidade

**O que é**

Não saber:

* Quem produziu,
* De qual sistema saiu,
* Qual versão do schema,
* Sob qual contrato/regra.

**Impacto**

* Dificulta explicar decisões,
* Impede reproduzir modelos,
* Atrapalha investigação de fraude / incidente.

**No lab**

* Se você não “marca” as fontes Z0, tudo vira massa amorfa — exatamente o que queremos evitar conceitualmente.

---

### 3.5 Cadeia de suprimentos de dados e modelos (data & model supply chain)

**O que é**

Confiar cegamente em:

* APIs de scoring,
* datasets “prontos”,
* modelos externos, sem:
  * due diligence,
  * garantias de integridade,
  * cláusulas de segurança.

**Risco**

* Dados adulterados,
* Modelo baseado em fonte tóxica,
* Exposição de dado ao terceiro sem controle.

---

### 3.6 Shadow data / caminhos paralelos

**O que é**

Times ou sistemas mandando dados para o pipeline por:

* SCP direto,
* Share interno,
* Bucket clandestino,
* Serviço fora do padrão.

**Risco**

* Bypass de validação,
* Dados sensíveis sem controle,
* Fonte não catalogada treinando modelo crítico.

> Em Z0 você define: **só existe "fonte oficial" se estiver registrada e entrar via Z1**. O resto é risco.

---

## 4. Controles esperados para Z0

>Reforçando: Z0 é **onde declaramos as regras de quem pode falar com a plataforma**. A implementação prática desses controles começa em Z1, mas nasce aqui como política.

### 4.1 Allowlist de fontes e canais oficiais

**Ideia**

Só é reconhecido como fonte Z0 “legítima” aquilo que:

* Está documentado (catálogo),
* Tem dono (owner),
* Tem contrato (interno ou externo),
* Entra por um canal controlado.

**Como isso aparece no lab**

* No README/tutoriais:
  * Você deixa claro quais rotas do reverse proxy/API GW representam fontes externas/parceiras,
  * Quais buckets/pastas de ingestão são usados,
  * Quais conexões Airflow simulam sistemas internos.
* Tudo fora disso é considerado **shadow** (flag de risco).

**Por quê**

Evita que qualquer arquivo jogado “do lado” acabe entrando no pipeline sem passar pelos gates.

---

### 4.2 Contratos de Schema, Formato e Qualidade

**Ideia**

Antes mesmo de chegar em Z1, cada fonte Z0 deveria ter um **data contract** mínimo:

* Campos obrigatórios/opcionais,
* Tipos fortes (string, int, date),
* Domínios válidos (ex: status ∈ {Aprovado, Negado}),
* Formato técnico (CSV, JSON, Avro, Parquet),
* Regras de nulidade, ranges.

**Como isso aparece no lab**

* Definições em YAML/README para cada dataset de exemplo.
* Tasks de validação em Z1/Z2 usam esses contratos (ex.: Great Expectations, validações simples no Airflow).

**Relação com risco**

* Sujeito a ML01/ML02 (OWASP ML) → sem contrato, qualquer ruído entra.
* Contrato é a base para as validações de Z1.

---

### 4.3 Validação de origem e identidade

**Ideia**

Toda fonte Z0 séria tem:

* Identificação técnica:
  * Chave de API,
  * Cliente OIDC/OAuth,
  * MTLS,
  * Credenciais armazenadas em Vault (no lab),
* Restrições:
  * IP allowlist,
  * Escopos/roles limitados (RBAC/ABAC),
  * Logs de uso.

**No lab**

* Reverse proxy/API GW frontando serviços → simula autenticação e controle de quem pode enviar dados.
* Vault/IdP usados para entregar credenciais com segurança a componentes legítimos.

**Por quê**

* Impede que “qualquer um” empurre dado para dentro.
* Amarra com NIST/OWASP/CSA:

  * **NIST-style**: controle de acesso, canal seguro, auditoria.
  * **CSA AICM**: autenticação, autorização, proveniência.
  * **OWASP API**: proteção de endpoints.

---

### 4.4 Classificação de dados e finalidade de uso

**Ideia**

Z0 é onde você começa a dizer:

* Que tipo de dado é (público, interno, confidencial, sensível),
* Para que ele pode ser usado:

  * Treino,
  * Teste,
  * Monitoração,
  * Antifraude,
  * Reporting,
* o que **não** pode (ex.: “não usar este dataset para modelos de marketing”).

**No lab**

* Você pode:

  * Anotar datasets de exemplo com nível (ex.: `classification: internal`, `pii: true/false`),
  * Indicar nos docs:

    * “Este dataset simula PII, logo exigiria mascaramento/pseudonimização antes de treino”.

**Por quê**

* Liga direto com LGPD, AICM, NIST AI RMF:

  * Evita uso indevido,
  * Prepara o terreno para controles em Z2–Z5 (mascarar, limitar acesso, etc.),
  * Apoia explicabilidade (“esse modelo só usa atributos X e Y, não dados proibidos”).

---

### 4.5 Como isso se conecta com a linha do README (Z0)

No quadro do README você tem:

> **Z0** | Fontes externas, parceiras e internas
> **Riscos**: Data poisoning, arquivos malformados
> **Controles**: Allowlist, schema/MIME, validação de origem

Com o detalhamento acima, você ganha:

* Narrativa clara pra documentação,
* Conteúdo pra explicar em uma entrevista:

  * “Antes de falar de WAF, Vault, MLflow, eu começo em Z0 definindo quem pode produzir dados, como, com qual contrato, identidade, classificação e finalidade. Tudo o que não segue isso é tratado como risco e bloqueado em Z1.”

---

## 5. Teoria ↔ Prática (como Z0 se conecta aos Docs do lab)

Ainda que os Docs 01–07 não sejam “sobre Z0”, eles dão base para que Z0 seja tratada corretamente:

* **01 / 03 / 05 — Vault, OIDC, IAM**

  * Garantem identidade forte e entrega segura de segredos para serviços que consomem fontes.
* **02 — Core Infra & Segurança**

  * Garante que apenas caminhos oficiais (reverse proxy, portas definidas) existam para ingestão.
* **04 — Airflow / MLflow / FastAPI**

  * Tasks de ingestão no Airflow podem representar fontes Z0 com:

    * conexões autenticadas,
    * validações de formato,
    * uso de credenciais seguras via Vault.
* **06 — CI/CD seguro**

  * Garante que qualquer componente que consome fontes Z0 seja construído com segurança (SAST, SBOM, policies).

Quando você escrever os tutoriais do lab, sempre deixe claro:

* “Esta fonte de dados representa uma origem Z0”,
* “Ela só é aceita porque passa pelas verificações definidas em Z1”.

---

## 6. Alinhamento com Frameworks NIST-style risk-based + OWASP + CSA (AICM)

* **OWASP Top 10 / OWASP API Security**

  * Segurança dos aplicativos que geram dados (injeção, auth quebrada, etc.).
* **OWASP LLM / GenAI**

  * Se LLM for usado para coletar/anotar dados, trate como fonte não confiável: risco de prompt injection/exfiltração.
* **CSA AI Controls Matrix (AICM)**

  * Domínios sobre:

    * data provenance,
    * third-party management,
    * governance de dados de treino.
* **NIST-style (AI RMF / 800-53)**

  * Terceiros (SA-9, SR-*),
  * proteção de comunicações (SC-*),
  * auditoria (AU-*),
  * integridade (SI-7),
  * tudo pensado desde a origem.
* **Princípios bancários**

  * Zero trust,
  * “Somente canais oficiais”,
  * “Sem dado sem dono”.
