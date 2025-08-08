# 📦 pix-service — Modelagem Completa DDD

---

## 1. Entidades (Domain Layer)

### PixTransaction

| Propriedade       | Tipo    | Descrição                                      |
| ----------------- | ------- | ---------------------------------------------- |
| id                | UUID    | Identificador único da transação               |
| originKey         | string  | Chave PIX do pagador (origem)                  |
| destinationKey    | string  | Chave PIX do recebedor (destino)               |
| amount            | number  | Valor da transação                             |
| status            | enum    | `PENDING`, `PROCESSING`, `COMPLETED`, `FAILED` |
| createdAt         | Date    | Data/hora da criação                           |
| updatedAt         | Date    | Data/hora da última atualização                |
| externalReference | string? | Referência externa (ex: banco)                 |
| failureReason     | string? | Motivo do erro (se `FAILED`)                   |

---

## 2. Casos de Uso (Application Layer)

* **CreatePixTransaction**
  Valida dados e cria nova transação com status `PENDING`.

* **ProcessPixTransaction**
  Atualiza status para `PROCESSING`, executa validações e envia evento para Kafka.

* **CompletePixTransaction**
  Atualiza status para `COMPLETED`, registra timestamp, notifica outros microsserviços.

* **FailPixTransaction**
  Atualiza status para `FAILED`, registra motivo e timestamps.

* **GetPixTransactionStatus**
  Retorna status e detalhes da transação para consulta via API.

---

## 3. Diagrama de Classes (Resumo)

* **PixTransaction** (Entity)

  * Métodos: `validate()`, `updateStatus()`, `toEventPayload()`

* **PixTransactionRepository** (Interface)

  * Métodos: `save()`, `findById()`, `update()`

* **PixTransactionService** (Application Service)

  * Métodos: `createTransaction()`, `processTransaction()`, `completeTransaction()`, `failTransaction()`

* **PixProducer** (Infraestrutura)

  * Método: `publish(event)`

* **PixConsumer** (Infraestrutura)

  * Método: `consume(event)`

---

# Diagrama de Sequência — pix-service

---

### Fluxo 1: Criação de Transação PIX

1. Cliente faz requisição HTTP POST para `/pix/send` com os dados: chave origem, chave destino e valor.
2. Pix API recebe a requisição e chama o serviço de domínio para criar a transação.
3. O serviço valida os dados recebidos.
4. O serviço salva a transação no banco DynamoDB com status `PENDING`.
5. DynamoDB registra o novo item e dispara evento no DynamoDB Stream.
6. API responde ao cliente com status 202 (Aceito), indicando que a transação está pendente de processamento.

---

### Fluxo 2: Processamento Assíncrono via Lambda

1. AWS Lambda é disparada pelo evento do DynamoDB Stream.
2. Lambda chama o serviço Pix para processar o evento.
3. Serviço atualiza o status da transação para `PROCESSING` no banco.
4. Serviço publica evento `PixTransactionUpdated` no Kafka, indicando o processamento iniciado.
5. Lambda finaliza com sucesso ou erro, podendo reprocessar se necessário.

---

### Fluxo 3: Consumo do Evento Kafka e Atualização Final

1. Serviço Pix Consumer escuta o tópico Kafka `PixTransactionUpdated`.
2. Ao receber o evento, o serviço processa regras de negócio e validações.
3. Se a transação for aprovada, atualiza status para `COMPLETED`.
4. Caso ocorra erro, atualiza status para `FAILED` e registra o motivo.
5. Atualizações são refletidas no DynamoDB.
6. Serviço publica evento `AccountBalanceUpdated` para informar o account-service.
7. Cliente pode consultar o status via API.

---

## 5. Eventos Kafka e Integrações

| Evento                  | Descrição                                         | Consumidores                                 |
| ----------------------- | ------------------------------------------------- | -------------------------------------------- |
| `PixTransactionCreated` | Nova transação PIX criada                         | pix-service (interno), audit                 |
| `PixTransactionUpdated` | Status atualizado (PROCESSING, COMPLETED, FAILED) | account-service, notification-service, audit |
| `AccountBalanceUpdated` | Atualização de saldo após PIX                     | -                                            |

---

## 6. Considerações Extras

* **Tabelas DynamoDB** configuradas com Streams ativados para eventos em tempo real.
* **Lambda** tem retry configurado para falhas transitórias, e DLQ para falhas persistentes.
* **Kafka** usado para propagação e desacoplamento de eventos para demais microsserviços.
* **API REST** expõe endpoints para criar e consultar transações PIX.
* **Segurança**: validação de payloads, autenticação via tokens JWT, e auditoria.

