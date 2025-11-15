# 🛡️ Hardening da Conta AWS — MLOps Security Lab

## 1. Objetivo

Este documento descreve como **endurecer (hardening)** a conta AWS utilizada pelo projeto `MLOps-Security-Lab`, com foco em:

* Reduzir risco de **comprometimento da conta**
* Proteger o **ambiente de laboratório** e facilitar a futura separação entre **lab** e **prod**
* Garantir que os serviços usados no lab (Vault, MLOps stack, etc.) rodem em uma base minimamente segura **sem estourar custos**

---

## 2. Escopo

Este hardening cobre principalmente:

* Configurações de **conta AWS** (root, IAM, Organizations, billing, etc.)
* Controles **globais** aplicáveis a qualquer workload do lab
* Itens de segurança que **não dependem de um serviço específico** (S3, EC2, etc. serão detalhados em baselines próprios)

Serviços gerenciados de segurança com custo adicional (ex.: GuardDuty, Security Hub, Macie, AWS Config) aparecem em uma seção separada como **recomendados**, mas **não fazem parte do mínimo essencial do lab**.

---

## 3. Princípios Gerais

1. **Root não é para uso diário**
2. **MFA físico para operações críticas** (root e, se possível, usuários administrativos)
3. **Privilégio mínimo** para identidades e roles (IAM)
4. **Tudo observado**: pelo menos CloudTrail ativo e preservado
5. **Tudo alertado**: eventos críticos geram notificação (e-mail / SNS)
6. **Criptografia por padrão**, sempre que possível
7. **Infra como código**: sempre que puder, registrar essas configs em Terraform/CloudFormation depois

---

## 4. Hardening da Conta Root

### 4.1 Senha forte e exclusiva para root

**O que**

* Definir uma senha **longa, aleatória e exclusiva** para o usuário root da conta.

**Por que**

* O root tem poder total na conta.
* Reaproveitar senha (ou usar senha fraca) torna o impacto de vazamento muito grave.

**Como**

1. Acesse a console com o usuário root.
2. Vá em **My Security Credentials → Password → Change password**.
3. Use um gerenciador de senhas para gerar e armazenar essa senha.
4. Nunca use essa senha no dia a dia; apenas em cenários excepcionais (ex.: recuperação de conta, algumas configurações de billing).

---

### 4.2 Habilitar MFA físico para root

**O que**

* Ativar **MFA com dispositivo físico** (chave U2F/FIDO2 ou token hardware) para o usuário root.

**Por que**

* Mesmo que a senha vaze, um atacante ainda precisa do dispositivo físico para logar.
* MFA em app (TOTP) já é bom; MFA físico é ainda melhor.

**Como**

1. Logar como root.
2. Acessar **IAM → Dashboard → Security recommendations → Activate MFA on your root account**.
3. Escolher **Security key** (ou outro dispositivo suportado).
4. Registrar a chave seguindo o passo a passo da AWS.
5. Testar logout/login para garantir que está funcionando.

---

### 4.3 Bloquear uso diário do root

**O que**

* Após habilitar MFA, **não usar root** para tarefas diárias.
* Realizar atividades administrativas via **usuário IAM + role de administração**.

**Por que**

* Reduz a chance de erro humano com credenciais de altíssimo privilégio.
* Permite auditoria melhor (CloudTrail registra quem fez o quê).

**Como**

1. Criar um usuário IAM administrativo (ver seção 5).
2. Guardar as credenciais root de forma segura, para uso apenas em emergência.
3. Nunca criar chaves de acesso (Access Keys) para o root.
4. Opcional: criar alerta de **login de root** (ver seção 8).

---

## 5. Identidade e Acesso (IAM)

### 5.1 Criar grupo e usuário admin (para lab)

**O que**

* Criar um **grupo IAM** para administração (ex.: `Admins-Lab`) e um usuário IAM para você, com MFA.

**Por que**

* Evita usar o root.
* Permite distinguir suas ações das ações de outros usuários futuros.
* Facilita aplicar política de “human admin” x “service role”.

**Como**

1. Em **IAM → User groups**, criar grupo `Admins-Lab`.
2. Anexar a esse grupo a política `AdministratorAccess` (para lab inicial; depois pode refinar).
3. Em **IAM → Users**, criar um usuário para você (ex.: `rapha-admin`):

   * Tipo: acesso à **Console** e, se necessário, **Programmatic access**.
