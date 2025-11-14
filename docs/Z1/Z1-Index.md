# Z1 — Ingestion & Security Gateway

↩️ [Voltar ao README — Mapa Z0–Z9](../../README.md)

![Diagram](../../assets/Project_Diagram-Z1.drawio.svg)

## 1. Papel da Z1 no MLOps Security Lab

A **Z1 é a porta de entrada controlada** entre as fontes da Z0 e o ambiente interno (Z2+).
Tudo que sai da **Z0 — Data Sources** e quer entrar na plataforma de IA/ML/LLM **precisa obrigatoriamente atravessar a Z1**. 

No contexto do **MLOps Security Lab**:

* Concentra a **exposição mínima** de serviços de ingestão.
* Implementa filtros técnicos e de segurança (*front door de dados*).
* Garante que apenas fluxos:

  * autenticados,
  * autorizados,
  * validados,
  * auditáveis,
    avancem para o **Raw Data Lake (Z2)**.

Conecta diretamente com o README unificado:

> “Z1 — Camada de ingestão controlada → Riscos: DoS, malware, exfiltração via upload → Controles: Rate limiting, limites de tamanho, CDR, sandbox, validação de conteúdo.”

**Princípio chave do lab:**
Se não passa por Z1, **não entra no ecossistema de dados e modelos**.

---

## 2. Componentes da Z1

A Z1 é composta por blocos que, juntos, formam o **Ingestion & Security Gateway**.

### 2.1 Reverse Proxy / API Gateway (Ingestion Gateway) - [Detalhamento](./Z1-2.1.md)

Responsável por:

* Ser o **único endpoint público** (ou de borda) exposto.
* Terminar TLS (HTTPS).
* Roteamento apenas para serviços internos autorizados.
* Aplicar:

  * restrições de método (ex.: só `POST /ingest`),
  * regras de path,
  * headers de segurança.

**No lab:**

* Implementado com Traefik/Nginx como front door de ingestão.

---

### 2.2 WAF (Web Application Firewall) - [Detalhamento](./Z1-2.2.md)

Colocado na frente ou integrado ao gateway para:

* Inspecionar requisições,
* Bloquear padrões conhecidos de ataque:

  * injection, XSS, path traversal, etc.

**No lab:**

* Regras básicas de WAF já demonstram o conceito:

  * nada de payload livre,
  * nada de endpoint “cru”.

---

### 2.3 Autenticação & Autorização (AuthN/AuthZ) na borda - [Detalhamento](./Z1-2.3.md)

Todo produtor (externo ou interno) que envia dados deve:

* Ser autenticado:

  * OIDC/OAuth2,
  * API keys gerenciadas,
  * mTLS para integrações sensíveis.
* Ser autorizado:

  * cada identidade tem acesso somente às rotas/tópicos/datasets definidos.

**No lab:**

* Integração com IdP/Keycloak e Vault:

  * service accounts,
  * tokens emitidos de forma segura,
  * nada de credencial hardcoded.

---

### 2.4 Content Validation (Validação de Conteúdo) - [Detalhamento](./Z1-2.4.md)

Antes de aceitar o dado, Z1 checa:

* **Content-Type / MIME** permitido.
* **Tamanho máximo** (payload e arquivo).
* **Formato básico**:

  * JSON válido,
  * CSV com número correto de colunas,
  * campos obrigatórios presentes.
* Aderência ao **data contract** definido em Z0.

**No lab:**

* Handlers de ingestão + validações simples (FastAPI/Airflow) para não deixar lixo passar.

---

### 2.5 Anti-malware, CDR & Sandbox - [Detalhamento](./Z1-2.5.md)

Para uploads de arquivos:

* Varredura com antivírus / engine de segurança.
* Uso de **CDR (Content Disarm & Reconstruction)** para remover conteúdo ativo (quando aplicável).
* Isolamento/sandbox para tipos de alto risco.

