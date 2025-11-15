# 🔐 Segurança em GitHub Actions — `MLOps-Security-Lab`

## 1. Objetivo

Este documento descreve como o repositório `MLOps-Security-Lab` utiliza **GitHub Actions** com foco em:

* **Segurança** (mínimo privilégio, proteção de segredos, isolamento de jobs)
* **Qualidade** (lint, testes, análise estática)
* **Governança** (padronização, rastreabilidade e previsibilidade do pipeline)

Ele complementa:

* `docs/hardening/github-hardening-mlops-security-lab.md`
* Demais documentos de hardening (AWS, Vault, etc.)

---

## 2. Arquivo de Workflow: `.github/workflows/ci.yml`

O workflow principal de CI do projeto está definido em:

```text
.github/workflows/ci.yml
```

Ele é responsável por:

* Validar mudanças em **push** e **pull requests**
* Executar checagens de qualidade e segurança em código Python

### 2.1 Gatilhos (`on`)

Trecho relevante:

```yaml
on:
  push:
    branches: [ "main", "dev" ]
  pull_request:
    branches: [ "main", "dev" ]
```

**O que faz**

* Dispara o CI em:

  * `push` para as branches `main` e `dev`
  * `pull_request` direcionados para `main` e `dev`

**Por que é importante**

* Garante que:

  * Tudo que entra em `main` ou `dev` passou pela mesma validação
  * PRs sejam checados **antes** de serem mesclados

---

### 2.2 Permissões do `GITHUB_TOKEN` (mínimo privilégio)

```yaml
permissions:
  contents: read
```

**O que faz**

* Restringe o `GITHUB_TOKEN` do workflow para ter apenas **permissão de leitura** no conteúdo do repositório.

**Por que é importante**

* Segue o princípio de **mínimo privilégio**:

  * Se um job ou dependência for comprometido, o impacto fica limitado.
  * Evita que o pipeline modifique o repo (ex.: criar tags, commits ou PRs) sem necessidade.

---

### 2.3 Controle de Concorrência

```yaml
concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true
```

**O que faz**

* Garante que:

  * Só exista **um** workflow de CI rodando por branch (`github.ref`) ao mesmo tempo.
  * Se um novo commit for enviado, o workflow anterior é **cancelado**.

**Por que é importante**

* Evita:

  * Consumo desnecessário de recursos
  * Resultados desatualizados (tests passando em commit antigo enquanto há um novo commit que quebrou algo)

---

## 3. Job Principal: `python-ci`

```yaml
jobs:
  python-ci:
    name: Lint, Testes e Checagens de Segurança (Python)
    runs-on: ubuntu-latest
    timeout-minutes: 30
```

**O que faz**

* Define um job único, chamado **“Lint, Testes e Checagens de Segurança (Python)”**.
* Executa em `ubuntu-latest`.
* Tem timeout de 30 minutos para evitar jobs travados.

**Por que é importante**

* Dá previsibilidade:

  * Mesmo que algo trave (ex.: deadlock em teste), o job expira.
* Facilita leitura e debug, já que é um job único e bem nomeado.

---

## 4. Detalhamento das Etapas (Steps)

### 4.1 Checkout do Código

```yaml
- name: Checkout do código
  uses: actions/checkout@v4
```

**O que faz**

* Faz checkout do repositório na VM do runner.

**Por que é importante**

* É o ponto de partida: todas as análises e testes são executados sobre esse código.

**Cuidados de segurança**

* `actions/checkout@v4` é a versão recomendada.
* Permite ajustes futuros, como **fetch-depth** reduzido para minimizar histórico baixado.

---

### 4.2 Configuração do Python

```yaml
- name: Configurar Python
  uses: actions/setup-python@v5
  with:
    python-version: "3.11"
```

**O que faz**

* Instala/configura o Python 3.11 no runner.

**Por que é importante**

* Garante consistência entre:

  * Ambiente local (dev)
  * Ambiente de CI
* Facilita debugging e previsibilidade de versões.

---

### 4.3 Cache de Dependências (`pip`)

```yaml
- name: Cache do pip
  uses: actions/cache@v4
  with:
    path: ~/.cache/pip
    key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements*.txt') }}
    restore-keys: |
      ${{ runner.os }}-pip-
```

**O que faz**

* Cria cache para pacotes instalados via `pip`.

**Por que é importante**

* Reduz tempo de execução do workflow.
* Menos chamadas externas → ligeira redução da superfície de ataque (menos exposição a falhas intermitentes de rede e mirrors).

---

### 4.4 Atualização do `pip`

```yaml
- name: Atualizar pip
  run: pip install --upgrade pip
```

**O que faz**

* Atualiza o `pip` para a versão mais recente.

**Por que é importante**

* Mitiga problemas de compatibilidade
* Inclui correções de bugs e possíveis melhorias de segurança na própria ferramenta de instalação.

---

### 4.5 Instalação de Dependências Principais

```yaml
- name: Instalar dependências principais
  if: hashFiles('requirements.txt') != ''
  run: |
    echo "Instalando dependências de requirements.txt"
    pip install -r requirements.txt
```

**O que faz**

* Se existir `requirements.txt`, instala as dependências do projeto.

**Por que é importante**

* Reproduz, no CI, o ambiente necessário para:

  * Rodar testes
  * Executar ferramentas de lint e segurança que dependam do projeto

**Nota**

