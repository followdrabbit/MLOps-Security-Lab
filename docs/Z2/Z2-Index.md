# Z2 — Raw Data Lake (Restricted)

↩️ [Voltar ao README — Mapa Z0–Z9](../../README.md)

![Diagram](../../assets/Project_Diagram-Z2.drawio.svg)

## 1. Papel da Z2 no MLOps Security Lab

A **Z2 é o primeiro repositório interno confiável** depois da borda segura (Z1).
Tudo que chega aqui:

* já passou por WAF, AuthN/AuthZ, validação de conteúdo, anti-malware/CDR, higiene inicial e conectores oficiais da Z1,
* é armazenado **no formato mais próximo possível da origem** (*raw*),
* é tratado como **fonte de verdade técnica e forense** para os próximos estágios (Z3–Z4–Z5).

No contexto do **MLOps Security Lab**:

* Z2 é implementada como **object storage seguro (ex.: MinIO)** com:

  * criptografia em repouso,
  * controle de acesso forte,
  * versionamento,
  * logs de acesso.
* Não é “pasta bagunçada”: é um **domínio restrito**, com:

  * separação por fonte,
  * políticas rígidas,
  * trilha de auditoria.

**Princípios-chave da Z2:**

1. **Somente escrita via caminhos oficiais (Z1 → conectores).**
2. **Somente leitura por pipelines autorizados (Z3/Z4), nunca por “curioso aleatório”.**
3. **Nada é sobrescrito “no escuro”: versionamento + integridade + auditoria.**
4. **É raw, não é público, não é playground.**

---

## 2. Componentes da Z2

A Z2 é menos “serviços expostos” e mais **como o storage é configurado, segmentado e governado**.

### 2.1 Raw Zone (Object Storage seguro) - [Detalhamento](./Z2-2.1.md)

* Repositório central (ex.: MinIO/S3-like) para armazenamento de dados brutos.
* Organização recomendada:

  * por **fonte** (`source_system`),
  * por **domínio** (fraude, risco, canais, etc.),
  * por **particionamento temporal** (ano/mês/dia),
  * com metadados mínimos (quem ingeriu, quando, via qual pipeline).

**Regras:**

* Apenas **serviços de ETL/ingestão oficiais** podem escrever aqui.
* Acesso de leitura é sempre autenticado, logado e restrito.

---

### 2.2 Criptografia em repouso & KMS - [Detalhamento](./Z2-2.2.md)

* Todos os objetos em Z2 devem ser protegidos com **criptografia em repouso**:

  * chaves gerenciadas por **KMS/HSM/Vault**,
  * nada de chave simétrica hardcoded em app.
* Suporte a:

  * rotação periódica de chaves,
  * segregação de chaves por domínio/sensibilidade.

**Objetivo:**
Se alguém tiver acesso físico ou dump de disco, não lê nada útil.

---

### 2.3 Network Isolation & Endpoint Policy - [Detalhamento](./Z2-2.3.md)

* Z2 só é acessível via:

  * rede interna,
  * endpoints privados,
  * ou service endpoints controlados.
* Sem acesso público direto.
* Sem montar MinIO/S3 aberto na internet.

**Integração com Z1:**

* Somente os conectores oficiais (Z1 → Z2) têm acesso de escrita.
* Ferramentas de Analytics/ML só acessam Z2 a partir de redes/hosts autorizados.

---

### 2.4 Controles de Acesso (IAM / ABAC / Least Privilege) - [Detalhamento](./Z2-2.4.md)

* Acesso baseado em:

  * **service accounts** (Airflow, jobs de curadoria, feature store),
  * **perfis técnicos específicos** (ex.: engenheiro de dados de plataforma).
* Políticas:

  * **Least privilege**:

    * cada serviço só enxerga os buckets/prefixos necessários.
  * sem credenciais compartilhadas entre times/sistemas.
  * nada de usuário humano com permissão ampla de `s3:*` em Z2 em produção.

**Mensagem importante:**

> Leitura direta de Z2 por pessoas deve ser exceção extrema, auditada.
> O caminho normal é Z2 → Z3 (curado) → consumo governado.

---

### 2.5 Versionamento & Imutabilidade - [Detalhamento](./Z2-2.5.md)

* **Versioning habilitado**:

  * cada alteração/overwrite gera nova versão,
  * evita perda acidental,
  * permite reconstruir contexto de treinamento.
* Quando possível, uso de:

  * **pastas WORM / buckets com retenção** (Write Once Read Many),
  * políticas de retenção mínima para dados críticos.

**Por quê:**