**Objetivo:**

* Cumprir o papel de proteção contra código malicioso nos pontos de entrada (NIST SI-3).

**No lab:**

* Pode ser simulado com:

  * chamadas para um serviço de scan,
  * validações de extensão + bloqueio de tipos proibidos.

---

### 2.6 Data Quality & Anomaly Checks (Higiene inicial) - [Detalhamento](./Z1-2.6.md)

Z1 também aplica checagens leves de sanidade:

* Campos críticos não nulos,
* ranges minimamente plausíveis,
* consistência estrutural por lote.

Não é “modelagem”, é **higiene de entrada** para reduzir lixo óbvio.

---

### 2.7 ETL / Streaming / MQ (Conectores Oficiais) - [Detalhamento](./Z1-2.7.md)

Após passar pelos controles:

* Dados são enviados para:

  * filas,
  * tópicos,
  * jobs de ETL,
    que escrevem em **Z2 — Raw Zone**.

**Somente** conectores oficiais (ex.: Airflow, serviços internos autenticados) podem:

* ler da camada de ingestão,
* persistir em Z2.

---

### 2.8 Auditoria & Observabilidade - [Detalhamento](./Z1-2.8.md)

Z1 registra:

* origem (cliente/sistema/parceiro),
* identidade utilizada,
* tipo e tamanho de payload,
* resultado (aceito, rejeitado, motivo),
* correlações com WAF/AV.

Esses eventos vão para **Z9 — Monitoring & Audit**.

---

## 3. Riscos × Controles × Frameworks (Z1)

> Z1 já é uma zona com **controles ativos**, mas é a linha de frente: se falhar aqui, você contamina Z2+ e enfraquece todo o ecossistema.
> Abaixo, cada risco está ligado diretamente aos componentes da Z1 (seção 2) e aos frameworks de mercado.

---

### 3.1 DoS / Abuso de API / Consumo excessivo

**Risco**

* Flood de requisições, uploads massivos ou uso abusivo das rotas de ingestão, levando à exaustão de CPU, memória, disco ou banda.
* Indisponibilidade da ingestão ou da própria plataforma.

**Controles associados (seção 2)**

* **2.1 Reverse Proxy / API Gateway**

  * Ponto único de entrada → facilita aplicar limites e proteger tudo em um só lugar.
* **2.2 WAF**

  * Pode bloquear padrões básicos de ataque volumétrico e requests suspeitas.
* **2.4 Content Validation**

  * Limites de tamanho por endpoint.
* **Rate limiting / throttling no gateway**

  * Políticas por identidade, rota e método.
* **2.8 Auditoria & Observabilidade**

  * Métricas e logs para detectar spikes e abuso.

**Frameworks**

* **OWASP API Security API4:2023 — Unrestricted Resource Consumption**.
* **NIST SP 800-53**: SC-5, SC-6 (proteção contra DoS), SC-8 (comunicações protegidas).
* **CSA CCM**: domínios de disponibilidade, proteção de fronteira e monitoramento.

---

### 3.2 Upload inseguro / arquivos maliciosos

**Risco**

* Arquivos contendo malware, webshell, macros maliciosas ou PDFs exploráveis.
* Comprometimento de parsers, serviços internos e possibilidade de movimento lateral.

**Controles associados (seção 2)**

* **2.4 Content Validation**

  * Allowlist de MIME/extensões, tamanho máximo, formato esperado.
* **2.5 Anti-malware, CDR & Sandbox**

  * Scan AV, CDR para documentos, quarentena/bloqueio.
* **2.7 ETL / Streaming / MQ (Conectores oficiais)**

  * Somente pipelines internos gravam em Z2; nada vai direto do upload pro Raw.
* **2.8 Auditoria & Observabilidade**

  * Registro de uploads suspeitos/rejeitados.

**Frameworks**