* O uso de `if: hashFiles(...) != ''` evita falha caso `requirements.txt` ainda não exista (fase inicial do projeto).

---

### 4.6 Instalação de Dependências de Desenvolvimento

```yaml
- name: Instalar dependências de desenvolvimento
  if: hashFiles('requirements-dev.txt') != ''
  run: |
    echo "Instalando dependências de desenvolvimento"
    pip install -r requirements-dev.txt
```

**O que faz**

* Instala bibliotecas adicionais usadas apenas em desenvolvimento (lint extras, ferramentas específicas, etc.).

**Por que é importante**

* Mantém separação entre:

  * Dependências de **runtime**
  * Dependências de **desenvolvimento/qualidade**

---

### 4.7 Instalar Ferramentas de Qualidade e Segurança

```yaml
- name: Instalar ferramentas de qualidade e segurança
  run: |
    pip install ruff pytest bandit pip-audit
```

**O que faz**

* Instala:

  * `ruff` → lint (estilo e possíveis erros)
  * `pytest` → framework de testes
  * `bandit` → análise estática de segurança de código Python
  * `pip-audit` → auditoria de vulnerabilidades em dependências

**Por que é importante**

* Centraliza ferramentas críticas de qualidade e segurança.
* Facilita padronização (quem roda localmente usa as mesmas ferramentas).

---

### 4.8 Lint (Ruff)

```yaml
- name: Lint (ruff)
  run: |
    echo "Executando lint com ruff..."
    ruff check .
```

**O que faz**

* Analisa o código em busca de:

  * Más práticas
  * Problemas de estilo
  * Alguns tipos de erros lógicos

**Por que é importante**

* Melhora legibilidade
* Evita bugs simples
* Facilita manutenção e revisões

---

### 4.9 Testes com Pytest

```yaml
- name: Testes (pytest)
  if: hashFiles('tests/**/*.py') != ''
  run: |
    echo "Executando testes com pytest..."
    pytest -q
```

**O que faz**

* Se existir ao menos um arquivo `tests/*.py`, executa a suíte de testes.

**Por que é importante**

* Garante que mudanças não quebrem comportamento existente.
* Apoia a criação de uma base de testes contínua à medida que o lab cresce.

---

### 4.10 Análise Estática de Segurança com Bandit

```yaml
- name: Análise estática de segurança (bandit)
  run: |
    echo "Executando Bandit..."
    bandit -r . -ll
```

**O que faz**

* Roda o `bandit` recursivamente no repo.
* Nível `-ll` gera mais detalhes.

**O que procura**

* Padrões perigosos, como:

  * Uso inseguro de `eval`, `exec`
  * Hardcoded passwords
  * Uso inseguro de módulos criptográficos
  * Entre outros

**Por que é importante**

* Cria um **gate de segurança** no pipeline:

  * Ajuda a impedir que código com problemas óbvios de segurança seja mesclado em `main`.

---

### 4.11 Auditoria de Dependências com `pip-audit`

```yaml
- name: Auditoria de dependências (pip-audit)
  if: hashFiles('requirements.txt') != ''
  run: |
    echo "Executando pip-audit em requirements.txt..."
    pip-audit -r requirements.txt
```

**O que faz**

* Analisa as dependências listadas em `requirements.txt`:

  * Compara versões instaladas com bases de dados de vulnerabilidades conhecidas (como PyPI advisories, etc.).

**Por que é importante**

* Endereça riscos de **supply chain**:

  * Não basta o seu código ser seguro; as bibliotecas que você usa também precisam estar atualizadas e sem CVEs relevantes.

---

## 5. Boas Práticas Complementares

Estas práticas podem ser adotadas em conjunto com esse workflow:

1. **Integrar o CI com a proteção da branch `main`**

   * Em **Branch protection rules**, marcar:

     * “Require status checks to pass before merging”
     * Selecionar este workflow `CI`.

2. **Usar Environments (`lab`, `prod`) para workflows de deploy**

   * Este `ci.yml` foca em qualidade e segurança de código.
   * Workflows de deploy podem usar:

     * `permissions` ainda mais restritivas
     * OIDC para acesso à AWS
     * Aprovação manual antes de deploy em ambiente crítico.

3. **Tratar falhas do Bandit/pip-audit como parte do fluxo de revisão**

   * Falha no job deve **bloquear merge**.
   * Comentários na PR devem explicar decisões de aceitar ou corrigir findings.

4. **Padronizar execução local**

   * Criar scripts `make` ou `poetry`/`tox` que rodem:

     * `ruff check .`
     * `pytest`
     * `bandit`
     * `pip-audit`
   * Assim, o dev consegue reproduzir o que o CI faz antes de abrir um PR.

---

## 6. Checklist de Segurança para Workflows

Use este checklist quando criar/alterar workflows:

```markdown
- [ ] Defini os `on:` (gatilhos) de forma restrita e previsível
- [ ] Configurei `permissions:` com o mínimo necessário
- [ ] Evitei usar secrets onde não é estritamente necessário
- [ ] Não armazeno credenciais fixas de cloud (uso OIDC sempre que possível)
- [ ] Tenho etapas de lint e testes no CI
- [ ] Tenho ao menos uma etapa de análise estática de segurança (ex.: Bandit)
- [ ] Tenho auditoria de dependências (ex.: pip-audit, Dependabot)
- [ ] Integrei o workflow com branch protection (status checks obrigatórios)
- [ ] Documentei o propósito e escopo do workflow (neste arquivo)
```