* Garante:

  * reprodutibilidade de experimentos de ML,
  * trilha forense (o que existia na época do treino),
  * proteção contra deleção/alteração maliciosa.

---

### 2.6 Integridade & Metadados - [Detalhamento](./Z2-2.6.md)

* Armazenar ou calcular:

  * **hashes (ex.: SHA-256)** por arquivo/lote,
  * metadados de:

    * origem (fonte Z0/Z1),
    * pipeline de ingestão,
    * timestamp de chegada,
    * classificação (se disponível).
* Utilizar esses metadados para:

  * verificar integridade (detectar alteração indevida),
  * rastrear rapidamente a origem de um dado usado em treino.

---

### 2.7 Áreas de Staging, Quarentena & Reprocessamento - [Detalhamento](./Z2-2.7.md)

* Áreas separadas dentro da própria Z2 (ou adjacentes) para:

  * dados parcialmente suspeitos,
  * lotes que falharam em regras de qualidade,
  * dumps para reprocessamento.
* Com políticas mais restritivas de acesso.
* Nada em quarentena deve ser usado diretamente em treino/produção.

---

### 2.8 Logging de Acesso & Integração com Z9 - [Detalhamento](./Z2-2.8.md)

* Todas as operações sensíveis em Z2 devem ser logadas:

  * `GET`, `PUT`, `DELETE`, `LIST`,
  * quem acessou,
  * de onde,
  * qual objeto/prefixo,
  * horário.

* Logs enviados para **Z9 — Monitoring & Audit**:

  * correlação com identidades, pipelines e modelos,
  * detecção de acessos suspeitos a dados sensíveis.

---

## 3. Riscos × Controles × Frameworks (Z2)

> Na Z2, o risco muda: não é mais “entrar qualquer coisa”, é “o que entrou está protegido, íntegro e governado?”.
> Abaixo, os riscos principais e como os componentes da Z2 atuam.

---

### 3.1 Vazamento de dados brutos sensíveis

**Risco**

* Exposição de PII, chaves, segredos ou dados financeiros brutos.
* Acesso direto ao Raw (muito mais rico e menos anonimizado que zonas curadas).

**Controles Z2**

* **2.2 Criptografia em repouso & KMS**
* **2.3 Network Isolation**
* **2.4 IAM com least privilege**
* **2.8 Logging de acesso & Z9**

**Frameworks**

* NIST SP 800-53: SC-28 (proteção de dados em repouso), AC-*, AU-*.
* CSA CCM: DSI, IAM, LOG.
* LGPD/GDPR: minimização de acesso, segurança técnica adequada.

---

### 3.2 Alteração não autorizada (integridade comprometida)

**Risco**

* Dados em Raw sendo alterados, sobrescritos ou injetados depois da ingestão.
* Quebra de confiança, contaminação de pipelines e modelos.

**Controles Z2**

* **2.5 Versionamento & Imutabilidade**
* **2.6 Hashes & metadados**
* **2.4 IAM restritivo**
* **2.8 Logs de escrita/alteração**

**Frameworks**

* NIST SP 800-53: SI-7 (integridade), CM-5.
* CSA CCM: DSI, IAM.
* Suporte à rastreabilidade exigida em ambientes regulados.

---

### 3.3 Exclusão acidental ou maliciosa

**Risco**

* Perda de dados brutos necessários para:

  * reprocessar features,
  * explicar decisões,
  * reproduzir treinos,
  * atender auditorias.

**Controles Z2**

* **2.5 Versionamento**
* Políticas de retenção + proteção contra delete imediato (WORM/retention quando aplicável).
* **2.4 IAM**: proibir `delete` amplo para usuários humanos.
* **Backups e snapshots gerenciados** (infra).

**Frameworks**

* NIST SP 800-53: CP-9/10 (backup/restore), SI-12.
* CSA CCM: BCR, DSI.

---

### 3.4 Uso indevido do Raw (acesso direto por times/modelos)

**Risco**

* Pessoas, notebooks ou modelos consumindo Z2 diretamente:

  * sem curadoria,
  * sem mascaramento,
  * sem revisão de finalidade (LGPD/compliance).

**Controles Z2**

* **2.4 IAM / ABAC**

  * separar roles:

    * plataforma vs. consumo analítico.
* Fluxo padrão: Z2 → Z3 (curado) → consumo.
* **2.8 Logs** para detectar acessos diretos fora do esperado.

**Frameworks**

* NIST: AC-6 (least privilege), AR-*, PL-2 (governança).
* CSA CCM / AICM: governança de dados e AI lifecycle.