* **OWASP File Upload / Unrestricted File Upload** (validação estrita, AV, CDR).
* **NIST SP 800-53**: SI-3 (Malicious Code Protection), SC-7 (boundary protection).
* **CSA CCM**: DSI, IVS (proteção da entrada de dados).

---

### 3.3 Falhas de autenticação ou autorização na borda

**Risco**

* Endpoints sem Auth,
* tokens/genéricas compartilhados,
* falta de segregação entre clientes, parceiros e sistemas internos.
* Resultado: qualquer um injeta dados, não há responsabilização, há mistura de domínios.

**Controles associados (seção 2)**

* **2.3 AuthN/AuthZ na borda**

  * OIDC/OAuth2, API keys gerenciadas, mTLS, escopos/roles por cliente.
* **2.1 Reverse Proxy / API Gateway**

  * Centraliza a aplicação das políticas de AuthN/AuthZ.
* **2.7 Conectores oficiais**

  * Apenas service accounts autorizadas escrevem em Z2.
* **2.8 Auditoria**

  * Loga identidade + ação → accountability.

**Frameworks**

* **OWASP API Security**: Broken Authentication, Broken Access Control.
* **NIST SP 800-53**: AC-*, IA-* (controle de acesso e identidade).
* **CSA CCM**: IAM, AAC.

---

### 3.4 Bypass de validação → data poisoning “limpo”

**Risco**

* Dados “aparentemente ok”, mas semanticamente manipulados:

  * colunas trocadas,
  * valores invertidos,
  * duplicações massivas,
  * padrões estranhos que passam direto pro Raw.
* Isso contamina modelos sem levantar alerta inicial.

**Controles associados (seção 2)**

* **2.4 Content Validation**

  * Enforcement do contrato mínimo: tipos, campos obrigatórios, formato.
* **2.6 Data Quality & Anomaly Checks (Higiene inicial)**

  * Regras simples de sanidade (ranges, enums, proporções).
* **2.3 AuthN/AuthZ**

  * Ajuda a garantir proveniência (quem mandou o quê).
* **2.7 Conectores oficiais**

  * Padronizam como o dado chega em Z2.
* **2.8 Auditoria**

  * Permite rastrear lotes suspeitos até a fonte.

**Frameworks**

* **OWASP (ML / LLM / data integrity)**: alinhado ao risco de poisoning em cadeias de dados.
* **NIST SP 800-53**: SI-7 (integridade), CA-7 (monitoramento contínuo).
* **CSA AICM/CCM**: controles de proveniência e integridade de dados.

---

### 3.5 Injeção, traversal e falhas clássicas de aplicação

**Risco**

* SQL/NoSQL/Command Injection, Path Traversal, SSRF etc. nos endpoints de ingestão.
* Acesso indevido a recursos internos, vazamento de configs/segredos, alteração de dados.

**Controles associados (seção 2)**

* **2.2 WAF**

  * Regras contra padrões de injection, traversal, SSRF conhecidos.
* **2.4 Content Validation**

  * Inputs restritos, tipos previsíveis, menos superfície para injeção.
* **2.1 Reverse Proxy / API Gateway**

  * Roteamento controlado (sem paths “soltos”).
* **2.3 AuthN/AuthZ**

  * Reduz ataque anônimo e automatizado.
* **2.8 Auditoria**

  * Eventos para detecção/reação.

**Frameworks**

* **OWASP Top 10 & API Top 10** (Injection, SSRF, etc.).
* **NIST SP 800-53**: SI-10 (input validation), SC-7 (boundary).

---

### 3.6 Exfiltração via erro, logs ou canais indevidos

**Risco**

* Mensagens de erro expondo detalhes internos,
* logs contendo PII, segredos ou payload completo,
* uso criativo do endpoint pra extrair informação (side-channel).

**Controles associados (seção 2)**

* **2.8 Logging & auditoria seguros**

  * Padrão: logar decisão e metadados, não dados sensíveis.
