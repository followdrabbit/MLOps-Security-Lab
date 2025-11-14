# Z3 — Curated Data & Feature Store (Governed & Trusted)

↩️ [Voltar ao README — Mapa Z0–Z9](../../README.md)

![Diagram](../../assets/Project_Diagram-Z3.drawio.svg)

## 1. Papel da Z3 no MLOps Security Lab

A **Z3 é a camada onde o dado bruto (Z2) vira dado utilizável com governança**.
É o ponto de **curadoria, padronização, anonimização, qualidade e versionamento semântico**, servindo tanto:

* aos **domínios de negócio** (relatórios, análises, APIs internas), quanto
* à **Fábrica de Modelos (Z4)**, por meio de uma **Feature Store confiável**.

Enquanto a Z2 é o cofre de tudo que entrou, a Z3 é:

* a **fonte oficial de leitura** para analytics, engenharia de dados e modelos,
* o lugar onde aplicamos **regras de negócio, qualidade, privacidade, minimização e lineage**,
* o ponto que **quebra o acoplamento direto com o Raw** (ninguém mais deveria atacar Z2 diretamente, salvo exceções controladas). 

No contexto do **MLOps Security Lab**, a Z3 é composta por:

* **Curated Zone** (datasets tratados, normalizados, versionados),
* **Feature Store** (camada de features offline/online com contrato estável),
* **mecanismos de Data Quality, Catalog, Classificação, Masking, Tokenização e Governança**.

**Princípios-chave da Z3:**

1. **Só entra dado vindo da Z2 (ou de fontes internas oficiais), nunca direto da Z0/Z1.**
2. **Dado é tratado segundo contratos, regras de negócio e políticas de privacidade.**
3. **Feature = produto gerenciado (dono, documentação, testes, versão, SLA).**
4. **Consumidores (Z4/Z7) preferem sempre Z3/Feature Store a acessar Raw diretamente.**

---

## 2. Componentes da Z3

Z3 combina **engenharia de dados + governança + MLOps**.
Abaixo a visão macro (cada item terá um arquivo de detalhamento, como em Z1/Z2).

### 2.1 Curated Zone (Camada Curada) — [Detalhamento](./Z3-2.1.md)

Camada de dados **tratados, limpos e alinhados ao modelo de negócio**, derivada exclusivamente da Z2.

* Normalização de schemas,
* padronização de tipos, unidades, referências,
* remoção ou mascaramento de PII quando não necessária,
* aplicação de **regras de negócio oficiais** (ex.: conceitos de cliente, conta, contrato, transação, limites, status).

**Ideia:**
É aqui que “JSON caótico + CSV de parceiro + log de evento” viram tabelas/datasets consistentes, versionados e documentados.

---

### 2.2 Data Modeling & Schema Registry — [Detalhamento](./Z3-2.2.md)

Mecanismos para **definir, versionar e validar esquemas** de dados curados:

* Modelagem de dados orientada a domínio (Data Vault / Dimensional / Wide Tables).
* Registro de schemas (ex.: **Schema Registry**, contratos Avro/JSON, migrações versionadas).
* Validação contínua de schema nas pipelines (CI de dados).

**Objetivo:**
Evitar *schema drift silencioso*, garantir compatibilidade, dar previsibilidade para Z4/Z7 e alinhar com boas práticas de **contratos de API/dados** (OWASP API Security, NIST CM).

---

### 2.3 Data Quality & Validation as Code — [Detalhamento](./Z3-2.3.md)

Data quality sai do Excel e vira **código versionado e executado em pipeline**:

* Regras de:

  * completude, unicidade, integridade referencial,
  * ranges de valor, distribuições esperadas, regras de negócio,
  * detecção de outliers e anomalias (ex.: volumes, padrões de fraude).
* Integração com ferramentas tipo Great Expectations/Deequ (no lab, simulado).
* “Teste unitário de dado”: se falhar, o dataset não é promovido para consumo.

**Objetivo:**
Reduzir **drift de dados, garbage in / garbage out** e mitigar riscos de:

* previsões incorretas,
* decisões erradas em crédito/fraude,
* violações de OWASP ML Top 10 (data poisoning, integrity issues).

---

### 2.4 Data Catalog & Lineage — [Detalhamento](./Z3-2.4.md)

Catálogo centralizado com:

* descrição dos datasets e features,
* donos (data owners / stewards),
* classificações de sensibilidade,
* lineage ponta a ponta:

  * Z0/Z1 → Z2 (Raw) → Z3 (Curated/Features) → Z4/Z5/Z6.

**Objetivo:**
Permitir descobrir dados com contexto, reforçar **accountability**, dar visibilidade para auditoria e para a equipe de segurança (Z8/Z9).

---

### 2.5 Classificação, Pseudonimização & PII Handling — [Detalhamento](./Z3-2.5.md)

Aqui entram as políticas de **privacidade e proteção de dados**:

* Classificação automática/manual (PII, SPI, confidencial, regulado).
* **Pseudonimização e tokenização** de identificadores sensíveis (ex.: CPF, cartão, chave PIX).
* **Mascaramento** para usos analíticos onde o identificador não é necessário.
* Regras de minimização:

  * “por padrão, use versão anonimizada”,
  * “use PII real apenas se estritamente necessário e autorizado”.

**Objetivo:**
Alinhar Z3 com **LGPD/GDPR**, CSA CCM (DSI, IAM), NIST (AR, PT), e reduzir risco de vazamento ou uso indevido de dados sensíveis.

---

### 2.6 Feature Store (Offline & Online) — [Detalhamento](./Z3-2.6.md)

A **Feature Store** é a materialização do conceito de:

> “Features são produtos de dados governados.”

Inclui:

* **Feature Store Offline**

  * Tabelas/tópicos analíticos com histórico de features,
  * versionadas, derivadas da Curated Zone,
  * usadas para treino/re-treino de modelos (Z4).
* **Feature Store Online**

  * Serviço/banco de baixa latência,
  * expõe features calculadas/pre-agrupadas para **inferência em Z6**,
  * com **consistência lógica** com as features de treino (no leakage / no train–serving skew).

Controles de segurança:

* Autenticação forte (IAM/Z8),
* autorização por feature/group,
* logs de acesso,
* isolamento por tenant/domínio,
* uso exclusivo de **modelos aprovados (Z5)** como fonte de features derivadas.

---

### 2.7 Data Products & Access Patterns — [Detalhamento](./Z3-2.7.md)

Definição de **“produtos de dados”** e contratos de consumo:

* Exposição via:

  * views materiais,
  * APIs internas,
  * endpoints de consulta (GraphQL/REST/gRPC),
  * notebooks controlados.
* Documentação clara:

  * quais campos, semântica, limites, SLAs, exemplos.
* Governança:

  * cada data product com owner,
  * regras de quem pode consumir (RBAC/ABAC),
  * alinhado ao modelo de domínios (Data Mesh, se aplicável).

**Objetivo:**
Evitar consultas diretas e ad hoc à Raw Zone; promover um **acesso previsível, auditável e de baixo acoplamento**.

---

### 2.8 Logging, Observabilidade & Drift de Dados — [Detalhamento](./Z3-2.8.md)

Monitorar não só infraestrutura, mas **saúde do dado curado e das features**:

* métricas de:

  * frescor (freshness),
  * completude,
  * cardinalidade,
  * distribuições,
  * correlação com labels (para detectar **data/label drift**).
* logs de:

  * quem acessa quais datasets/features,
  * quando, quanto, de onde.
* integração com **Z9 — Monitoring & Audit**:

  * alarmes para:

    * queda na qualidade,
    * mudanças bruscas de distribuição,
    * acessos suspeitos a features sensíveis.

**Objetivo:**
Transformar Z3 em **sensor de anomalias de dados** e de uso, alimentando a defesa contra:

* ML drift,
* poisoning,
* uso indevido de features sensíveis.

---

## 3. Riscos × Controles × Frameworks (Z3)

Na Z3, o foco muda de “entrada segura” (Z1) e “cofre bruto” (Z2) para:

* **correção sem corromper o histórico**,
* **qualidade & consistência**,
* **privacidade & minimização**,
* **segurança da cadeia de features para ML**.

Abaixo os principais riscos e como os componentes da Z3 respondem.

---

### 3.1 Dados incorretos ou inconsistente (Garbage In → Garbage Model)

**Risco**

* Datasets curados com erros silenciosos:

  * joins errados,
  * duplicidades,
  * valores fora de regra de negócio,
  * schema drift não detectado.
* Modelos aprendendo padrões falsos, decisões injustas ou perdas financeiras.

**Controles Z3**

* **2.1 Curated Zone** com regras de negócio explícitas.
* **2.2 Schema Registry** com validações de contrato.
* **2.3 Data Quality as Code** com gates em pipelines.
* **2.8 Monitoramento de qualidade & drift**.

**Frameworks**

* OWASP ML Top 10 — Data Poisoning / Integrity (ML01, ML10).
* NIST SP 800-53: SI-10, SI-7 (validação e integridade).
* CSA AICM — Data Quality & Integrity.

---

### 3.2 Vazamento de PII e dados sensíveis na camada curada/Features

**Risco**

* Tabelas curadas ou features expondo:

  * CPF, e-mail, endereço, dados financeiros,
  * ou correlações que permitem reidentificação.
* Sistemas de negócio ou times de DS acessando mais do que precisam.

**Controles Z3**

* **2.5 Classificação & Pseudonimização**
* **2.4 Catalog & Lineage** com marcação de sensibilidade.
* **2.6 Feature Store** com controle de acesso por feature.
* **2.8 Logging de acesso** integrado ao SIEM (Z9).
* IAM/ABAC (Z8) aplicado por dataset/feature.

**Frameworks**

* OWASP A01/A02 (Broken Access Control / Cryptographic Failures).
* NIST SP 800-53: AC-6, SC-28, PL-2, AR-*.
* CSA CCM: DSI, IAM, SEF; LGPD/GDPR (minimização, necessidade, propósito).

---

### 3.3 Target Leakage & Leakage de Sinais Sensíveis

**Risco**

* Features curadas incluem:

  * rótulos futuros,
  * atributos pós-fato (ex.: “foi inadimplente?” dentro do conjunto de treino),
  * proxies para PII/sensíveis não autorizados.
* Modelos têm performance irreal e comportamento discriminatório.

**Controles Z3**

* **2.3 Data Quality & Regras de Negócio**

  * checagens de causalidade/temporalidade.
* **2.6 Feature Store**

  * separação clara entre features online vs. labels,
  * revisão de features por time de dados/risco.
* **Data Product Governance (2.7)**

  * aprovação de features críticas,
  * documentação do racional de cada feature.

**Frameworks**

* OWASP ML Top 10 — ML03 (Training Data Poisoning), ML07 (Privacy).
* NIST AI RMF — Governança, Validação, Testing.
* CSA AICM — Fairness & Transparency.

---

### 3.4 Acesso direto à Z2 (bypass da Z3)

**Risco**

* Times ou serviços ignoram Z3 e consomem Raw:

  * reconstruindo lógica própria,
  * replicando bugs,
  * fugindo das regras de privacidade/qualidade.

**Controles Z3 + Z2**

* Arquitetura e políticas reforçando:

  * consumo **oficial** via Z3/Feature Store.
* **2.7 Data Products**

  * oferecendo caminhos fáceis, documentados e performáticos.
* IAM/Network Policy (Z2) para restringir acessos diretos.
* Logs em Z2/Z3 (2.8 + Z2.8) para detectar bypass. 

**Frameworks**

* NIST: AC-4, CM-7, GRC.
* CSA CCM: IVS, IAM, GRC.

---

### 3.5 Falta de Lineage & Transparência

**Risco**

* Ninguém sabe:

  * como um campo foi calculado,
  * de quais fontes veio,
  * qual versão do dataset alimentou o modelo.
* Impossível explicar decisões para regulador/cliente.