---

### 3.5 Shadow pipelines / escrita direta fora da Z1

**Risco**

* Scripts, jobs ou serviços escrevendo direto em Z2:

  * sem passar pela Z1,
  * sem validação, sem AV, sem contrato.

**Controles Z2**

* **2.3 Network Isolation**

  * apenas sub-redes/pods autorizados.
* **2.4 IAM**

  * apenas service accounts oficiais com permissão de `put`.
* Políticas internas + IaC:

  * qualquer exceção é visível e revisada.
* **2.8 Logs**

  * fácil detectar origem de gravações fora do padrão.

**Frameworks**

* NIST: SC-7 (boundary), CM-* (config mgmt).
* CSA CCM: IVS, GRC.

---

### 3.6 Falta de proveniência (não saber de onde o dado veio)

**Risco**

* Não conseguir responder:

  * “Esse dataset usado no modelo X veio da onde?”
  * “Passou por quais controles?”
* Dificulta auditoria, explicabilidade e resposta a incidentes.

**Controles Z2**

* **2.6 Metadados (fonte, pipeline, timestamps, hashes)**
* Integração com logs da Z1 (quem mandou, quando, por qual endpoint).
* Integração com catálogos (Z8) e trilhas (Z9).

**Frameworks**

* CSA AICM & CCM: proveniência e rastreabilidade.
* NIST RMF-style: documentação de fluxo de dados.

---

### 3.7 Treino de modelos com dados não tratados / não conformes

**Risco**

* Times usando diretamente Z2:

  * incluindo dados sensíveis que não deveriam ir para certos modelos,
  * violando LGPD, políticas internas ou limites éticos.

**Controles Z2**

* Reforço arquitetural:

  * modelos devem treinar a partir de Z3 (curada/governada).
* **2.4 IAM**:

  * restringir acesso de pipelines de ML apenas ao necessário.
* **2.8 Logs**:

  * inspeção de quem está lendo o quê.

**Frameworks**

* CSA AICM: controles de dados para AI.
* LGPD/GDPR: base legal, minimização, finalidade.
* NIST AI RMF-style: governança de dados de treinamento.

---

## 4. Teoria ↔ Prática (como Z2 se conecta aos Docs do Lab)

Z2 se materializa nos docs e stack do lab assim:

* **02-Core-Infraestrutura-e-Seguranca.md**

  * Provisiona MinIO/obj storage em rede interna.
  * Configura criptografia, volumes, políticas de acesso.
* **01 / 03 — Vault & Vault Agent**

  * Armazena credenciais de acesso ao storage.
  * Garante que serviços usem segredos dinâmicos, não fixos.
* **04-Servicos-MLOps-(Airflow-MLflow-FastAPI).md**

  * DAGs de ingestão e pipelines escrevem em Z2 com service accounts.
  * Serviços nunca escrevem em Z2 anonimamente.
* **05-OIDC-e-Seguranca-Avancada.md**

  * Integra identidades de serviços/usuários para controle fino de acesso.
* **06-CI-CD-e-GitHub-Actions-Vault-OIDC.md**

  * Infraestrutura como código:

    * buckets, policies, users, roles versionados.
  * Evita criação de “atalhos” fora do padrão.

Mensagem pro leitor:

> “Z2 é o cofre bruto. Só entra dado aprovado pela Z1, só escreve quem é pipeline oficial, só lê quem tem justificativa forte.
> A partir daqui, qualquer modelo sério consegue provar de onde vieram seus dados.”

---

## 5. Alinhamento com Frameworks (Z2)

* **NIST SP 800-53 Rev.5**

  * SC-28: proteção de dados em repouso.
  * AC-*, IA-*: controle de acesso.
  * AU-*: trilhas de auditoria.
  * SI-7: integridade.
* **CSA Cloud Controls Matrix**

  * DSI (Data Security & Information Lifecycle),
  * IAM,
  * LOG,
  * IVS.
* **CSA AI Controls Matrix (AICM)**

  * Proveniência, governança de datasets de treino, gestão de risco de dados.
* **OWASP (visão complementar)**

  * Princípios de proteção de dados sensíveis, hardening de storage, segurança de APIs que alimentam Z2.
* **Princípios bancários/regulatórios**

  * Confidencialidade forte em dados brutos,
  * rastreabilidade,
  * retenção adequada,
  * integridade como pré-requisito para auditoria e modelos de risco.

Se quiser, no próximo passo a gente quebra Z2 em arquivos detalhados (2.1, 2.2, etc.) como você fez com a Z1, mantendo o padrão 100% consistente.
