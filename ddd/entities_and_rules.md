# 📚 DDD Detalhado Completo para Todos Microsserviços FinBank

---

## 1. auth-service

### Contexto

Gerenciamento de usuários, autenticação, MFA, sessão, autorização via tokens JWT.

### Agregados / Entidades

* **User**

  * `id: UUID`
  * `username: string`
  * `email: string`
  * `passwordHash: string` (armazenado só se usar custom auth)
  * `status: enum (ACTIVE, BLOCKED, PENDING_CONFIRMATION)`
  * `roles: Role[]` (admin, user, etc)
  * `createdAt: Date`
  * `updatedAt: Date`

* **Session**

  * `id: UUID`
  * `userId: UUID`
  * `refreshToken: string`
  * `expiresAt: Date`
  * `createdAt: Date`

* **MfaChallenge**

  * `userId: UUID`
  * `challengeType: enum (SMS, TOTP)`
  * `session: string` (referencia do Cognito ou sistema MFA)
  * `expiresAt: Date`

### Value Objects

* **Role**

  * `name: string`
  * `permissions: Permission[]`

* **Permission**

  * `name: string` (ex: "user.create", "loan.approve")

### Regras/Serviços de Domínio

* Validar senha e login
* Gerar/chamar MFA Challenge
* Validar código MFA
* Gerar JWT access e refresh tokens
* Revogar tokens e sessões

### Interfaces / Portas

* `IUserRepository` — métodos CRUD e buscar por username/email
* `ISessionRepository` — CRUD sessões
* `IMfaService` — criação e validação MFA challenges (integrado com Cognito ou outro)
* `ITokenService` — geração e validação JWT

---

## 2. account-service

### Contexto

Gerenciamento de contas bancárias, saldos, movimentações internas e transferências.

### Agregados / Entidades

* **Account** (Conta bancária)

  * `id: UUID`
  * `ownerId: UUID` (usuário dono da conta)
  * `type: enum (CHECKING, SAVINGS)`
  * `status: enum (ACTIVE, BLOCKED, CLOSED)`
  * `balance: Money`
  * `createdAt: Date`
  * `updatedAt: Date`

* **Transaction**

  * `id: UUID`
  * `accountId: UUID`
  * `type: enum (DEPOSIT, WITHDRAWAL, TRANSFER_IN, TRANSFER_OUT)`
  * `amount: Money`
  * `date: Date`
  * `description: string`
  * `relatedAccountId?: UUID` (para transferências)
  * `status: enum (PENDING, COMPLETED, FAILED)`

### Value Objects

* **Money**

  * `amount: number`
  * `currency: string` (ex: "BRL")

### Regras/Serviços de Domínio

* Depositar valor: adiciona valor no saldo e registra transação
* Sacar valor: valida saldo suficiente e registra transação
* Transferência interna: débito em uma conta e crédito na outra, evento Kafka emitido
* Atualizar saldo consistentemente
* Validar regras de bloqueio e status da conta

### Interfaces / Portas

* `IAccountRepository` — CRUD contas
* `ITransactionRepository` — CRUD transações
* `IEventPublisher` — para publicar eventos no Kafka sobre transações

---

## 3. loan-service

### Contexto

Solicitação, aprovação, gerenciamento e pagamento de empréstimos.

### Agregados / Entidades

* **LoanProposal**

  * `id: UUID`
  * `applicantId: UUID`
  * `amountRequested: Money`
  * `termMonths: number`
  * `status: enum (PENDING, APPROVED, REJECTED)`
  * `score: number` (pontuação para aprovação)
  * `createdAt: Date`

* **LoanContract**

  * `id: UUID`
  * `proposalId: UUID`
  * `amountApproved: Money`
  * `interestRate: number` (ao ano)
  * `startDate: Date`
  * `endDate: Date`
  * `status: enum (ACTIVE, PAID_OFF, DEFAULTED)`
  * `installments: LoanInstallment[]`

* **LoanInstallment**

  * `id: UUID`
  * `contractId: UUID`
  * `dueDate: Date`
  * `amount: Money`
  * `status: enum (PENDING, PAID, LATE)`

