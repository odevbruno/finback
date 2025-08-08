# loan-service — Modelagem e Fluxos

---

## Entidades Principais

* **LoanProposal**
  Representa a proposta de empréstimo feita pelo cliente.
  Atributos: id, clienteId, valorSolicitado, prazoMeses, status (ex: PENDING, APPROVED, REJECTED), dataCriacao, dataAtualizacao, scoreCliente

* **LoanContract**
  Contrato formal do empréstimo aprovado.
  Atributos: id, proposalId, clienteId, valorAprovado, taxaJuros, parcelas (lista de Parcelas), status (ACTIVE, PAID, DEFAULTED), dataAssinatura, dataVencimento

* **Installment (Parcela)**
  Cada parcela do empréstimo.
  Atributos: id, contractId, numeroParcela, valorParcela, dataVencimento, dataPagamento (nullable), status (PENDING, PAID, LATE)

* **Cliente** (referência ao serviço de auth/account)
  Atributos relevantes: id, nome, CPF, scoreCredito

---

## Casos de Uso

* Solicitar empréstimo
* Avaliar proposta e calcular score
* Aprovar ou rejeitar empréstimo
* Gerar contrato com parcelas
* Registrar pagamento de parcela
* Consultar status do empréstimo e parcelas
* Enviar notificações e eventos para account-service e auditoria

---

## Fluxo Completo Descritivo — loan-service

---

### 1. Solicitação de Empréstimo

* O cliente envia via API uma solicitação de empréstimo, contendo valor desejado, prazo e dados pessoais.
* O serviço valida dados básicos (ex: campos obrigatórios, formato).
* Uma nova entidade `LoanProposal` é criada com status `PENDING` e salva no banco (Aurora).
* O sistema inicia a análise de crédito, podendo consultar score no serviço de auth/account (ou sistema externo).
* A proposta fica aguardando análise.

---

### 2. Avaliação e Decisão

* O serviço de avaliação pega a `LoanProposal` pendente e aplica regras de negócio, calculando score e risco.
* Baseado no score e políticas internas, a proposta é aprovada ou rejeitada:

  * Se aprovada, status da proposta muda para `APPROVED`.
  * Se rejeitada, status muda para `REJECTED`.
* Um evento Kafka `LoanProposalEvaluated` é publicado com o resultado.

---

### 3. Geração de Contrato

* Quando a proposta é aprovada, o serviço gera um `LoanContract`.
* O contrato inclui detalhes: valor aprovado, taxa de juros, número de parcelas, datas de vencimento.
* As parcelas (`Installment`) são criadas com status `PENDING`.
* Contrato é salvo no banco e status atualizado para `ACTIVE`.
* Evento Kafka `LoanContractCreated` é publicado.

---

### 4. Pagamento de Parcela

* O cliente realiza pagamento de parcela via API (ou integração com account-service).
* O serviço verifica se o pagamento corresponde a uma parcela pendente.
* A parcela é marcada como `PAID` e dataPagamento é atualizada.
* Se pagamento atrasado, atualiza status para `LATE` e pode gerar multa/juros.
* Evento Kafka `InstallmentPaid` é emitido.
* O serviço pode atualizar saldo no account-service via evento.

---

### 5. Monitoramento e Encerramento

* O sistema monitora o contrato para verificar se todas as parcelas foram pagas.
* Se todas as parcelas estiverem pagas, o contrato muda status para `PAID`.
* Em caso de inadimplência prolongada, contrato pode mudar para `DEFAULTED`.
* Eventos correspondentes são publicados no Kafka para alertas e auditoria.

---

### 6. Consultas e Relatórios

* APIs para consulta de status de propostas, contratos e parcelas.
* Geração de relatórios de empréstimos ativos, pagamentos realizados, inadimplências.
* Logs e auditoria com AWS QLDB para garantir rastreabilidade.

---
