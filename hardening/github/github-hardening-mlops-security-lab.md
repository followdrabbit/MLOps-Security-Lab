# 🔐 Guia de Hardening no GitHub — `MLOps-Security-Lab`

## 1. Objetivo e Escopo

Este documento define como **endurecer (hardening)** o repositório GitHub `MLOps-Security-Lab` para reduzir riscos como:

* Vazamento de segredos (tokens, chaves, senhas)
* Inclusão de código malicioso ou de baixa qualidade
* Problemas de supply chain via dependências vulneráveis
* Uso indevido de GitHub Actions e do `GITHUB_TOKEN`
* Alterações acidentais ou não autorizadas em branches críticos

Cada controle é descrito com:

* **O que**: o que configurar
* **Por que**: motivação de segurança
* **Como**: passos práticos para o `MLOps-Security-Lab`

---

## 2. Pré-requisito: Higiene da Conta e Organização

> Esses itens são fora do repo, mas são a base da segurança.

### 2.1 Habilitar MFA no GitHub

**O que**
Ativar **autenticação em dois fatores (MFA)** na sua conta GitHub (e na organização, se estiver usando uma).

**Por que**
Se alguém descobre sua senha, pode:

* Fazer push de código malicioso
* Roubar segredos
* Apagar ou corromper o repo

MFA reduz muito esse risco.

**Como**

1. No GitHub, vá em **Settings → Password and authentication**.
2. Ative **Two-factor authentication** (app TOTP ou chave física são preferíveis).
3. Se você usar uma **Organization**, nas configurações da org ative:

   * **Require two-factor authentication for all members**, se disponível.

---

## 3. Visibilidade do Repositório e Configurações Básicas

### 3.1 Definir se o Repo é Público ou Privado

**O que**
Definir se `MLOps-Security-Lab` será **público** ou **privado**.

**Por que**

* **Público**: bom pra portfólio/comunidade, mas qualquer segredo vazado é explorável imediatamente.
* **Privado**: reduz exposição, mas não elimina o risco (segredo no repo ainda é problema, só tem menos olhos).

**Como**

1. No repo, vá em **Settings → General → Danger Zone**.
2. Use **Change repository visibility** para ajustar conforme sua estratégia:

   * Se usar URLs reais ou exemplos “quase reais”, considere **Privado**.
   * Se for portfólio didático com tudo anonimizado, pode ser **Público**.

---

## 4. Proteção de Branch & Fluxo de Trabalho

### 4.1 Proteger a Branch `main` (sem push direto)

**O que**
Criar uma **branch protection rule** para `main`:

* Proibir commits diretos na `main`
* Exigir Pull Requests (PR) para merge
* Exigir checks e revisão

**Por que**

* Evita push acidental
* Evita sobrescrever histórico com `force push`
* Garante que todo código passe por um “gate” (PR + checagens)

**Como**

1. Vá em **Settings → Branches**.
2. Em **Branch protection rules**, clique em **Add rule**:

   * **Branch name pattern:** `main`
   * Marque:

     * ✅ **Require a pull request before merging**

       * Minimum approvals: `1` (mesmo se você for o único dev, ainda assim usa PR como checkpoint)
       * ✅ **Dismiss stale pull request approvals when new commits are pushed**
       * ✅ **Require review from Code Owners** (vamos criar `CODEOWNERS` depois)
     * ✅ **Require status checks to pass before merging**

       * Selecione os workflows de CI (testes, lint, CodeQL, etc.) assim que existirem
       * ✅ **Require branches to be up to date before merging**
     * ✅ **Require signed commits** (se você for usar commit assinado)
     * ✅ **Restrict who can push to matching branches** (permitir só você/um time específico)
3. Salve.

> Sugestão de fluxo: trabalhar em branches `feature/*` ou uma `dev`, e sempre fazer PR → `main`.

---

## 5. Segredos: Detecção, Prevenção e Tratamento

### 5.1 Nunca Versionar Segredos no Repositório

**O que**
Evitar commitar:

* Chaves de cloud (AWS, GCP, Azure, etc.)
* Tokens do Vault
* Senhas de banco
* API keys em geral

**Por que**
Repos públicos são constantemente varridos por bots procurando segredos. Mesmo em repos privados, um leak interno ainda é grave.

**Como**

* Para segredos, usar:

  * **HashiCorp Vault** (já faz parte do seu lab)
  * **GitHub Actions → Secrets** para CI/CD
  * Arquivos `.env` apenas **locais** e incluídos no `.gitignore`
* Ajustar seu `.gitignore` no `MLOps-Security-Lab`, por exemplo:

```gitignore
# Arquivos de ambiente locais
.env
.env.local
.env.*.local

# IDE & SO
.vscode/
.idea/
.DS_Store

# Artefatos gerados
dist/
build/
*.log
```

---

### 5.2 Habilitar Secret Scanning & Push Protection

**O que**
Ativar **Secret scanning** e **Push protection** (se disponível no seu plano/org) para o repo.

**Por que**

* **Secret scanning**: escaneia histórico, PRs, issues, etc., em busca de segredos.
* **Push protection**: bloqueia pushes que contenham segredos detectados.

