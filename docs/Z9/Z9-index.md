# Z9 — Monitoring, Observability & Audit

*(Logs estruturados • Métricas & SLOs • Tracing distribuído • Monitoração de dados/modelos • Custos (FinOps/LLMOps) • Auditoria & Evidências • Alertas & Runbooks)*

![Diagram](../../assets/Project_Diagram-Z9.drawio.svg)

> A Z9 centraliza **telemetria operacional e de segurança** (logs, métricas, traces) de todas as zonas para **detectar anomalias cedo**, **correlacionar eventos ponta-a-ponta** (Z0→Z7) e **produzir trilhas de auditoria e evidências** para compliance e resposta a incidentes. Ela recebe eventos das zonas de ingestão (Z1), dados (Z2/Z3), serving (Z6) e consumo (Z7), conforme já indicado nesses documentos.     

---

## 1) Papel da Z9 no MLOps Security Lab

**Objetivo central:** transformar o pipeline inteiro em um **sensor confiável** — com **visibilidade**, **detecção**, **alerta** e **auditoria reprodutível** — apoiando decisões técnicas/negociais e requisitos regulatórios.

**Resultados-chave:**

* **Coleta padronizada** (OpenTelemetry) de **logs estruturados, métricas e traces** de Z1, Z2, Z3, Z6 e Z7. Z1 e Z7 já definem envio de eventos/telemetria para Z9; Z2 e Z3 também citam integração para auditoria e qualidade/drift.    
* **Dashboards & SLOs** por domínio: ingestão (spikes, 4xx/5xx, AV/CDR), dados (qualidade/distribuição), serving (p95/p99, erro), consumo (latência, **custo** e *hit-ratio*). Z7 já antecipa métricas de p95/p99, erro e custo reportadas para Z9. 
* **Trilhas de auditoria** e **evidence links** para relacionar chamadas, modelos/versões e decisões (alinhado a Z6 e Z7, que exigem auditoria end-to-end).  

---

## 2) Componentes da Z9

> Cada item pode ter um arquivo “**Z9-2.x**” de detalhamento, no mesmo padrão dos índices de Z6/Z7.

### 2.1 Logging Estruturado & Correlação — [Detalhamento](./Z9-2.1.md)

* **Formato único** (JSON) com **campos canônicos**: `timestamp`, `request_id/trace_id`, `identity/tenant`, `rota/endpoint`, `model_id@versão`, `dataset/ref`, `resultado` (aceito/negado/erro), **sem PII em payload**.
* **Correlação ponta-a-ponta**: o `trace_id` nasce no gateway (Z1/Z6) e aparece nas camadas de dados (Z2/Z3) e nos consumidores (Z7).   

### 2.2 Métricas, SLOs & Painéis — [Detalhamento](./Z9-2.2.md)

* **Métricas de ingestão (Z1)**: taxa, p95/p99, 4xx/5xx, limites (429/413), contagem de AV/CDR. 
* **Métricas de serving (Z6)**: p95/p99 por rota/modelo, taxa de erro, throughput, *resource usage*. 
* **Métricas de consumo (Z7)**: p95/p99 por rota/tenant, **custo** e *hit-ratio* de rotas como `/last`. 
* **SLOs & Error Budgets** com alertas (burn rate).

### 2.3 Tracing Distribuído (OpenTelemetry) — [Detalhamento](./Z9-2.3.md)

* **Spans encadeados**: Z1 → Z6 → Z7, com anotações (modelo/versão, *policy result*, *feature hits*).
* **Acompanhamento de decisões** (score → decisão → efeito) para auditoria.  

### 2.4 Monitoração de Ingestão & Upload Safety — [Detalhamento](./Z9-2.4.md)

* **Indicadores de borda (Z1)**: bloqueios WAF, **anti-malware/CDR**, *rate limiting*, tamanhos de payload, identidades mais ativas, fontes com maior rejeição. 

### 2.5 Monitoração de Dados (Z2/Z3): Qualidade, Integridade & Drift — [Detalhamento](./Z9-2.5.md)

