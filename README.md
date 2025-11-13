# 🛡️ MLOps Security Lab — (Arquitetura + Hands-on)

>Este README unifica a arquitetura conceitual (Z0–Z9) e o laboratório prático passo a passo (Docs 01–07), conectando o porquê de cada camada ao como implementar tudo em uma única VPS com hardening aplicado, para apoiar meu aprendizado em MLOps seguro: entender as terminologias, as etapas do fluxo MLOps, os ataques mais relevantes e os controles necessários para proteger dados, modelos e infraestrutura.

---

## 0) Visão Geral

* **Objetivo**: simular uma plataforma de IA/ML/LLM com padrão **bancário**, cobrindo segurança ponta a ponta, governança e auditabilidade — rodando em **uma VPS**.
* **Escopo**: ingestão segura → data lake bruto → zona curada / feature store → fábrica de modelos → registry confiável → serving online e batch → consumidores — apoiados por serviços de segurança e monitoração.
* **Teoria ↔ Prática**: as **zonas Z0–Z9** são a espinha dorsal conceitual. Os **docs 01–07** são a implementação concreta.
* **Alinhamento**: OWASP (LLM/GenAI), CSA AI Controls Matrix (AICM), visão NIST-style de gestão de risco, mais controles de supply chain (SAST, SBOM, imagens corporativas, segredos, etc).

---

## 1) Arquitetura Conceitual (Z0–Z9)

![Diagram](./assets/Project_Diagram.drawio.svg)

**Ideia central**: *Todo tráfego entra por portas controladas (Z1, Z6). Somente modelos do **Prod Model Registry** podem ir para produção. Todo uso de LLM passa pelo **LLM/GenAI Security Gateway** — nunca direto no provider.*

---

## 2) Mapeamento Teoria ↔ Prática (o que você constrói)

| Zona   | Função (Teórica)                      | Artefatos práticos (Docs / Serviços)                                                      | Riscos principais                                              | Controles / Gates                                             |
| ------ | ------------------------------------- | ----------------------------------------------------------------------------------------- | -------------------------------------------------------------- | ------------------------------------------------------------- |
| [**Z0**](./docs/Z0.md) | Fontes externas, parceiras e internas | DAGs de ingestão, contratos de schema, catálogos de fonte                                 | Data poisoning, arquivos malformados                           | Allowlist, schema/MIME, validação de origem                   |
| [**Z1**](./docs/Z1/index.md) | Camada de **ingestão controlada**     | Reverse Proxy/API GW + WAF, AuthN/AuthZ, Content Validation, Anti-malware/CDR, ETL/MQ     | DoS, malware, exfiltração via upload                           | Rate limiting/throttling, limites de tamanho, CDR, sandbox    |
| [**Z2**](./docs/Z2/index.md) | Data lake bruto restrito              | Buckets MinIO criptografados, versionados e assinados                                     | Vazamento, alteração maliciosa                                 | Criptografia, políticas de bucket, isolamento de rede         |
| [**Z3**](./docs/Z3/index.md) | Dados curados & Feature Store         | Zonas curadas, feature store versionada                                                   | Drift silencioso, exposed PII                                  | Catálogo, classificação, políticas de acesso                  |
| [**Z4**](./docs/Z4/index.md) | Fábrica de modelos (dev/train)        | Workbench seguro, MLflow Tracking, SAST/DAST, SBOM, scan de imagens, checagem de datasets | Supply chain, segredos em código, uso de dados não autorizados | Vault Agent, imagens oficiais, policies-as-code, CI com gates |
| [**Z5**](./docs/Z5/index.md) | Registry & governança                 | MLflow Registry como origem de verdade, esteiras de aprovação                             | Modelo não aprovado em produção                                | Assinatura, revisão, LGPD/compliance, SCDR/ADR                |
| **Z6** | Serving & inferência                  | Inference Gateway, serviços online, LLM Gateway, batch, Scored Store                      | Prompt injection, vazamento de dados, abuso de API             | mTLS, AuthZ fina, filtros de saída, logs/audit, policies      |
| **Z7** | Consumidores                          | Core banking/risk/fraud, canais, chatbots internos                                        | Uso indevido, acoplamento frágil                               | Least privilege, contratos de API, versionamento              |
| **Z8** | Serviços de segurança compartilhados  | IAM/IdP, Vault, KMS/HSM, DLP, SIEM, catálogo                                              | Identidade fraca, má gestão de chaves                          | RBAC/ABAC, rotação, segregação de funções                     |
| **Z9** | Observabilidade & auditoria           | Logs centralizados, monitoração de drift/bias, trilhas de auditoria                       | Blind spots, fraude não detectada                              | Correlação, alertas, auditoria periódica                      |

---

## 3) Estrutura do Repositório (Docs 01–07)

Os documentos práticos implementam a arquitetura. A ordem sugerida é:

1. **01-Vault-Instalacao-Configuracao.md**
   Instalação do Vault como **autoridade de confiança**: KV v2, Transit, políticas por serviço, AppRole/JWT/OIDC, init/unseal seguro, auditoria, rotação.