4. Adicionar o usuário ao grupo `Admins-Lab`.
5. Logar com esse usuário e configurar **MFA** (app ou chave física).

> Em um cenário mais maduro, você pode trocar esse modelo por **IAM Identity Center / SSO** e roles temporárias.

---

### 5.2 MFA obrigatório para usuários privilegiados

**O que**

* Exigir MFA para qualquer usuário com privilégios altos (admin, segurança, billing).

**Por que**

* Reduz risco de comprometimento de contas privilegiadas.
* Consistência com o que foi feito para root.

**Como**

* Criar uma política IAM que exija MFA para chamadas sensíveis (console e/ou API).
* Anexar essa política aos grupos/usuários administrativos.
* Documentar a obrigatoriedade neste arquivo e em um futuro **baseline IAM**.

---

### 5.3 Evitar uso de chaves de acesso estáticas

**O que**

* Minimizar (ou eliminar) o uso de **Access Keys** em usuários IAM.
* Usar **roles** e **perfis de instância** (EC2/ECS) ou **OIDC** (com GitHub, etc.).

**Por que**

* Chaves estáticas podem vazar e permanecer válidas por muito tempo.
* Roles fornecem credenciais temporárias, rotacionadas automaticamente.

**Como**

* Para workloads na AWS:

  * Usar **Instance Profiles** em EC2, **Task Roles** em ECS, **Lambda Execution Roles**, etc.
* Para integração com GitHub:

  * Usar **OIDC** em vez de Access Keys (ver docs de GitHub Actions Security).
* Revisar periodicamente em **IAM → Users → Security credentials**:

  * Remover Access Keys antigas ou não utilizadas.

---

### 5.4 Políticas de privilégio mínimo

**O que**

* Evitar políticas com `"Action": "*"`, `"Resource": "*"` exceto em casos muito específicos (ex.: role de admin em lab).
* Criar políticas mais específicas para serviços (S3, EC2, etc.).

**Por que**

* Reduz o impacto caso uma credencial seja comprometida.
* Torna mais claro o que cada identidade pode fazer.

**Como (alto nível)**

1. Mapear quais serviços o lab realmente usa.
2. Criar policies IAM específicas para:

   * Serviços de MLOps
   * Vault
   * Pipelines de CI/CD
3. Anexar políticas a roles/grupos em vez de usuários diretamente.

---

## 6. Logging, Auditoria e Monitoramento Essenciais (Baixo Custo)

### 6.1 Habilitar CloudTrail multi-região (mínimo essencial)

**O que**

* Configurar um **AWS CloudTrail** que:

  * Seja multi-região
  * Envie logs para um bucket S3 dedicado
  * Tenha **log file validation** habilitado

**Por que**

* Registra quem fez o quê na conta.
* Multi-região garante que ações fora da região principal também sejam auditadas.
* Validação ajuda a detectar alteração de logs.

**Como (resumo)**

1. Acessar **CloudTrail → Trails**.
2. Criar um trail novo (ex.: `lab-account-trail`):

   * **Apply trail to all regions**: Yes
   * **Management events**: Read/Write
   * **Data events**: desabilitar ou habilitar **apenas recursos críticos** se for necessário (data events podem gerar custo maior).
   * **Log file validation**: habilitado
   * **S3 bucket**: criar um bucket dedicado, ex.: `mlops-lab-cloudtrail-logs-<id>`
3. Garantir que o bucket de logs tenha:

   * Criptografia habilitada
   * Acesso público bloqueado

> Para o lab, o foco é **management events multi-região** (parte gratuita/bem barata). Data events podem ficar desligados ou muito restritos para não gerar custo alto.

---

## 7. Proteções Globais de S3

> Detalhes finos de S3 ficarão em um baseline próprio, mas alguns controles são **globais por conta**.

### 7.1 Bloqueio de acesso público em nível de conta

**O que**

* Habilitar o **Block Public Access (BPA)** em nível de conta para S3.

**Por que**

* Evita exposição acidental de buckets/objetos ao público.

**Como**

1. Acessar **S3 → Block Public Access (account settings)**.
2. Marcar todas as opções:

   * Block public ACLs
   * Ignore public ACLs
   * Block public bucket policies
   * Restrict public bucket policies
3. Salvar.

> Se houver caso **real** de bucket público, você pode liberar de forma pontual (e documentada).

---

### 7.2 Criptografia padrão em buckets

**O que**

* Habilitar **Server-Side Encryption (SSE)** como padrão para todos os buckets (pelo menos SSE-S3; idealmente SSE-KMS).