### Value Objects

* **Money** (igual account-service)

### Regras/Serviços de Domínio

* Solicitar empréstimo: cria proposta com validações iniciais
* Aprovar empréstimo: calcula score, regras, cria contrato e parcelas
* Registrar pagamento: atualiza parcela e contrato
* Calcular juros e multas por atraso

### Interfaces / Portas

* `ILoanProposalRepository`
* `ILoanContractRepository`
* `ILoanPaymentService` — orquestra pagamentos e status
* `IAccountService` — integração para débito/ crédito nas contas

---

## 4. pix-service

### Contexto

Recebimento e envio de transações PIX via DynamoDB streams e Kafka.

### Agregados / Entidades

* **PixTransaction**

  * `id: UUID`
  * `pixKey: string`
  * `payerAccountId: UUID`
  * `payeeAccountId: UUID`
  * `amount: Money`
  * `status: enum (PENDING, COMPLETED, FAILED)`
  * `timestamp: Date`

### Value Objects

* **Money** (reutilizar)

### Regras/Serviços de Domínio

* Receber evento do DynamoDB stream (nova transação)
* Validar e processar transação PIX
* Publicar eventos Kafka para downstream
* Atualizar status da transação

### Interfaces / Portas

* `IPixTransactionRepository` (DynamoDB)
* `IKafkaPublisher`

---

## 5. bank-reconciliation-service

### Contexto

Processamento batch de arquivos CNAB para conciliação bancária.

### Agregados / Entidades

* **BankStatementFile**

  * `id: UUID`
  * `fileName: string`
  * `status: enum (UPLOADED, PROCESSING, PROCESSED, FAILED)`
  * `uploadDate: Date`

* **BankReconciliationEntry**

  * `id: UUID`
  * `statementFileId: UUID`
  * `transactionDate: Date`
  * `amount: Money`
  * `accountId: UUID`
  * `status: enum (MATCHED, UNMATCHED)`

### Value Objects

* **Money**

### Regras/Serviços de Domínio

* Processar arquivo CNAB: parsear e extrair transações
* Conciliar com base de dados interna (Aurora)
* Gerar relatórios de conciliação
* Enviar alertas se inconsistências

### Interfaces / Portas

* `IBankStatementRepository`
* `IBankReconciliationRepository`

---

## 6. open-banking-service

### Contexto

Integração com APIs externas via gRPC para consulta de dados financeiros.

### Agregados / Entidades

* **ExternalBankAccount**

  * `id: UUID`
  * `externalId: string`
  * `userId: UUID`
  * `bankName: string`
  * `accountType: string`
  * `balance: Money`

* **ExternalTransaction**

  * `id: UUID`
  * `externalId: string`
  * `bankAccountId: UUID`
  * `amount: Money`
  * `transactionDate: Date`

### Regras/Serviços de Domínio

* Adaptar dados externos para modelo interno
* Cache e refresh tokens OAuth2
* Sincronizar dados periodicamente

### Interfaces / Portas

* `IExternalBankApiClient` (gRPC)
* `IBankAccountRepository`
* `ITransactionRepository`

---

## 7. legacy-migration-service

### Contexto

Middleware para migrar APIs SOAP antigas para REST modernas.

### Agregados / Entidades

* **LegacyEntity** (pode variar conforme sistema)

  * `id: UUID`
  * dados diversos conforme entidade SOAP

* **MigrationStatus**

  * `entityId: UUID`
  * `status: enum (PENDING, MIGRATED, FAILED)`

### Regras/Serviços de Domínio

* Converter chamadas SOAP para REST
* Mapear dados entre formatos
* Registrar status e logs de migração

### Interfaces / Portas

* `ILegacySoapClient`
* `IMigrationRepository`
* `IRestApiController`

---

# ⚠️ Nota Final

* Use **Value Objects** para dados que precisam de regras (ex: Money com validação de moeda e precisão).
* Entidades são mutáveis, com identidade própria (UUID).
* Agregados isolam consistência e regras de negócio.
* Repositórios abstraem persistência.
* Serviços de domínio orquestram regras que envolvem múltiplas entidades.
