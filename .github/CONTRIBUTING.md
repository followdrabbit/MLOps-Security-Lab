# Contribuindo com o MLOps-Security-Lab

Obrigado por contribuir com o **MLOps-Security-Lab**!  
Este projeto existe para estudar **MLOps com foco em segurança**, então qualidade e segurança do código são prioridade.

---

## 🔀 Modelo de branches

- `main`  
  - Branch **estável** e **protegida**.  
  - Sempre deve refletir um estado utilizável do projeto.

- `dev`  
  - Branch de **integração**.  
  - Recebe merges de `feature/*` após validação via Pull Request.

- `feature/*`  
  - Branches de desenvolvimento de novas features, ajustes ou fixes.  
  - Ex.: `feature/ajustar-vault`, `feature/docs-datops`.

---

## 🔁 Fluxo de trabalho (OBRIGATÓRIO)

O fluxo abaixo **é obrigatório para qualquer contribuição**:

1. **Criar uma branch de feature a partir de `dev`:**
   ```bash
   git checkout dev
   git pull origin dev
   git checkout -b feature/minha-feature
````

2. **Desenvolver as mudanças na branch `feature/*`.**

3. **Abrir um Pull Request de `feature/*` para `dev`:**

   * Origem: `feature/minha-feature`
   * Destino: `dev`
   * O PR só pode ser aprovado se:

     * [ ] Todos os checks de **CI** (testes, lint, code scanning, etc.) estiverem **verdes**
     * [ ] As regras de segurança forem respeitadas (sem segredos, sem dados sensíveis)

4. **Depois que `dev` estiver estável e validado**, abrir um novo Pull Request:

   * Origem: `dev`
   * Destino: `main`
   * Também **obrigatoriamente** com CI verde.

> ✅ **É proibido** fazer commits ou merges diretamente em `main` ou `dev`.
> Todos os fluxos devem passar por Pull Request + CI.

---

## ✅ Regras para Pull Requests

* Pelo menos **1 review aprovado** é obrigatório.
* Todos os **checks de CI** devem estar **verdes**:

  * Testes automatizados
  * Lint
  * Code scanning / security checks (quando configurados)
* É **proibido** adicionar segredos no repositório:

  * Chaves de API
  * Tokens
  * Senhas
  * Certificados
* Para mudanças sensíveis (segurança, infra, arquitetura), a descrição do PR deve incluir:

  * **O que foi alterado**
  * **Por que foi alterado**
  * **Impactos esperados** (risco, compatibilidade, segurança)

Sugestão de checklist na descrição do PR:

* [ ] Testes locais executados
* [ ] CI passando
* [ ] Sem segredos adicionados
* [ ] Documentação atualizada (se aplicável)

---

## 💬 Padrão de commits

Use mensagens **claras e descritivas**, preferencialmente no padrão *conventional commits*:

* `feat: adicionar docs de arquitetura DatOps`
* `fix: ajustar configuração do Vault`
* `chore: atualizar dependências`
* `docs: melhorar guia de instalação`
* `refactor: simplificar pipeline de CI`

Boas práticas para commits:

* Commits menores e coesos.
* Evitar mensagens vagas como `ajustes`, `fix`, `wip` sem contexto.
* Cada commit deve manter o projeto em estado funcional.

---

## 🔐 Boas práticas de segurança

Como o foco do projeto é segurança, observe sempre:

* Não versionar:

  * `.env`
  * Chaves privadas
  * Dumps de banco
  * Arquivos com dados sensíveis
* Usar variáveis de ambiente ou secret managers (quando aplicável).
* Se encontrar algum problema de segurança:

  * Descrever claramente no PR (sem expor segredos reais).
  * Sugerir mitigação ou controles adicionais, se possível.

---

## 🧪 Testes e qualidade

Antes de abrir um PR:

1. Execute os testes locais (quando disponíveis).
2. Corrija *linters* e *warnings* reportados.
3. Verifique se a documentação continua coerente com as mudanças.

> O PR só deve ser aberto quando você estiver razoavelmente confiante de que as alterações não quebram o fluxo principal do lab.

---

## 👥 Revisores e CODEOWNERS

Este repositório utiliza o arquivo `CODEOWNERS` para definir responsáveis por revisão de código.

* Pull Requests para `dev` e `main` **devem** ser aprovados por pelo menos um **Code Owner** (quando aplicável).
* As regras de proteção de branch impedem:

  * Push direto em `main` e `dev`;
  * Merge sem CI verde;
  * Merge sem aprovação dos revisores obrigatórios.

Se o seu PR afetar áreas críticas (segurança, infraestrutura, CI/CD), é esperado que você envolva o(s) Code Owner(s) correspondente(s).

---

Se tiver dúvida sobre como contribuir, abrir um PR ou estruturar uma feature, sinta-se à vontade para abrir uma **issue** com a tag `question` ou `help wanted`.