**Como**

1. Abra o repo → aba **Security → Code security and analysis** (ou **Settings → Code security and analysis**).
2. Certifique-se de que estão **Enabled**:

   * ✅ **Secret scanning**
   * ✅ **Secret scanning → Push protection**
3. Se estiver em uma org com planos mais avançados, usar também **custom patterns** para tipos de segredo internos.

---

## 6. Dependências & Análise Estática

### 6.1 Habilitar Dependabot Alerts & Security Updates

**O que**
Ativar **Dependabot alerts** e **Dependabot security updates**.

**Por que**

* Dependências (Python, GitHub Actions, etc.) podem conter CVEs.
* O Dependabot avisa e pode abrir PR automático para atualizar libs vulneráveis.

**Como**

1. Vá em **Settings → Code security and analysis**.
2. Ative:

   * ✅ **Dependabot alerts**
   * ✅ **Dependabot security updates**
   * ✅ **Dependency graph** (se aparecer a opção; hoje costuma ser automático)
3. Adicione o arquivo `.github/dependabot.yml`:

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "pip"
    directory: "/"
    schedule:
      interval: "weekly"

  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
```

---

### 6.2 Code Scanning (CodeQL ou outro SAST)

**O que**
Habilitar **Code Scanning** com **CodeQL** ou outra ferramenta SAST suportada.

**Por que**

* Ajuda a detectar vulnerabilidades de código (injeção, uso inseguro de libs, etc.) cedo no ciclo de vida.

**Como (exemplo com CodeQL nativo)**

1. No repo, vá em **Security → Code scanning alerts**.
2. Clique em **Set up code scanning** → **CodeQL analysis**.
3. Aceite o workflow padrão e crie o PR → ele adiciona `.github/workflows/codeql.yml`.
4. Ajuste linguagens e gatilhos conforme o projeto evoluir.

---

## 7. Políticas do Repositório e Responsabilidades

### 7.1 Criar `SECURITY.md` – Política de Segurança

**O que**
Criar `SECURITY.md` descrevendo como reportar vulnerabilidades e o que esperar em termos de resposta.

**Por que**

* Mostra maturidade de segurança
* Dá um canal claro pra alguém te avisar se achar problema

**Como**

Crie `SECURITY.md` na raiz ou em `.github/SECURITY.md`:

```markdown
# Política de Segurança — MLOps-Security-Lab

## Como reportar uma vulnerabilidade

Se você encontrar um problema de segurança neste repositório, por favor entre em contato por:

- E-mail: **[seu_email@exemplo.com]**
- GitHub Issues: abra uma issue com o label `security` (sem expor segredos ou dados sensíveis).

Inclua:
- Descrição do problema
- Passos para reproduzir (se possível)
- Impacto potencial

## Prazo de resposta (intenção)

- **Confirmação de recebimento**: até 5 dias úteis
- **Avaliação inicial**: até 10 dias úteis
- **Correção / ETA**: comunicado após a avaliação, dependendo da complexidade

## Divulgação responsável

Por favor:
- Não divulgue a vulnerabilidade publicamente antes de existir correção.
- Não explore a vulnerabilidade além do necessário para o PoC.

## Escopo

Esta política vale para o código e configurações presentes neste repositório.
```

---

### 7.2 Criar `CODEOWNERS` – Donos de Código

**O que**
Definir um arquivo `CODEOWNERS` indicando quem é responsável por cada parte do repo.

**Por que**

* Combinado com branch protection (`Require review from Code Owners`):

  * Força review de quem realmente entende aquela área (infra, segurança, docs, etc.)

**Como**

1. Crie `.github/CODEOWNERS`:

```text
# Dono global
*          @seu-usuario-github

# Documentação de hardening
/docs/hardening/    @seu-usuario-github

# Infra / IaC (se existir)
infra/              @seu-usuario-github
```

2. Confirme que, na protection rule da `main`, a opção **Require review from Code Owners** está marcada.

---

### 7.3 Criar `CONTRIBUTING.md` – Regras de Contribuição

**O que**
Documentar como contribuir e quais regras de segurança seguir.

**Por que**

* Mesmo sendo projeto pessoal, te obriga a explicitar:

  * Estratégia de branches
  * Regras de PR
  * Expectativas de segurança

**Como (exemplo básico)**

```markdown
# Contribuindo com o MLOps-Security-Lab

## Modelo de branches

- `main`: branch estável e protegida.
- `dev`: branch de integração (opcional).
- `feature/*`: branches de feature curtas.

Todas as alterações devem passar por Pull Request.

## Regras para Pull Requests

- Pelo menos 1 review é obrigatório.
- Todos os checks (testes, lint, code scanning) devem estar verdes.
- É proibido adicionar segredos (chaves, senhas, tokens) no código ou configs.
- Mudanças sensíveis (segurança, infra) devem ser descritas claramente na descrição do PR.

## Commits

Use mensagens descritivas, por exemplo:

- `feat: adicionar docs de arquitetura DatOps`
- `fix: ajustar configuração do Vault`
- `chore: atualizar dependências`
```

---

## 8. Segurança em GitHub Actions / CI-CD

> Se você já usa ou pretende usar Actions neste lab, isto é crítico.

### 8.1 Restringir Permissões do `GITHUB_TOKEN`

**O que**
Definir **permissions** explícitas em cada workflow, em vez de usar o padrão (mais permissivo).

**Por que**

* Por padrão, o `GITHUB_TOKEN` pode ter permissões de escrita no repo.
* Se um workflow for comprometido, menos permissões = menor estrago possível.

**Como (exemplo)**

No topo de `.github/workflows/*.yml`:

```yaml
permissions:
  contents: read
  pull-requests: write
  # Adicione apenas o que realmente precisar
```

Evite deixar sem `permissions`, pois isso normalmente aplica um default mais amplo.

---

### 8.2 Usar OIDC para Acesso à Cloud (sem chaves fixas)

**O que**
Para AWS (no contexto do seu lab), usar **GitHub OIDC** em vez de chaves de acesso IAM estáticas armazenadas como secrets.

**Por que**

* Evita chaves de longa duração em GitHub Secrets
* Cada job obtém credenciais temporárias e auditáveis

**Como (visão geral)**

1. No IAM da AWS, crie um **provedor OIDC** para o GitHub.
2. Crie uma role com:

   * Trust policy permitindo GitHub OIDC com condições (repo = `MLOps-Security-Lab`, branch = `main`, etc.).
3. No workflow do GitHub, use:

```yaml
permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Configurar credenciais AWS
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-mlops-lab
          aws-region: us-east-1
```

---

### 8.3 Usar Environments (`lab`, `prod`) com Aprovação

**O que**
Criar **Environments** no GitHub (ex.: `lab`, `prod`) com revisores obrigatórios.

**Por que**

* Evita deploy acidental em “prod”
* Isola secrets por ambiente

**Como**

1. Em **Settings → Environments**, crie:

   * `lab`
   * `prod`
2. Para `prod`:

   * Configure **Required reviewers** (quem precisa aprovar qualquer deploy)
   * Configure secrets específicos do ambiente (`VAULT_ADDR_PROD`, etc.)

---

## 9. Estrutura de Diretórios de Hardening

### 9.1 Centralizar Documentação de Segurança

**O que**
Criar um diretório para documentar todas as decisões de hardening:

* `docs/hardening/github-hardening-mlops-security-lab.md` (este arquivo)
* `docs/hardening/aws-account-hardening.md`
* `docs/hardening/github-actions-security.md`

**Por que**

* Mantém a segurança visível e organizada
* Alinha com sua visão de “baseline” e “segurança como código”
* Facilita referência em README, ADRs, etc.

**Como**

Estrutura sugerida:

```text
docs/
  hardening/
    github-hardening-mlops-security-lab.md
    aws-account-hardening.md
    github-actions-security.md
```

No `README.md` principal, adicionar:

```markdown
## 🔐 Segurança & Hardening

Para detalhes sobre como este repositório é protegido:

- [Hardening do GitHub — MLOps-Security-Lab](docs/hardening/github-hardening-mlops-security-lab.md)
- [Hardening da Conta AWS / Lab](docs/hardening/aws-account-hardening.md)
```

---

## 10. Monitoramento Contínuo

### 10.1 Revisar Alertas de Segurança Periodicamente

**O que**
Monitorar regularmente:

* **Security → Dependabot alerts**
* **Security → Secret scanning alerts**
* **Security → Code scanning alerts**

**Por que**

* Segurança não é configuração “uma vez e pronto”.
* Dependências e superfícies de ataque mudam com o tempo.

**Como**

* Criar um lembrete recorrente (semanal ou quinzenal) no seu sistema de tarefas:

  * “Revisar alertas de segurança do `MLOps-Security-Lab` no GitHub”
* Ao revisar:

  * Priorizar severidade **Critical/High**
  * Avaliar se o repo é público ou privado pra medir impacto
  * Criar issues/PRs para correções necessárias

---

## 11. Checklist Rápido (TL;DR)

Você pode colar isto no final do arquivo como checklist:

```markdown
- [ ] MFA habilitada na conta GitHub (e na Org, se houver)
- [ ] Visibilidade do repo (público/privado) revisada e justificada
- [ ] Branch `main` protegida (sem push direto, PR obrigatório, checks exigidos)
- [ ] `CODEOWNERS` criado e integrado à proteção de branch
- [ ] Nenhum segredo versionado (uso de Vault / GitHub Secrets / .env no .gitignore)
- [ ] Secret scanning + Push protection habilitados
- [ ] Dependabot alerts + security updates habilitados
- [ ] Code scanning configurado (CodeQL ou similar)
- [ ] `SECURITY.md` e `CONTRIBUTING.md` criados
- [ ] Workflows do GitHub Actions com `permissions` de mínimo privilégio
- [ ] Acesso à AWS via OIDC (sem chaves IAM de longa duração em secrets)
- [ ] Environments (`lab`, `prod`) configurados com approvals e secrets por ambiente
- [ ] Revisão periódica dos alertas de segurança do repo
```
