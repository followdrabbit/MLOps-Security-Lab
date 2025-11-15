# 🔍 Automatizando revisão de Pull Requests com Danger JS

Este documento descreve como configurar o **Danger JS** no repositório `MLOps-Security-Lab` para fazer **revisão automática de PRs**, reforçando o hardening do fluxo de contribuição.

O **Danger** não substitui revisão humana, mas automatiza os “comentários repetitivos”, liberando tempo para analisar arquitetura, segurança e design de forma mais profunda. :contentReference[oaicite:0]{index=0}  

---

## 1. O que é o Danger JS?

- **O que é**  
  É uma ferramenta que roda durante o **CI** e executa regras definidas em um arquivo `dangerfile.(js|ts)`. Com base nessas regras, ele:
  - comenta automaticamente no PR;
  - gera *warnings* ou *fails* que podem **bloquear o merge**;
  - aplica convenções de time (tamanho máximo de PR, obrigatoriedade de descrição, etc.). :contentReference[oaicite:1]{index=1}  

- **Por que usar no MLOps-Security-Lab**
  - Padronizar PRs (descrição mínima, links para issues, etc.).
  - Incentivar **pequenos PRs** e boa documentação.
  - Garantir que mudanças sensíveis (arquitetura, segurança, docs) sejam tratadas com mais cuidado.
  - Transformar “boas práticas” em **regras automáticas**.

---

## 2. Visão geral da solução

Fluxo desejado:

1. Alguém abre ou atualiza um **Pull Request**.
2. O GitHub dispara o workflow `.github/workflows/danger.yml`.
3. O workflow executa o **Danger JS** (via GitHub Action).
4. O Danger lê o `dangerfile.js` e:
   - adiciona comentários no PR;
   - cria *checks* de sucesso/falha.
5. Opcional: a branch `main` é configurada para **exigir o check do Danger** antes de permitir merge.

---

## 3. Passo a passo de configuração

### 3.1. Criar o Dangerfile

1. Na raiz do repositório, crie o arquivo:

`dangerfile.js`

2. Exemplo de regras iniciais adaptadas para o `MLOps-Security-Lab`:

```js
// dangerfile.js
// Regras básicas de revisão automática para o MLOps-Security-Lab

// Importa helpers do Danger
import { danger, warn, fail, message } from "danger";

// Atalho para o PR
const pr = danger.github.pr;

// -----------------------------
// 1) Bloquear PRs marcados como WIP
// -----------------------------
const prTitle = (pr.title || "").toLowerCase();

if (prTitle.includes("wip") || prTitle.includes("work in progress")) {
  fail(
    "Este PR está marcado como WIP. Remova `WIP` do título antes de mesclar."
  );
}

// -----------------------------
// 2) Exigir descrição mínima do PR
// -----------------------------
const prBody = pr.body || "";

if (prBody.length < 20) {
  warn(
    "A descrição deste PR é muito curta. Explique o contexto, o objetivo e o impacto das mudanças (mínimo ~20 caracteres)."
  );
}

// -----------------------------
// 3) Alertar PRs muito grandes
// -----------------------------
const additions = pr.additions || 0;
const deletions = pr.deletions || 0;
const totalChanges = additions + deletions;
const bigPRThreshold = 500;

if (totalChanges > bigPRThreshold) {
  warn(
    `Este PR é grande (${totalChanges} linhas alteradas). Considere quebrar em PRs menores para facilitar a revisão.`
  );
}

// -----------------------------
// 4) Lembrar de atualizar documentação
//    quando arquitetura / security forem alterados
// -----------------------------
const modifiedFiles = [
  ...danger.git.modified_files,
  ...danger.git.created_files,
];

// Regra simples: se mexeu em arquivos de arquitetura/segurança,
// mas não mexeu em docs/, avisar.
const touchedArchitecture = modifiedFiles.some((path) =>
  path.match(/architecture|Security\.md|threat-model\.md|Lab-vs-Prod\.md/i)
);

const touchedDocs = modifiedFiles.some((path) => path.startsWith("docs/"));

if (touchedArchitecture && !touchedDocs) {
  message(
    "Você alterou arquivos de arquitetura/segurança, mas não atualizou nada em `docs/`. Verifique se a documentação precisa ser ajustada."
  );
}

// -----------------------------
// 5) Garantir que haja responsável pelo PR
// -----------------------------
if (pr.assignee === null) {
  warn(
    "Este PR não possui ninguém atribuído. Atribua um responsável (assignee) ou um revisor para garantir que será revisado."
  );
}
````

> 💡 **Ideia:** com o tempo, você pode ir adicionando regras específicas (ex.: exigir link para issue, check de pastas críticas, etc.). ([TabNews][1])

---

### 3.2. Criar o workflow do Danger no GitHub Actions

Vamos usar o **Danger JS como GitHub Action oficial**, sem precisar adicionar o pacote ao `package.json`. ([danger.systems][2])

Crie o arquivo:

`.github/workflows/danger.yml`

Com o conteúdo:

```yaml
name: 🔍 Danger PR Review