* **2.1 Reverse Proxy / API Gateway**

  * Padroniza respostas de erro.
* **2.2 WAF**

  * Pode inspecionar e mitigar padrões de resposta suspeitos.
* **2.3 AuthN/AuthZ**

  * Limita quem consegue ver respostas detalhadas.

**Frameworks**

* **OWASP**: Logging & Monitoring, Sensitive Data Exposure.
* **NIST SP 800-53**: AU-*, SC-7, SC-8.
* **CSA CCM**: LOG, DSI.

---

### 3.7 Shadow ingress / bypass da Z1

**Risco**

* Serviços internos expostos diretamente,
* buckets públicos,
* endpoints temporários “por fora”.
* Resultado: dados entrando sem WAF, Auth, validação ou log — quebra do modelo Z0→Z1→Z2.

**Controles associados (seção 2)**

* **2.1 Reverse Proxy / API Gateway como ponto único**

  * Padrão: tudo passa por ele.
* **Configuração de rede / firewall (infra)**

  * Somente gateway exposto; serviços e storage apenas em rede interna.
* **2.7 Conectores oficiais (ETL/MQ)**

  * Só essas identidades escrevem em Z2.
* **2.8 Auditoria & Observabilidade**

  * Correlação de eventos; facilita identificar caminhos suspeitos.
* **Governança/IaC fora do diagrama**

  * Revisão de arquitetura, scans de exposição, políticas internas.

**Frameworks**

* **OWASP / API Security**

  * Endpoints não governados = risco clássico.
* **NIST SP 800-53**: SC-7 (boundary), CM-* (config mgmt), CA-7 (monitoramento contínuo).
* **CSA CCM**: IVS, SEF, GRC.

---

## 4. Teoria ↔ Prática (como Z1 se conecta aos Docs do lab)

Z1 é onde vários docs se encontram:

* **02-Core-Infraestrutura-e-Seguranca.md**

  * Define o reverse proxy/API GW como Ingestion Gateway.
  * Configura TLS, portas mínimas, exposição controlada.
* **01 / 03 — Vault & Vault Agent**

  * Armazenam segredos usados por produtores e conectores de ingestão.
* **04-Servicos-MLOps-(Airflow-MLflow-FastAPI).md**

  * FastAPI pode representar endpoints internos atrás do gateway.
  * Airflow consome apenas de fontes que passaram por Z1.
* **05-OIDC-e-Seguranca-Avancada.md**

  * Implementa autenticação e autorização forte na borda.
* **06-CI-CD-e-GitHub-Actions-Vault-OIDC.md**

  * Garante que alterações na config de gateway/WAF sejam versionadas, revisadas e seguras.

Mensagem pro leitor:

> “Qualquer ingestão que você criar no lab deve obedecer ao padrão Z1. Se você expõe um serviço direto ou grava em Z2 sem passar por Z1, você está simulando um erro de arquitetura, não uma feature.”

---

## 5. Alinhamento com Frameworks (NIST-style + OWASP + CSA)

* **OWASP Top 10 / OWASP API Security**

  * Proteção de endpoints de ingestão, autenticação forte, validação de input, rate limiting, upload seguro.
* **OWASP File Upload / Cheat Sheets**

  * Base conceitual para tratamento de arquivos na borda.
* **NIST SP 800-53 Rev.5**

  * SC-8, SC-13: comunicação protegida.
  * SI-3: proteção contra código malicioso.
  * AC-*, AU-*: acesso e auditoria.
* **CSA Cloud Controls Matrix / CSA AICM**

  * Controles de:

    * interface segura,
    * gestão de chaves,
    * logging,
    * proteção de dados em trânsito,
    * terceiros e integrações.
* **Princípios bancários**

  * Zero trust,
  * Minimizar superfície,
  * Canais oficiais apenas,
  * Tudo autenticado, autorizado e auditado.