**Controles Z3**

* **2.4 Data Catalog & Lineage**

  * gráficos de dependência entre Z0/Z1/Z2/Z3/Z4/Z5/Z6.
* Metadados obrigatórios:

  * `source`, `transformation`, `owner`, `schema_version`.
* Integração com Model Registry (Z5):

  * cada modelo referencia datasets/features específicos.

**Frameworks**

* CSA AICM / CCM — Data Lineage & Transparency.
* NIST AI RMF — Govern, Map, Measure.

---

### 3.6 Segurança da Feature Store & Supply Chain de Features

**Risco**

* Feature Store comprometida:

  * features manipuladas (model poisoning),
  * alterações maliciosas em embeddings / vetores,
  * dependência de fontes externas sem validação.

**Controles Z3**

* **2.6 Feature Store**:

  * autenticação forte, mTLS, RBAC/ABAC,
  * versionamento de features,
  * only-from:

    * Z3/Z4 pipelines assinados,
    * modelos aprovados (Z5).
* **2.3 / 2.8**:

  * monitoramento de alterações e acessos,
  * regras para detectar mudanças anômalas em distribuições de features.

**Frameworks**

* OWASP ML Top 10 — ML06 (Supply Chain Attacks).
* NIST SP 800-53: SA-*, SI-7, CM-5.

---

## 4. Teoria ↔ Prática (como Z3 se conecta aos Docs do Lab)

No **MLOps Security Lab**, a Z3 se materializa combinando vários componentes que você já está documentando:

* **02-Core-Infraestrutura-e-Seguranca.md**

  * Provisiona bancos/warehouses (ex.: Postgres/ClickHouse) em rede interna para Curated/Features.
* **04-Servicos-MLOps-(Airflow-MLflow-FastAPI).md**

  * DAGs de curadoria:

    * leem de Z2 (Raw),
    * aplicam transformações, DQ, mascaramento,
    * escrevem em `curated/` e `feature_store/`.
  * Serviços de consulta (APIs internas) consumindo apenas Z3/Feature Store.
* **05-OIDC-e-Seguranca-Avancada.md & 08-Z8 (Security & Trust)**

  * IAM/IdP + Vault + KMS aplicados também à camada de leitura (Z3),
  * RBAC/ABAC por dataset/feature.
* **06-CI-CD-e-GitHub-Actions-Vault-OIDC.md**

  * Pipelines de dados como código:

    * testes de schema & DQ em PR,
    * políticas como código para acesso aos datasets e features,
    * integração com scanners/supply chain.

Mensagem pra entrevista:

> “Se Z2 é o *what happened*, Z3 é o *what it means* — mas com governança.
> Só depois que o dado passa por curadoria, qualidade, privacidade, classificação e lineage é que ele vira insumo oficial para modelos (Z4/Z5) e para os consumidores (Z7).”

---

## 5. Alinhamento com Frameworks (Z3)

Z3 é onde você mostra maturidade de **Data Governance + MLOps + Segurança + Privacidade** trabalhando juntos.

* **OWASP ML Security Top 10**
  Foca em proteção da cadeia de dados e features, validação de input, defesa contra poisoning, controle de acesso a modelos e dados.
* **OWASP GenAI / LLM Top 10**
  Para features e contextos usados em LLMs (prompt injection, data exfiltration, training data poisoning).
* **NIST SP 800-53 Rev.5**

  * AC, AU, CM, PL, RA, SA, SC, SI — aplicados a dados, pipelines e repositórios.
* **NIST AI RMF**

  * Governança de dados de treino, teste, validação e monitoramento contínuo.
* **CSA Cloud Controls Matrix (CCM) + CSA AI Controls Matrix (AICM)**

  * DSI (Data Security), IAM, LOG, SEF, TVM, AIS (AI Security).
* **Reguladores financeiros & privacidade (BACEN, LGPD/GDPR)**

  * Z3 é o ponto de controle chave para:

    * segregação de dados sensíveis,
    * minimização,
    * auditabilidade de modelos de crédito, fraude, PLD/FT.
