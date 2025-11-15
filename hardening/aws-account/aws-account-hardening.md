# 🛡️ Hardening da Conta AWS — MLOps Security Lab

## 1. Objetivo

Este documento descreve como **endurecer (hardening)** a conta AWS utilizada pelo projeto `MLOps-Security-Lab`, com foco em:

* Reduzir risco de **comprometimento da conta**
* Proteger o **ambiente de laboratório** e facilitar a futura separação entre **lab** e **prod**
* Garantir que os serviços usados no lab (Vault, MLOps stack, etc.) rodem em uma base minimamente segura

---

## 2. Escopo

Este hardening cobre principalmente:

* Configurações de **conta AWS** (root, IAM, Organizations, billing, etc.)
* Controles **globais** aplicáveis a qualquer workload do lab
* Itens de segurança que **não dependem de um serviço específico** (S3, EC2, etc. serão detalhados em baselines próprios)

---

## 3. Princípios Gerais

1. **Root não é para uso diário**
2. **MFA físico para operações críticas** (root e, se possível, usuários administrativos)
3. **Privilégio mínimo** para identidades e roles (IAM)
4. **Tudo observado**: logs centralizados (CloudTrail, Config, etc.)
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

## 6. Logging, Auditoria e Monitoramento

### 6.1 Habilitar CloudTrail multi-região

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
2. Criar um trail novo (ex.: `org-trail` ou `lab-account-trail`):

   * **Apply trail to all regions**: Yes
   * **Management events**: Read/Write
   * **Data events**: pelo menos S3 e Lambda mais críticos (opcional no início)
   * **Log file validation**: habilitado
   * **S3 bucket**: criar um bucket dedicado, ex.: `mlops-lab-cloudtrail-logs-<id>`
3. Garantir que o bucket de logs tenha:

   * Criptografia habilitada
   * Acesso público bloqueado

---

### 6.2 Habilitar AWS Config

**O que**

* Habilitar **AWS Config** para rastrear o histórico de configuração dos recursos.

**Por que**

* Permite saber “como” um recurso estava configurado em um momento anterior.
* Ajuda na investigação pós-incidente.

**Como**

1. Acessar **AWS Config**.
2. Escolher:

   * Regiões nas quais o lab opera (ex.: `us-east-1`, `sa-east-1`) ou **todas**.
3. Configurar:

   * Bucket S3 para armazenar os snapshots do Config.
   * Regras gerenciadas básicas (ex.: recursos sem criptografia, S3 público, etc.).

---

### 6.3 Ativar GuardDuty, Security Hub e (opcional) Macie

**O que**

* Ativar:

  * **GuardDuty** (detecção de ameaças)
  * **Security Hub** (painel unificado de segurança)
  * Opcional: **Macie** (descoberta de dados sensíveis em S3)

**Por que**

* GuardDuty aumenta a visibilidade sobre padrões de ataque (IAM, rede, logs).
* Security Hub consolida findings (inclusive de GuardDuty, Config, etc.).
* Macie ajuda se você armazenar dados sensíveis nos buckets do lab.

**Como (resumo)**

1. Em **GuardDuty**, clicar em **Enable** na(s) região(ões) relevantes.
2. Em **Security Hub**, clicar em **Enable Security Hub**:

   * Habilitar padrões como CIS, AWS Foundations, etc.
3. Em **Macie** (se fizer sentido para o lab), habilitar e configurar escaneamento dos buckets de dados.

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

1. A partir de **CloudTrail**, enviar eventos para **CloudWatch Logs** (ou EventBridge).
2. Criar regras no **EventBridge** para:

   * Filtrar eventos específicos (ex.: `ConsoleLogin` com `userIdentity.type = Root`).
3. Enviar esses eventos para:

   * **SNS topic** que dispare e-mails / integrações (ex.: webhook, n8n).

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

## 9. Integração com GitHub (OIDC / IAM Roles)

> Este item conecta o hardening da conta com o hardening do repositório.

### 9.1 Criar role dedicada para GitHub OIDC

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
          "token.actions.githubusercontent.com:sub": "repo:<seu-usuario- ou-org>/MLOps-Security-Lab:*"
        }
      }
    }
  ]
}
```

3. Anexar a essa role uma policy **de privilégio mínimo** necessária para as ações do CI (ex.: acesso limitado a S3, ECR, etc., dependendo do que o lab vai fazer).
4. Referenciar essa role no workflow `.github/workflows/ci.yml` ou em outro workflow de deploy.

---

## 10. Organização de Contas (Lab vs Prod) — Visão Futura

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

## 11. Checklist Rápido — Hardening da Conta AWS (Lab)

```markdown
- [ ] Senha root forte, aleatória e exclusiva
- [ ] MFA físico habilitado para o usuário root
- [ ] Root não usado no dia a dia (apenas emergência)
- [ ] Usuário IAM administrativo criado (ex.: rapha-admin) com MFA
- [ ] Grupo de admins (Admins-Lab) criado e configurado
- [ ] Política de privilégio mínimo definida para roles e usuários
- [ ] Uso de Access Keys minimizado (preferência por roles e OIDC)
- [ ] CloudTrail habilitado em todas as regiões, com log file validation
- [ ] AWS Config habilitado, com bucket de logs dedicado
- [ ] GuardDuty e Security Hub habilitados (Macie opcional)
- [ ] Block Public Access do S3 habilitado em nível de conta
- [ ] Criptografia padrão para buckets S3 habilitada
- [ ] Alertas de login root e eventos sensíveis configurados (EventBridge + SNS)
- [ ] AWS Budgets / Billing alarms configurados para controlar custos do lab
- [ ] Role IAM criada para integração com GitHub via OIDC (sem Access Keys)
- [ ] Plano futuro de multi-contas (Organizations) documentado
```
