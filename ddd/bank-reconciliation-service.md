# bank-reconciliation-service — Modelagem e Fluxos

---

## Entidades Principais

* **ReconciliationFile**
  Representa o arquivo CNAB recebido para conciliação bancária.
  Atributos: id, nomeArquivo, dataUpload, status (PENDING, PROCESSING, PROCESSED, FAILED), tamanho, urlS3

* **ReconciliationBatch**
  Processo batch que analisa um arquivo CNAB e atualiza dados financeiros.
  Atributos: id, arquivoId, dataProcessamento, status (RUNNING, COMPLETED, FAILED), registrosProcessados, registrosComErro

* **BankTransaction**
  Transação bancária conciliada extraída do arquivo CNAB.
  Atributos: id, batchId, tipo (CRÉDITO, DÉBITO), valor, dataTransacao, contaOrigem, contaDestino, status (RECONCILED, PENDING, ERROR), descrição

* **Report**
  Relatório gerado após a conciliação.
  Atributos: id, batchId, tipoRelatorio, dataGeracao, urlS3

---

## Casos de Uso

* Upload de arquivo CNAB para S3
* Detecção automática de novo arquivo e início do processamento batch
* Processar arquivo CNAB e extrair transações bancárias
* Atualizar base Aurora com status e dados financeiros conciliados
* Gerar relatórios detalhados (erros, saldo, transações)
* Notificar sistemas interessados sobre o resultado da conciliação
* Reprocessar arquivo em caso de falhas
* Consultar status e histórico de conciliações

---

## Fluxo Completo Descritivo — bank-reconciliation-service

---

### 1. Upload do Arquivo CNAB

* Um sistema ou usuário faz upload de um arquivo CNAB para o bucket S3 configurado para conciliações.
* O upload gera um evento S3 (ex: `s3:ObjectCreated:Put`) que é detectado pelo serviço (trigger AWS Lambda ou AWS Batch).
* O serviço registra um novo `ReconciliationFile` no banco Aurora com status `PENDING` e armazena a URL do arquivo no S3.

---

### 2. Início do Processamento Batch

* Um job AWS Batch (ou Lambda orchestrador) é disparado para processar o arquivo.
* O status do `ReconciliationFile` muda para `PROCESSING`.
* O job faz o download do arquivo CNAB do S3 para um ambiente temporário.

---

### 3. Processamento e Extração dos Dados

* O batch lê o arquivo CNAB, linha a linha, convertendo em registros `BankTransaction`.
* Cada transação é validada quanto à consistência (ex: formato, valores, contas).
* Para cada transação válida, um registro `BankTransaction` é criado com status `RECONCILED`.
* Transações inválidas ou com problemas recebem status `ERROR` e são registradas para análise.

---

### 4. Atualização da Base de Dados

* Após o processamento completo do arquivo, o serviço atualiza o status do `ReconciliationBatch` para `COMPLETED` ou `FAILED` conforme o resultado.
* O serviço atualiza o status do `ReconciliationFile` para refletir o término do processamento.
* Todas as transações conciliadas são persistidas na base Aurora para uso futuro (relatórios, auditoria).

---

### 5. Geração e Armazenamento de Relatórios

* Com base nas transações processadas, o sistema gera relatórios detalhados:

  * Resumo financeiro (saldos, totais crédito/débito)
  * Relatório de erros e inconsistências
  * Histórico de conciliações
* Os relatórios são armazenados em formato PDF/CSV no bucket S3, com seus metadados registrados em `Report`.

---

### 6. Notificações e Integrações

* O serviço publica eventos Kafka para notificar outros microsserviços sobre o status da conciliação.
* Pode integrar com sistemas de alerta ou dashboards para exibir os resultados.
* Em caso de falhas, sistemas externos são notificados para possível intervenção manual.

---

### 7. Reprocessamento

* O usuário ou sistema pode solicitar o reprocessamento de um arquivo `FAILED` ou `COMPLETED` para corrigir problemas.
* O fluxo de processamento é reexecutado, atualizando registros conforme necessário.

---

### 8. Consultas e Histórico

* APIs permitem consultar o status e detalhes dos arquivos CNAB processados.
* Permitem análise histórica para auditoria e compliance.