* **Eventos de Z2**: leitura/escrita/DELETE/LIST com quem, onde e quando (auditoria). 
* **Eventos de Z3**: **queda de qualidade**, **mudança de distribuição** e **acesso a features sensíveis** → alarmes. 

### 2.6 Model Monitoring (Z6): Performance & Safety — [Detalhamento](./Z9-2.6.md)

* **Clássicos**: *data drift*, *concept drift* (com labels tardios), *threshold hits*, calibração, *out-of-range* de features.
* **LLM/GenAI**: *prompt-injection flags*, *output redaction hits*, **custo por chamada/token**, *refusal rate*. (Z6 já define gateways e políticas para isso.) 

### 2.7 Consumo & FinOps/LLMOps (Z7) — [Detalhamento](./Z9-2.7.md)

* **KPIs de consumo**: taxa de erro, latência, **custo por rota/tenant**, *hit-ratio*, cobertura de decisões.
* **Detecções**: **leitura fora de contrato** (Z7 exige consumo **via APIs** e **não** direto do bucket), *scraping*, abuso. 

### 2.8 Auditoria & Evidências — [Detalhamento](./Z9-2.8.md)

* **Trilhas imutáveis** ligando **request → modelo/versão → decisão → escrita/leitura**; **evidence links** (artefatos, *attestations*, aprovações de Z5/Z6/Z7).  

### 2.9 Alerting, Runbooks & Post-mortems — [Detalhamento](./Z9-2.9.md)

* **Severidade e canais** (pager vs. ticket), **runbooks versionados**, *blameless post-mortems* com *action items* rastreáveis.

### 2.10 Telemetria com Privacidade — [Detalhamento](./Z9-2.10.md)

* **Redação/masking** (sem PII nos logs), **retenção** e **minimização** de campos; amostragem e *sampling* diferenciados por risco.

---

## 3) Riscos × Controles × Frameworks (Z9)

| Risco transversal                                   | Controles em Z9                                                                                          | Referências nos docs                                |
| --------------------------------------------------- | -------------------------------------------------------------------------------------------------------- | --------------------------------------------------- |
| **Invisibilidade** (sem logs/traces/métricas úteis) | 2.1/2.2/2.3 **OTel + logs estruturados + SLOs + tracing**; painéis por zona                              | Z6, Z7 citam auditoria/telemetria mandando p/ Z9.   |
| **Falhas não detectadas** (drift/poisoning/erros)   | 2.5/2.6 **monitoração de dados e modelos**, alarmes de *drift* e *threshold hits*                        | Z3 integra qualidade/drift com Z9.                  |
| **Vazamento via logs/telemetria**                   | 2.10 **redaction/masking**, retenção e minimização, revisão periódica                                    | Z7 já reforça logs sem PII.                         |
| **Bypass de gateway / shadow ingress**              | 2.4 indicadores de borda, correlação com acessos Z2/Z3/Z6/Z7, alertas de caminhos não autorizados        | Z1/Z2/Z7 impõem consumo/ingestão governados.        |
| **Auditoria insuficiente p/ compliance**            | 2.8 **trilhas imutáveis** + evidências ligadas a decisões e versões; 2.9 *post-mortems* e *action items* | Z6/Z7 exigem trilhas end-to-end.                    |

---

## 4) Teoria ↔ Prática (o que você constrói no Lab)

* **Coletores OTel + stack de observabilidade** (ex.: Prometheus/Grafana para métricas; Loki/ELK para logs; Jaeger/Tempo para traces).
* **Pipelines de ingestão de eventos** de:
  **Z1** (WAF/AV/CDR/limites), **Z2** (audit de S3/MinIO), **Z3** (qualidade/drift), **Z6** (gateway/serving), **Z7** (métricas de consumo e custo).     