2. **02-Core-Infraestrutura-e-Seguranca.md**
   Rede, volumes, Postgres, MinIO, Redis, etc. Exposição mínima (apenas proxy), TLS, criptografia em repouso, backups.
3. **03-Vault-Agent-Templates-e-Integracao.md**
   Padrão Vault Agent/sidecar, templates `.ctmpl`, injeção de segredos em runtime. Sem senha fixa em variável de ambiente/arquivo.
4. **04-Servicos-MLOps-(Airflow-MLflow-FastAPI).md**
   Airflow para orquestração, MLflow para tracking/registry, FastAPI para serving de modelos, tudo autenticado e integrado ao Vault.
5. **05-OIDC-e-Seguranca-Avancada.md**
   IdP/Keycloak, OIDC/OAuth2, RBAC, perfis de acesso, proteção de dashboards/admin UIs por proxy + IdP.
6. **06-CI-CD-e-GitHub-Actions-Vault-OIDC.md**
   Pipeline seguro: GitHub OIDC → Vault, scanners (código/dependência/imagem), geração de SBOM, gates de segurança, deploy automatizado.
7. **07-Hardening-e-Backup-Vault-(Opcional).md**
   Hardening avançado, políticas restritivas, audit devices, backups agendados, testes de restore, runbooks de incidente.

> Cada doc deve deixar explícito: **cenário**, **riscos**, **controles aplicados**, **referências OWASP/CSA/NIST** relacionados.

---

## 4) Fluxo Fim a Fim (Narrativa Resumida)

1. **Z0 → Z1**: qualquer dado entra apenas pelo **Ingestion Gateway**, é autenticado, validado (MIME/tamanho/schema), inspecionado (AV/CDR) e roteado para ETL/streaming.
2. **Z1 → Z2 → Z3**: dados brutos vão para a **Raw Zone** (criptografada/versionada), depois passam por processos de curadoria até compor a **Curated Zone** e a **Feature Store** governada.
3. **Z3 → Z4**: times de data science treinam modelos em ambiente isolado, com datasets autorizados, segredos via Vault e rastreabilidade completa (**proveniência**).
4. **Z4 → Z5**: modelos candidatos passam por avaliação funcional, de performance, segurança e risco (inclusive LGPD/ética). Só então são aprovados, assinados e registrados no **Prod Model Registry**.
5. **Z5 → Z6**: inferência online só acontece via **Inference Gateway**; LLMs só via **LLM/GenAI Security Gateway**. Jobs batch usam modelos do registry e escrevem em **Scored Output Store** governada.
6. **Z7, Z8, Z9**: sistemas de negócio consomem apenas saídas governadas. IAM, Vault, KMS, DLP, SIEM e trilhas de auditoria garantem identidade forte, minimização de privilégios, monitoração, detecção de anomalias, drift e abusos.

---

## 5) Referenciais de Segurança & Governança

Use o lab para experimentar **como** aplicar boas práticas na prática:

* **OWASP Top 10 para LLM / OWASP GenAI Security**
  Proteção contra prompt injection, exfiltração, insegurança em plugins, supply chain, etc.
* **CSA AI Controls Matrix (AICM)**
  Guia para requisitos de segurança, privacidade, compliance e operação de plataformas de IA.
* **Abordagem NIST-style (risk-based)**
  Conectar requisitos de negócio, riscos, controles, evidências e auditoria contínua.

Em cada componente, documente claramente:

* Qual risco está sendo mitigado.
* Quais controles foram implementados.
* Como isso se alinha aos frameworks acima.

---

## 6) Quickstart (VPS — Visão de Alto Nível)

1. Criar rede, reverse proxy (Traefik/Nginx) + HTTPS + security headers + rate limiting.
2. Subir o **Vault** como autoridade de segredos (KV, Transit, AppRole/JWT/OIDC + audit log).
3. Provisionar Postgres, MinIO, Redis e demais serviços **apenas em rede interna**.
4. Integrar todos os serviços ao Vault via Vault Agent / templates.
5. Subir **Airflow + MLflow + FastAPI** usando segredos do Vault.
6. Configurar **IdP/OIDC** (Keycloak, etc.) para UIs de gestão e serviços.
7. Configurar **CI/CD seguro** com GitHub Actions + OIDC + scanners + SBOM + gates.
8. Implementar **logging central**, métricas, alertas, monitoração de drift/bias e **backup/restore testado**.

---

## 7) Contribuição & Próximos Passos

* Adicionar cenários de **red teaming para LLMs** (jailbreak, data exfil, prompt injection).
* Ampliar exemplos de **Feature Store** com classificação de PII, finalidade e policies.
* Automatizar geração de **SCDR/ADR** após aprovação de modelos (Z5).
* Publicar runbooks de incidente (vazamento de segredos, modelo comprometido, abuso de API).
* Incluir exemplos de integrações com provedores externos de LLM, sempre via LLM Gateway.