**Por que**

* Garante que dados em repouso estejam sempre criptografados.
* Facilita aderência a requisitos de compliance e boas práticas.

**Como (alto nível)**

* Ao criar buckets, selecionar:

  * **Default encryption**: SSE-S3 ou SSE-KMS.
* Futuramente, usar KMS com chaves gerenciadas pelo cliente (CMK) para dados mais sensíveis.

---

## 8. Alertas e Billing

### 8.1 Alertas de atividades sensíveis (CloudWatch + SNS)

**O que**

* Criar alertas para eventos críticos, como:

  * Login de root
  * Alterações em IAM (policies, users, roles)
  * Desativação de CloudTrail
  * Uso de chaves sem MFA (se aplicável)

**Por que**

* Você quer ser notificado rapidamente se algo muito sensível acontecer, principalmente em um lab exposto a testes.

**Como (visão geral)**

1. A partir de **CloudTrail**, enviar eventos para **CloudWatch Logs** ou **EventBridge**.
2. Criar regras no **EventBridge** para:

   * Filtrar eventos específicos (ex.: `ConsoleLogin` com `userIdentity.type = Root`).
3. Enviar esses eventos para:

   * **SNS topic** que dispare e-mails / integrações (ex.: webhook, n8n).

> CloudWatch Logs + EventBridge + SNS têm custo, mas em um lab pequeno e com poucos eventos a fatura tende a ser baixa. Ajuste períodos e filtros para evitar volume desnecessário.

---

### 8.2 Alertas de custo (AWS Budgets / Billing Alarms)

**O que**

* Criar **AWS Budgets** e/ou alarmes de billing para:

  * Gasto mensal previsto
  * Picos inesperados (lab rodando além do esperado)

**Por que**

* Evita surpresas de custo, especialmente em lab.
* Picos de custo podem indicar também comportamento anômalo (ex.: recurso criado por atacante ou script bugado).

**Como (resumo)**

1. Acessar **Billing → Budgets**.
2. Criar um orçamento mensal (ex.: 20–50 USD, depende do lab) com:

   * Alertas em 50%, 80% e 100% do valor.
3. Configurar envio de e-mail para você.

---

## 9. Serviços Gerenciados de Segurança — Recomendados (Custo Adicional)

> **Não fazem parte do hardening essencial do lab**, mas valem ser considerados se o orçamento permitir ou quando o ambiente crescer.

### 9.1 AWS Config (histórico de configuração)

**O que**

* **AWS Config** registra o histórico de configuração dos recursos e pode aplicar regras de conformidade.

**Por que (benefícios)**

* Permite saber “como” um recurso estava configurado em um momento anterior.
* Ajuda na investigação pós-incidente.
* Permite criar regras do tipo “nenhum bucket S3 pode ser público” e marcar recursos não conformes.

**Custo**

* Cobrança por:

  * Config items gravados
  * Regras avaliadas
* Em lab pequeno, o custo pode ser baixo, mas se você habilitar em muitas regiões/contas, pode crescer.

**Recomendação para o lab**

* Se habilitar:

  * Comece **apenas na região principal** do lab.
  * Use poucas regras gerenciadas de alto impacto (ex.: S3 público, recursos sem criptografia).
  * Monitore custo nos primeiros meses.

---

### 9.2 GuardDuty (detecção de ameaças)

**O que**

* Serviço gerenciado de **detecção de ameaças** que analisa CloudTrail, VPC Flow Logs, DNS, etc.

**Por que (benefícios)**

* Identifica comportamentos suspeitos:

  * Credenciais comprometidas
  * Comunicação com IPs maliciosos
  * Atividades estranhas em IAM, EC2, etc.

**Custo**

* Cobrança baseada em volume de eventos analisados.
* Pode ficar caro em ambientes grandes.

**Recomendação para o lab**

* Não obrigatório no lab inicial.
* Avaliar habilitar GuardDuty:

  * Em regiões específicas
  * Por período de teste (ex.: 30 dias) para entender custo e benefício.

---

### 9.3 Security Hub (painel de segurança)

**O que**

* Serviço que agrega findings de vários serviços (GuardDuty, Config, Inspector, etc.) e aplica padrões como CIS AWS Foundations.

**Por que (benefícios)**

* Consolida findings em um painel único.
* Ajuda a enxergar postura de segurança da conta.

**Custo**

* Cobrança por número de controles avaliados e findings.
* Depende também de outros serviços alimentando dados (Config, GuardDuty, etc.).

