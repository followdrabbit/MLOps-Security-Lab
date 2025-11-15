# 🛡️ MLOps Security Lab — Arquitetura + Labs em AWS

> Este README unifica a **arquitetura conceitual (Z0–Z9)** e os **laboratórios práticos em AWS**, conectando o *porquê* de cada camada ao *como implementar* com foco em segurança ponta a ponta: entender terminologias, etapas do fluxo MLOps, ataques relevantes e os controles necessários para proteger dados, modelos e infraestrutura.

---

## 0) Visão Geral

- **Objetivo**  
  Simular uma plataforma de IA/ML/LLM com padrão **bancário** de segurança, governança e auditabilidade, usando:
  - um modelo de referência em zonas (**Z0–Z9**), e  
  - **labs em AWS** de baixo custo, alinhados ao free tier sempre que possível.

- **Escopo**  
  Ingestão segura → data lake bruto → zona curada / feature store → fábrica de modelos → registry confiável → serving online e batch → consumidores — tudo apoiado por serviços de identidade, segredos, criptografia, monitoração e auditoria.

- **Teoria ↔ Prática**  
  - As **zonas Z0–Z9** são a espinha dorsal conceitual (documentadas em `docs/`).
  - Os **labs em AWS** implementam subconjuntos da arquitetura (DatOps e MLOps) com foco em boas práticas de segurança desde a concepção.

- **Alinhamento**  
  OWASP (LLM/GenAI), CSA AI Controls Matrix (AICM), visão NIST-style de gestão de risco, além de recomendações de supply chain (SAST, SBOM, segredos, imagens corporativas, etc).

---

## 📁 Estrutura de diretórios

