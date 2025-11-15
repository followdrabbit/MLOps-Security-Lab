# Contribuindo com o MLOps-Security-Lab

## Modelo de branches

- `main`: branch estável e protegida.
- `dev`: branch de integração.
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