on:
  pull_request:
    branches:
      - main
      - develop
      - feature/**
      - fix/**

permissions:
  contents: read
  pull-requests: write
  checks: write
  statuses: write

jobs:
  danger:
    name: Run Danger JS
    runs-on: ubuntu-latest

    steps:
      - name: Checkout código
        uses: actions/checkout@v4

      - name: Executar Danger JS
        uses: danger/danger-js@11.2.6
        with:
          args: "--dangerfile ./dangerfile.js"
        env:
          # Usa o token padrão do GitHub Actions
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

**O que este workflow faz**

* Roda **somente em eventos `pull_request`** (não em push).
* Faz checkout do repositório.
* Executa a ação `danger/danger-js` apontando para `./dangerfile.js`. ([danger.systems][2])
* Usa o `secrets.GITHUB_TOKEN` (token automático do GitHub Actions) para:

  * comentar no PR;
  * criar checks/statuses na UI do PR.

---

### 3.3. Configurar o token e considerações de segurança

* Por padrão, o GitHub já cria o `GITHUB_TOKEN` para cada workflow, com permissões limitadas ao repositório. ([danger.systems][2])
* No bloco `permissions:` do workflow nós:

  * Restringimos o acesso a apenas o necessário:

    * `contents: read` → ler arquivos do repo.
    * `pull-requests: write` → comentar no PR.
    * `checks: write` e `statuses: write` → atualizar o status da *check*.

#### Repositório privado (caso atual mais provável)

* Usar apenas `GITHUB_TOKEN` é suficiente.
* O Danger vai comentar como usuário **`github-actions`**, o que é adequado para um bot.

#### Repositório público com contribuições de forks (opcional)

Em repositórios open source, o `GITHUB_TOKEN` possui restrições em PRs vindos de forks. Nesse caso:

1. Criar um usuário bot no GitHub (ex.: `mlops-security-lab-bot`).
2. Gerar um **Personal Access Token (PAT)** com escopo mínimo (`public_repo` para OSS). ([toni-develops.com][3])
3. Adicionar este token como secret do repositório, por exemplo `DANGER_GITHUB_API_TOKEN`.
4. Ajustar o workflow:

```yaml
      - name: Executar Danger JS
        uses: danger/danger-js@11.2.6
        with:
          args: "--dangerfile ./dangerfile.js"
        env:
          DANGER_GITHUB_API_TOKEN: ${{ secrets.DANGER_GITHUB_API_TOKEN }}
```

---

### 3.4. Tornar o check do Danger obrigatório (branch protection)

Para que o Danger faça parte do **hardening**:

1. Vá em **Settings → Branches → Branch protection rules**.
2. Edite (ou crie) a regra da branch `main`.
3. Em **“Require status checks to pass before merging”**:

   * marque o check que o workflow do Danger cria (ex.: `🔍 Danger PR Review / danger`).
4. (Opcional) Marque **“Include administrators”** se quiser exigir mesmo para quem tem admin no repo.

Assim, **PRs que violarem regras críticas (usando `fail`) não poderão ser mesclados.**

---

## 4. Regras sugeridas para o MLOps-Security-Lab

Abaixo, um resumo das regras implementadas no `dangerfile.js` de exemplo e como elas ajudam no hardening do repositório:

1. **Bloqueio de PRs WIP**

   * **O que**: Falha se o título contiver `WIP` ou `Work in progress`.
   * **Por que**: Evita merges acidentais de PRs incompletos.
   * **Como ajustar**: Trocar `fail(...)` por `warn(...)` se quiser só um aviso.

2. **Descrição mínima do PR**

   * **O que**: Emite *warning* se a descrição tiver menos de ~20 caracteres.
   * **Por que**: Força o autor a registrar contexto, objetivo e impacto.
   * **Possível evolução**:

     * Exigir padrão: ex.: se não tiver “Contexto:”, “Mudanças:”, “Riscos:”, avisar.

3. **Alerta para PR grande**

   * **O que**: *Warning* se `additions + deletions > 500`.
   * **Por que**: PRs grandes são difíceis de revisar e mais propensos a bugs.
   * **Possível evolução**:

     * Separar limites diferentes por pasta (ex.: mais rígido em `infra/`).

4. **Lembrete de documentação**

   * **O que**: Se arquivos de arquitetura/segurança forem alterados
     (`architecture*`, `Security.md`, `threat-model.md`, `Lab-vs-Prod.md`)
     mas nenhum arquivo em `docs/` foi modificado, o Danger deixa uma mensagem.
   * **Por que**: Garante alinhamento entre código/arquitetura e documentação.
   * **Possível evolução**:

     * Exigir *fail* ao invés de *message* se quiser tornar obrigatório.

5. **Responsável pelo PR**

   * **O que**: *Warning* se o PR não tiver `assignee`.
   * **Por que**: Ajuda a evitar PRs “órfãos” sem dono claro.
   * **Possível evolução**:

     * Transformar em `fail` para bloquear PR sem responsável.

---

## 5. Extensões futuras

Algumas ideias de regras adicionais que fazem sentido num lab focado em MLOps + Segurança:

* **Verificar se há link para issue/ticket**

  * Exigir que o corpo do PR contenha algo como `Refs:` ou `Issue:`.

* **Exigir atualização de changelog / release notes**

  * Se mexeu em `infra/`, `scripts/` ou `pipelines/`, exigir alteração em um arquivo de log de mudanças.

* **Bloquear arquivos sensíveis**

  * *Fail* se um PR tentar adicionar arquivos como:

    * `*.pem`, `*.key`, `id_rsa`, `.env`, etc.
  * Isso ajuda a evitar vazamento de segredo acidental via PR.

* **Regrar testes** (para quando houver testes automatizados):

  * Se arquivos de código forem alterados, exigir alteração em pastas de testes.

Essas regras podem ser implementadas incrementalmente no `dangerfile.js`, à medida que o fluxo do time amadurecer.

---

## 6. Checklist rápido de implementação

1. [ ] Criar `dangerfile.js` na raiz do repo com as regras básicas.
2. [ ] Criar `.github/workflows/danger.yml` usando `danger/danger-js`.
3. [ ] Confirmar que o `GITHUB_TOKEN` está disponível (padrão do GitHub Actions).
4. [ ] Abrir um PR de teste e verificar se o Danger comenta e cria o check.
5. [ ] Configurar **branch protection** para exigir o check do Danger na branch `main`.
6. [ ] Ajustar regras do `dangerfile.js` conforme o estilo de contribuição do projeto.

---

Com isso, o `MLOps-Security-Lab` passa a ter uma camada a mais de **governança e segurança no fluxo de PRs**, alinhada com o objetivo geral de hardening do repositório.