```text
.
├── docs/                # Arquitetura conceitual (Z0–Z9) e visão geral
│   ├── README.md        # Visão da arquitetura Z0–Z9
│   ├── Z0-index.md
│   ├── Z1-index.md
│   ├── Z2-index.md
│   ├── Z3-index.md
│   ├── Z4-index.md
│   ├── Z5-index.md
│   ├── Z6-index.md
│   ├── Z7-index.md
│   ├── Z8-index.md
│   └── Z9-index.md
│
├── assets/              # Diagramas/imagens usados na documentação
│   └── ...
│
├── hardening/
│   ├── github/          # Hardening do repositório GitHub
│   └── aws-account/     # Hardening da conta AWS usada pelos labs
│
└── labs/
    ├── 01-datops-aws/   # Lab de pipeline de dados (Z0–Z3 + Z8/Z9)
    │   ├── README.md
    │   ├── SECURITY.md
    │   ├── docs/        # ADRs, arquitetura detalhada, threat model, etc.
    │   ├── terraform/
    │   ├── src/
    │   ├── schemas/
    │   ├── config/
    │   ├── samples/
    │   └── tests/
    │
    └── 02-mlops-aws/    # Lab de pipeline de MLOps (Z3–Z7 + Z8/Z9)
        ├── README.md
        ├── SECURITY.md
        ├── docs/
        ├── terraform/
        ├── src/
        ├── config/
        ├── notebooks/
        └── tests/
````

## 1) Arquitetura Conceitual (Z0–Z9)

![Diagram](./assets/Project_Diagram.drawio.svg)

A plataforma é organizada em 10 zonas principais, cobrindo desde as fontes de dados até a observabilidade e auditoria:

- **Z0 — Data Sources**  
  Fontes externas, parceiras e internas (APIs, sistemas legados, open data, bureaus, etc.), com contratos de schema, classificação de dados, finalidade de uso e riscos de data poisoning.

- **Z1 — Ingestion & Security Gateway**  
  Porta de entrada controlada para dados: APIs, proxies, autenticação/autorização, validação de conteúdo (MIME, tamanho, schema), mecanismos de quarentena e roteamento controlado.

- **Z2 — Raw Data Lake (Restricted)**  
  Zona de armazenamento bruto, com criptografia, versionamento, políticas restritivas e trilha de auditoria. Serve como base forense e “ponto de verdade” do que foi recebido.

- **Z3 — Curated Data & Feature Store**  
  Dados tratados, padronizados e prontos para uso, com **data quality como código**, anonimização/pseudonimização quando necessário, catálogo, classificação e políticas de acesso.

- **Z4 — Model Factory (Dev/Train)**  
  Ambiente controlado para experimentos e treinamento de modelos, com rastreio de experimentos, integração com segredos (Vault/Secrets Manager), scanners de código/dependência e políticas de uso de dados.

- **Z5 — Model Registry & Governance**  
  Fonte única de verdade sobre modelos: versões, estágios (candidate, staging, production), metadados de treino, dataset de origem, revisões de risco/compliance e trilhas de aprovação.

- **Z6 — Serving & Inference Gateway**  
  Camada responsável por expor modelos (APIs de inferência, jobs batch, gateways de LLM/GenAI), com autenticação, autorização fina, rate limiting, logging e filtros de entrada/saída.

- **Z7 — Consumers (Apps, Sistemas de Negócio)**  
  Sistemas que consomem as saídas dos modelos (core banking, risco, fraude, canais digitais, chatbots, etc.). A regra é consumir sempre via **APIs/contratos**, nunca diretamente dos buckets internos.

- **Z8 — Security & Trust Services**  
  Serviços transversais de segurança: IAM/IdP, Vault/Secrets Manager, KMS/HSM, DLP, catálogos de dados, SIEM, policies-as-code, controles de acesso e segregação de funções.

- **Z9 — Observability & Audit**  
  Logs estruturados, métricas, tracing, monitoração de drift/bias, alertas, dashboards e evidências de auditoria. Viabiliza detecção de problemas, investigação e comprovação de conformidade.

Cada zona é detalhada nos documentos específicos (ex.: `docs/Z0-index.md`, `docs/Z1-Index.md`, `docs/Z2-Index.md`, `docs/Z3-index.md`, …, `docs/Z9-index.md`).

---

## 2) Mapeamento Teoria ↔ Prática (o que você constrói)

| Zona   | Função (Teórica) | Artefatos práticos (Docs / Serviços) | Riscos principais | Controles / Gates |
| ------ | ---------------- | ------------------------------------- | ----------------- | ----------------- |
| [**Z0**](./docs/Z0-index.md) | Fontes externas, parceiras e internas | DAGs de ingestão, contratos de schema, catálogos de fonte | Data poisoning, arquivos malformados | Allowlist, schema/MIME, validação de origem |
| [**Z1**](./docs/Z1-Index.md) | Camada de **ingestão controlada** | Reverse Proxy/API GW + (opcional) WAF, AuthN/AuthZ, Content Validation, Anti-malware/CDR, ETL/MQ | DoS, malware, exfiltração via upload | Rate limiting/throttling, limites de tamanho, CDR, sandbox |
| [**Z2**](./docs/Z2-Index.md) | Data lake bruto restrito | Buckets S3/MinIO criptografados, versionados e assinados | Vazamento, alteração maliciosa | Criptografia, políticas de bucket, isolamento de rede |
| [**Z3**](./docs/Z3-index.md) | Dados curados & Feature Store  | Zonas curadas, feature store versionada | Drift silencioso, exposed PII | Catálogo, classificação, políticas de acesso |
| [**Z4**](./docs/Z4-index.md) | Fábrica de modelos (dev/train) | Workbench seguro, MLflow Tracking ou equivalente, SAST/DAST, SBOM, scan de imagens, checagem de datasets | Supply chain, segredos em código, uso de dados não autorizados | Vault/Secrets Manager, imagens oficiais, policies-as-code, CI com gates |
| [**Z5**](./docs/Z5-index.md) | Registry & governança | Model Registry como origem de verdade, esteiras de aprovação | Modelo não aprovado em produção | Assinatura, revisão, LGPD/compliance, SCDR/ADR |
| [**Z6**](./docs/Z6-index.md) | Serving & inferência | Inference Gateway, serviços online, LLM Gateway, batch, Scored Store | Prompt injection, vazamento de dados, abuso de API | mTLS, AuthZ fina, filtros de saída, logs/audit, policies |
| [**Z7**](./docs/Z7-index.md) | Consumidores | Core banking/risk/fraud, canais, chatbots internos | Uso indevido, acoplamento frágil | Least privilege, contratos de API, versionamento |
| [**Z8**](./docs/Z8-index.md) | Serviços de segurança compartilhados | IAM/IdP, Vault/Secrets Manager, KMS/HSM, DLP, SIEM, catálogo | Identidade fraca, má gestão de chaves | RBAC/ABAC, rotação, segregação de funções |
| [**Z9**](./docs/Z9-index.md) | Observabilidade & auditoria | Logs centralizados, monitoração de drift/bias, trilhas de auditoria | Blind spots, fraude não detectada | Correlação, alertas, auditoria periódica |

> Observação: os labs em AWS podem implementar uma versão mínima desses artefatos (por exemplo, S3 em vez de MinIO, registry “pobre” em S3/Dynamo em vez de MLflow completo), mas sempre seguindo a mesma lógica de risco → controle → evidência.

---

## 3) Labs em AWS (DatOps + MLOps)

Os labs são pensados para serem **modulares**:

- Você pode subir **apenas a pipeline de dados (DatOps)**.
- Ou **apenas a pipeline de modelos (MLOps)**, apontando para um bucket de dados curados já existente.
- Ou ainda subir **ambos** e reproduzir o fluxo ponta a ponta.

> O desenho dos labs considera **custos** como fator crítico: prioriza serviços serverless, uso de free tier e recursos mínimos para tangibilizar o fluxo.

### 3.1) Lab 01 — DatOps em AWS (Z0–Z3 + Z8/Z9)

Foco na **pipeline de dados**, desde a recepção até a disponibilização curada:

- **Z0/Z1**  
  - Catálogo de fontes de dados (data sources) e contratos de schema.  
  - Ingestão via **API Gateway + Lambda**, com validação de conteúdo, domínios conhecidos, tratamento de erros e “quarentena”.

- **Z2**  
  - **S3 Raw** criptografado, com versionamento, acesso mínimo e logs de acesso.  
  - Uso de metadados e prefixos para representar proveniência e classificação.

- **Z3**  
  - Pipeline de curadoria (por exemplo, Lambda + Glue + Athena) para:
    - aplicar regras de data quality,
    - normalizar dados,
    - produzir datasets curados e/ou features.

- **Z8/Z9**  
  - IAM com princípio de menor privilégio (roles específicas para Lambdas, acesso limitado a buckets/prefixos).  
  - CloudWatch/CloudTrail para logs, métricas básicas de ingestão e monitoramento de erros.  
  - Budgets/alertas de custo configurados na conta de lab.

### 3.2) Lab 02 — MLOps em AWS (Z3–Z7 + Z8/Z9)

Foco na **pipeline de modelos**, consumindo dados curados:

- **Z3**  
  - Leitura de dados curados produzidos pelo Lab 01 (ou por outra origem equivalente).

- **Z4**  
  - Treinamento de modelos (por exemplo, script Python rodando localmente ou em serviço gerenciado), com rastreio de:
    - dataset usado,
    - parâmetros de treino,
    - métricas de avaliação.

- **Z5**  
  - “Registry pobre” com **S3 + DynamoDB** (ou similar) armazenando:
    - artefatos de modelo,
    - metadados (versão, dataset, métricas),
    - estágio (candidate, staging, production).

- **Z6/Z7**  
  - Exposição do modelo via **API Gateway + Lambda** (`/predict`).  
  - Consumidores chamam a API e recebem previsões, sem acesso direto aos dados internos.

- **Z8/Z9**  
  - Autenticação/autorização básica para o endpoint, logs de requisição/resposta (sanitizados), métricas de latência, taxa de erro, etc.  
  - IAM segregando leitura de modelos, escrita de logs e acesso a dados curados.

Cada lab possui:

- `README.md` → instruções de uso, pré-requisitos e passos para subir/destruir.  
- `SECURITY.md` → riscos e controles de segurança específicos daquele lab.  
- `docs/` → ADRs, arquitetura detalhada, threat model, data contracts, etc.

---

## 4) Fluxo Fim a Fim (Narrativa Resumida)

1. **Z0 → Z1**  
   Qualquer dado entra apenas pelo **Ingestion Gateway**: é autenticado, validado (MIME/tamanho/schema), opcionalmente inspecionado (AV/CDR) e roteado para os fluxos de ETL/streaming apropriados.

2. **Z1 → Z2 → Z3**  
   Dados brutos vão para a **Raw Zone** (criptografada, versionada, com acesso mínimo).  
   Em seguida, passam por processos de curadoria e data quality até compor a **Curated Zone** e a **Feature Store** governada.

3. **Z3 → Z4**  
   Times de data science treinam modelos em ambiente isolado e controlado, usando apenas datasets autorizados, com segredos fornecidos de forma segura (Vault/Secrets Manager) e rastreabilidade completa da **proveniência**.

4. **Z4 → Z5**  
   Modelos candidatos passam por avaliação funcional, de performance, segurança e risco (incluindo aspectos regulatórios e de ética, como LGPD).  
   Só então são aprovados, “assinados” e registrados no **Prod Model Registry**.

5. **Z5 → Z6**  
   Inferência online acontece somente via **Inference Gateway**; no caso de LLMs, apenas via um **LLM/GenAI Security Gateway**.  
   Jobs batch usam modelos do registry e escrevem em uma **Scored Output Store** governada.

6. **Z7, Z8, Z9**  
   Sistemas de negócio consomem apenas **saídas governadas** (APIs, datasets autorizados).  
   IAM, Vault/Secrets Manager, KMS, DLP, SIEM, logging e trilhas de auditoria garantem identidade forte, minimização de privilégios, monitoração, detecção de anomalias, drift e abusos.

---

## 5) Referenciais de Segurança & Governança

Use o lab para experimentar **como** aplicar boas práticas na prática, conectando o que você faz com frameworks conhecidos:

- **OWASP Top 10 para LLM / OWASP GenAI Security**  
  Proteção contra prompt injection, exfiltração, insegurança em plugins, vulnerabilidades de supply chain, inputs maliciosos, etc.

- **CSA AI Controls Matrix (AICM)**  
  Guia para requisitos de segurança, privacidade, compliance e operação de plataformas de IA, cobrindo temas como gestão de modelos, dados, acesso, riscos e monitoramento.

- **Abordagem NIST-style (risk-based)**  
  Pensar em termos de identificar, proteger, detectar, responder e recuperar, conectando:
  - requisitos de negócio,
  - riscos,
  - controles,
  - evidências,
  - e auditoria contínua.

Em cada componente (lab, recurso AWS, script, pipeline), tente sempre documentar:

- **Qual risco está sendo mitigado.**  
- **Quais controles foram implementados.**  
- **Como isso se alinha aos frameworks acima** (por exemplo, “este controle ajuda a cobrir OWASP GenAI X, CSA AICM Y, função *Protect* no contexto NIST”).

---

## 6) Segurança by Design (Z8/Z9 + Hardening de Conta/Repo)

Segurança não é um “apêndice”: é parte da arquitetura desde o início.

- **Z8 — Security & Trust Services**  
  - IAM, IdP, gestão de segredos, KMS/HSM, DLP, catálogos de dados, SIEM.  
  - Nos labs em AWS, isso se reflete em:
    - uso de roles específicas para cada componente,
    - segredos fora de código (SSM Parameter Store / Secrets Manager),
    - criptografia de buckets e logs.

- **Z9 — Observability & Audit**  
  - Logs estruturados, métricas e auditoria.  
  - Nos labs:
    - CloudWatch Logs e métricas,
    - CloudTrail habilitado na conta,
    - AWS Budgets/alertas para controlar custo.

Além disso, o projeto considera:

- **Hardening do repositório GitHub**  
  - Proteção da branch `main`, PR obrigatória, Dependabot, secret scanning, code scanning, etc.

- **Hardening da conta AWS**  
  - MFA (de preferência físico) no root, root sem access keys,
  - usuários/roles com menor privilégio,
  - budgets e alertas de uso para evitar surpresas de custo.

---

## 7) Quickstart (Labs AWS)

### 7.1) Pré-requisitos

- Conta AWS de laboratório (idealmente separada de ambientes pessoais/produtivos).
- Terraform instalado (se os labs usarem Terraform).
- AWS CLI configurada com um usuário/role que tenha permissões adequadas para criar os recursos do lab.

### 7.2) Passos gerais

1. **Clonar o repositório**

```bash
   git clone <url-do-repo>
   cd <nome-do-repo>