* **Dashboards prontos**: Ingestão Segura; Qualidade & Drift; Serving SLOs; Consumo & Custos; Auditoria & Evidências.
* **Alertas**:
  *DoS/abuso* (spike 429/413), *AV/CDR hits*, *p99 acima de X*, *erro > Y%*, *drift > limiar*, *leitura fora de contrato*, *padrões suspeitos de acesso*.

---

## 5) Fluxos Resumidos

### 5.1 Trace de inferência on-line

1. **Z6-Gateway** cria `trace_id` e loga rota/modelo/versão.
2. Serviço de inferência anota latência/resultado.
3. Consumidor (Z7) anota decisão e **custo** (LLM).
4. Tudo converge na Z9 (logs/métricas/traces) com visualização por `trace_id`.  

### 5.2 Evento de ingestão → dados

1. Z1 bloqueia/aceita upload; registra WAF/AV/CDR e identidade.
2. Se aceito, Z2 grava objeto e **audita** (quem/onde/quando).
3. Alarmes em Z9 para anomalias (tamanho, taxa, fontes com rejeição).  

### 5.3 Qualidade/drift de features (Z3)

1. Pipeline em Z3 detecta drift/queda de qualidade.
2. Evento chega à Z9 e abre alerta; dashboards mostram impacto esperado. 

---

## 6) Métricas & KPIs sugeridos

* **Ingestão (Z1)**: req/s, p95/p99, 4xx/5xx por rota, **AV/CDR hits**, payload médio, **429/413**. 
* **Dados (Z2/Z3)**: volume/dia, % falhas de validação, *data drift score*, *schema violations*, operações `PUT/GET/DELETE/LIST` por identidade.  
* **Serving (Z6)**: p95/p99, taxa de erro, *resource usage*, *startup verification passes*. 
* **Consumo (Z7)**: p95/p99 por rota/tenant, **custo por decisão**, *hit-ratio*, cobertura. 

---

## 7) Catálogo de Alertas (exemplos)

* **Abuso de API (Z1)**: aumento de 5× em 10min + 429/413 → *page* N2. 
* **Malware/CDR**: `infected/suspicious` ≥ N em 1h → *ticket* + bloqueio de fonte. 
* **Data Drift (Z3)**: PSI > limiar por 30min → *page* dados + *ticket* ML. 
* **Serving SLO (Z6)**: p99 > alvo por 15min ou erro > Y% → *page* on-call. 
* **Consumo fora de contrato (Z7)**: rota não permitida ou padrão de scraping → *ticket* + bloqueio. 

---

## 8) Checklists Operacionais

**Antes do go-live**

* [ ] Campos canônicos de log (JSON) padronizados em todos os serviços.
* [ ] Dashboards por domínio (Ingestão, Dados, Serving, Consumo, Auditoria).
* [ ] SLOs + alertas (lentas/erros/drift/custos) implantados.
* [ ] Trilhas de auditoria ligando **request→modelo→decisão→persistência**. 

**Durante a operação**

* [ ] Revisão mensal de *alert fatigue* e ajustes de limiares.
* [ ] *Post-mortems* com *action items* rastreados.
* [ ] Auditoria de acessos a dados/modelos e **access reviews** de consumo. 

---

## 9) Frases prontas (entrevista)

* “**Z9 unifica logs, métricas e traces** com **correlação por `trace_id`**, permitindo auditar cada decisão de modelo até a camada de dados/consumo.”
* “Exponho **SLOs por rota/modelo/tenant** e **alertas de drift e custo** para reduzir risco técnico e financeiro.”
* “**Nada de PII em logs**; telemetria segue **redaction/minimização** e retenção adequada.”

---

### Navegação rápida (âncoras úteis)

* **Z1 → Z9** (auditoria & observabilidade): ver “2.8 Auditoria & Observabilidade” em Z1. 
* **Z2 → Z9** (logs de acesso): ver “2.8 Logging de Acesso & Integração com Z9” em Z2. 
* **Z3 → Z9** (qualidade & drift): ver integração em Z3. 
* **Z6/Z7 → Z9** (SLOs, auditoria de consumo): ver introduções e componentes em Z6/Z7.  
