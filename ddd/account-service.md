# account-service - Documentação Técnica Completa

---

## 1. Diagrama de Classes (DDD)

### 1.1 Entidades

* **Account**

  * id: UUID
  * userId: UUID (dono da conta)
  * accountNumber: string (número único da conta)
  * accountType: enum (corrente, poupança, etc)
  * balance: decimal (saldo atual)
  * status: enum (ativa, bloqueada, encerrada)
  * createdAt: Date
  * updatedAt: Date

* **Transaction**

  * id: UUID
  * accountId: UUID
  * type: enum (deposit, withdraw, transfer)
  * amount: decimal
  * status: enum (pending, completed, failed)
  * referenceId: string (ex: id da transferência ou pagamento)
  * createdAt: Date
  * updatedAt: Date

* **Transfer** (Value Object)

  * fromAccountId: UUID
  * toAccountId: UUID
  * amount: decimal
  * status: enum (pending, completed, failed)
  * transactionId: UUID
  * createdAt: Date

### 1.2 Interfaces (Ports)

* **IAccountRepository**

  * findById(id: UUID): Promise\<Account | null>
  * findByUserId(userId: UUID): Promise\<Account\[]>
  * create(account: Account): Promise<Account>
  * update(account: Account): Promise<void>

* **ITransactionRepository**

  * create(transaction: Transaction): Promise<Transaction>
  * findByAccountId(accountId: UUID): Promise\<Transaction\[]>

* **ITransferService**

  * executeTransfer(transfer: Transfer): Promise<void>

---

## Fluxos Detalhados - Account Service

### 1. Criação de Conta Bancária

1. O cliente envia uma requisição HTTP POST para o endpoint `/accounts` com dados do cliente (ex: CPF/CNPJ, nome, tipo de conta).
2. O serviço `account-service` valida os dados (ex: formato válido de CPF/CNPJ, regras de negócio).
3. Verifica se já existe uma conta ativa para o mesmo CPF/CNPJ no repositório (`AccountRepo`).
4. Se existir, retorna HTTP 409 Conflict indicando que já existe conta para o cliente.
5. Se não existir:

   * Cria a entidade **Conta** com saldo inicial zero.
   * Persiste a conta no banco relacional Aurora PostgreSQL.
   * Retorna HTTP 201 Created com dados da conta criada.

---

### 2. Depósito em Conta

1. O cliente envia uma requisição POST para `/accounts/{accountId}/deposit` com o valor a ser depositado.
2. O serviço valida se a conta existe e está ativa.
3. Cria uma entidade **Transação** do tipo depósito com status inicial `pendente`.
4. Atualiza o saldo da conta somando o valor depositado.
5. Persiste a transação e o novo saldo no banco.
6. Publica um evento Kafka `account.deposited` com os dados da transação para outros serviços consumirem.
7. Retorna HTTP 200 OK com detalhes da transação.

---

### 3. Saque de Conta

1. O cliente envia uma requisição POST para `/accounts/{accountId}/withdraw` com o valor do saque.
2. O serviço valida se a conta existe, está ativa e possui saldo suficiente.
3. Se saldo insuficiente, retorna HTTP 400 Bad Request com mensagem apropriada.
4. Se saldo suficiente:

   * Cria uma entidade **Transação** do tipo saque com status `pendente`.
   * Atualiza o saldo subtraindo o valor solicitado.
   * Persiste a transação e o novo saldo.
   * Publica evento Kafka `account.withdrawn`.
   * Retorna HTTP 200 OK com detalhes da transação.

---

### 4. Transferência Interna entre Contas

1. O cliente envia requisição POST para `/accounts/transfer` com dados: conta origem, conta destino, valor.
2. O serviço valida:

   * Ambas as contas existem e estão ativas.
   * A conta origem possui saldo suficiente.
3. Se alguma validação falhar, retorna erro HTTP 400 Bad Request.
4. Se válido:

   * Cria duas entidades **Transação**: uma de débito (origem) e uma de crédito (destino), ambas com status `pendente`.
   * Atualiza saldos das duas contas.
   * Persiste as transações e saldos atualizados.
   * Publica eventos Kafka `account.withdrawn` e `account.deposited`.
   * Retorna HTTP 200 OK com resumo da transferência.

---

### 5. Consulta de Saldo e Histórico

1. O cliente faz requisição GET para `/accounts/{accountId}/balance` para consultar saldo atual.
2. O serviço retorna saldo atualizado consultando a base relacional.

---

### Observações Gerais

* Todas as operações devem ser atômicas e transacionais para garantir consistência.
* Eventos Kafka são usados para integração com outros serviços (ex: loan-service para débito automático, audit logs).
* Regras de negócio e validações são centralizadas no domínio (`domain/`).
* Operações devem emitir logs estruturados para auditoria e monitoramento.

---

## 3. Mapa de Eventos Kafka

| Evento              | Tópico Kafka           | Descrição                           | Consumidores                        |
| ------------------- | ---------------------- | ----------------------------------- | ----------------------------------- |
| DepositRequested    | `deposit-requested`    | Pedido de depósito iniciado         | pix-service, audit-service          |
| DepositCompleted    | `deposit-completed`    | Depósito confirmado                 | notification-service, audit-service |
| WithdrawalRequested | `withdrawal-requested` | Pedido de saque iniciado            | audit-service                       |
| WithdrawalCompleted | `withdrawal-completed` | Saque confirmado                    | notification-service, audit-service |
| TransferCompleted   | `transfer-completed`   | Transferência concluída com sucesso | audit-service, notification-service |

---

## 4. Observações importantes

* Saldo **deve ser atualizado de forma atômica** para evitar condições de corrida.
* Transações financeiras seguem o princípio de consistência eventual com eventos Kafka.
* Validar saldo antes de realizar saques ou transferências.
* Registrar todos eventos relevantes para auditoria.
* API deve expor endpoints REST seguros e com autenticação JWT.