```

2. **Aplicar hardening básico da conta**
   Seguir o que estiver documentado em `hardening/aws-account/` (quando esse diretório estiver disponível):

   * habilitar MFA para root,
   * habilitar CloudTrail,
   * configurar AWS Budgets, etc.

3. **Subir o Lab de DatOps (opcional, mas recomendado)**

   * Entrar em `labs/01-datops-aws/`
   * Seguir as instruções do `README.md` desse lab
   * Anotar:

     * URL da API de ingestão,
     * nome dos buckets raw/curated.

4. **Subir o Lab de MLOps (opcional)**

   * Entrar em `labs/02-mlops-aws/`
   * Informar o bucket/prefixo de dados curados (do Lab 01 ou outro)
   * Seguir as instruções do `README.md` desse lab.

5. **Sempre destruir os recursos ao final**

   * Rodar `terraform destroy` (ou o mecanismo definido no lab) para não estourar o orçamento.

---

## 8) Contribuição & Próximos Passos

* Adicionar novos labs (por exemplo: LLM Gateway, red teaming de modelos, supply chain de modelos).
* Refinar os labs de DatOps/MLOps com mais controles (GuardDuty, Security Hub, WAF), conforme o orçamento permitir.
* Incluir exemplos de Feature Store com classificação de PII, finalidade e policies por coluna/campo.
* Automatizar geração de SCDR/ADR com base nas decisões de deploy e promoção de modelos.
* Publicar runbooks de incidente (vazamento de dados, modelo comprometido, abuso de endpoint de inferência).

Sinta-se à vontade para adaptar este laboratório ao seu contexto, enviar PRs com melhorias ou acrescentar novos cenários de segurança em MLOps/DatOps. 🛡️🤖
