# legacy-migration-service — Modelagem e Fluxos

---

## Entidades Principais

* **LegacyEndpoint**
  Representa um endpoint SOAP legado.
  Atributos: id, url, métodoSOAP, descrição, status

* **RestEndpoint**
  Endpoint REST moderno que expõe os mesmos dados ou funcionalidades do legado.
  Atributos: id, url, métodoHTTP, descrição, status

* **MigrationAdapter**
  Componente responsável por adaptar requisições e respostas entre SOAP e REST.
  Atributos: id, nome, versão, estado (ativo/inativo)

* **MigrationRequest**
  Representa uma requisição recebida do cliente para o sistema legado via REST.
  Atributos: id, endpointId, payload, timestamp, status (pendente, processando, concluído, erro)

* **MigrationResponse**
  Representa a resposta convertida do legado para o formato REST.
  Atributos: id, migrationRequestId, payload, timestamp, status

---

## Casos de Uso

* Expor APIs REST modernas que simulam funcionalidades legadas SOAP
* Adaptar chamadas REST para SOAP e encaminhar para sistemas antigos
* Processar e adaptar respostas SOAP para formato REST para os consumidores
* Permitir migração gradual, mantendo compatibilidade com sistemas legados
* Registrar logs detalhados de chamadas, erros e tempo de resposta
* Monitorar a saúde dos endpoints legados e do serviço de migração
* Configurar timeouts, retries e fallback para garantir resiliência
* Facilitar futuras desativação dos sistemas legados com mínima interrupção

---

## Fluxo Completo Descritivo — legacy-migration-service

---

### 1. Recepção da Requisição REST

* Cliente (outro microsserviço ou aplicação externa) faz uma requisição HTTP REST para o **legacy-migration-service**, solicitando uma operação originalmente implementada no sistema legado SOAP.
* O serviço valida o payload e autentica a chamada conforme políticas internas.

---

### 2. Adaptação da Requisição para SOAP

* O **MigrationAdapter** converte a requisição REST para o formato XML SOAP esperado pelo sistema legado.
* Parâmetros e cabeçalhos são mapeados conforme o contrato SOAP.

---

### 3. Envio da Requisição SOAP

* O serviço envia a requisição SOAP para o endpoint legado configurado.
* Pode usar clientes SOAP específicos com suporte a WS-Security, se necessário.

---

### 4. Recepção da Resposta SOAP

* O sistema legado responde com um XML SOAP contendo os dados solicitados ou resultado da operação.
* O serviço captura essa resposta para processamento.

---

### 5. Adaptação da Resposta para REST

* O **MigrationAdapter** transforma a resposta SOAP em um objeto JSON compatível com a API REST do FinBank.
* Campos são mapeados e convertidos, incluindo tratamento de erros e códigos de status.

---

### 6. Resposta ao Cliente

* O serviço retorna ao cliente a resposta REST, completando o ciclo.
* Em caso de erro no sistema legado, mensagens amigáveis são geradas e códigos HTTP apropriados retornados.

---

### 7. Logs, Monitoramento e Resiliência

* Todas as requisições e respostas são logadas para auditoria e troubleshooting.
* Métricas de latência, taxa de erro e disponibilidade são monitoradas.
* Implementa políticas de retry e circuit breaker para mitigar falhas temporárias.

---

### 8. Migração Gradual e Atualizações

* Novos endpoints REST podem ser expostos conforme funcionalidades do legado são migradas para serviços modernos.
* A arquitetura permite desativar gradualmente endpoints SOAP legados sem impactar os clientes REST.