**Recomendação para o lab**

* Interessante quando o lab crescer ou virar base para produção.
* No momento, tratar como **nice-to-have** de médio prazo.

---

### 9.4 Macie (descoberta de dados sensíveis em S3)

**O que**

* Serviço que analisa buckets S3 em busca de dados sensíveis (PII, etc.).

**Por que (benefícios)**

* Ajuda a identificar onde dados sensíveis estão armazenados.
* Útil em ambientes com muitos buckets ou dados de clientes.

**Custo**

* Cobrança por volume de dados analisados.
* Pode ficar caro se você apontar Macie para muitos buckets grandes.

**Recomendação para o lab**

* Não faz sentido no estágio inicial, a menos que você coloque dados sensíveis reais nos buckets.
* Avaliar somente se:

  * houver forte requisito de privacidade;
  * o lab estiver armazenando dados que simulam produção (com cuidado).

---

### 9.5 Checklist — Serviços Opcionais (Recomendados)

```markdown
- [ ] AWS Config avaliado (habilitar apenas em regiões/recursos críticos, se fizer sentido)
- [ ] GuardDuty avaliado (período de teste, escopo limitado)
- [ ] Security Hub avaliado como painel unificado (quando houver mais serviços de segurança ativos)
- [ ] Macie avaliado apenas se houver necessidade de descobrir dados sensíveis em S3
- [ ] Monitoramento de custo desses serviços documentado (para não estourar orçamento do lab)
```

---

## 10. Integração com GitHub (OIDC / IAM Roles)

> Este item conecta o hardening da conta com o hardening do repositório.

### 10.1 Criar role dedicada para GitHub OIDC

**O que**

* Criar uma **IAM Role** que será assumida pelos jobs de **GitHub Actions** via OIDC, sem uso de chaves fixas.

**Por que**

* Evita Access Keys armazenadas como secrets no GitHub.
* Usa credenciais temporárias, com escopo claro (repo, branch, etc.).

**Como (alto nível)**

1. Configurar o **provedor OIDC** do GitHub na conta AWS (se ainda não estiver configurado).
2. Criar uma role com trust policy similar a:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:<seu-usuario-ou-org>/MLOps-Security-Lab:*"
        }
      }
    }
  ]
}
```

3. Anexar a essa role uma policy **de privilégio mínimo** necessária para as ações do CI (ex.: acesso limitado a S3, ECR, etc., dependendo do que o lab vai fazer).
4. Referenciar essa role no workflow `.github/workflows/ci.yml` ou em outro workflow de deploy.

---

## 11. Organização de Contas (Lab vs Prod) — Visão Futura

> Opcional neste momento, mas importante como visão.

**O que**

* Usar **AWS Organizations** para separar:

  * **Conta de Lab**
  * **Conta de Produção** (futura)
  * Eventualmente contas específicas para segurança, logging, etc.

**Por que**

* Isola ambientes em nível de **conta**, reduzindo impacto de incidentes.
* Facilita aplicar SCPs (Service Control Policies) com regras diferentes para lab e prod.

**Como (visão futura)**

* Criar uma organização com:

  * `root` org
  * OU `Sandbox` / `Lab`
  * OU `Prod`
* Mover a conta existente para a OU `Lab` (quando fizer sentido).

---

## 12. Checklist Rápido — Hardening Essencial (Baixo Custo)

```markdown
- [ ] Senha root forte, aleatória e exclusiva
- [ ] MFA físico habilitado para o usuário root
- [ ] Root não usado no dia a dia (apenas emergência)
- [ ] Usuário IAM administrativo criado (ex.: rapha-admin) com MFA
- [ ] Grupo de admins (Admins-Lab) criado e configurado
- [ ] Política de privilégio mínimo definida para roles e usuários
- [ ] Uso de Access Keys minimizado (preferência por roles e OIDC)
- [ ] CloudTrail habilitado em todas as regiões, com log file validation e data events apenas se necessário
- [ ] Block Public Access do S3 habilitado em nível de conta
- [ ] Criptografia padrão para buckets S3 habilitada
- [ ] Alertas de login root e eventos sensíveis configurados (EventBridge + SNS)
- [ ] AWS Budgets / Billing alarms configurados para controlar custos do lab
- [ ] Role IAM criada para integração com GitHub via OIDC (sem Access Keys)
- [ ] Plano futuro de multi-contas (Organizations) documentado
```
