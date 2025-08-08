# open-banking-service — Modelagem e Fluxos

---

## Entidades Principais

* **BankProvider**
  Representa um provedor de dados financeiros externo conectado via APIs Open Banking.
  Atributos: id, nome, endpointGrpc, tokenOAuth, statusConexao, ultimoSync

* **AccountData**
  Dados de conta bancária obtidos via Open Banking.
  Atributos: id, providerId, contaNumero, tipoConta, saldo, moeda, status

* **TransactionData**
  Transações financeiras associadas às contas bancárias externas.
  Atributos: id, accountId, dataTransacao, valor, tipo (CRÉDITO, DÉBITO), descrição, categoria

* **OAuthToken**
  Token OAuth2 para autenticação junto ao provedor Open Banking.
  Atributos: id, providerId, accessToken, refreshToken, expiresAt

* **SyncJob**
  Registro de sincronização periódica dos dados bancários.
  Atributos: id, providerId, dataInicio, dataFim, status (RUNNING, COMPLETED, FAILED), registrosSincronizados, erros

---

## Casos de Uso

* Registrar e configurar provedores Open Banking (credenciais, endpoints gRPC)
* Autenticar via OAuth2 e manter tokens atualizados (refresh token)
* Sincronizar dados bancários periodicamente (contas, transações)
* Adaptar e transformar dados externos para modelo interno do domínio
* Expor APIs REST/gRPC para outros microsserviços consumirem dados financeiros
* Gerenciar erros e falhas de conexão com provedores
* Monitorar histórico de sincronizações e gerar relatórios
* Garantir segurança e conformidade (LGPD, criptografia)

---

## Fluxo Completo Descritivo — open-banking-service

---

### 1. Registro e Configuração do Provedor

* O administrador do sistema cadastra um novo **BankProvider** com as credenciais OAuth2 e endpoints gRPC da API externa.
* Configura parâmetros como escopos, URL do token OAuth, URLs das APIs gRPC, e políticas de sincronização.

---

### 2. Autenticação e Gestão de Tokens OAuth2

* O serviço inicia o fluxo OAuth2 para obter o `accessToken` e `refreshToken` do provedor externo.
* Os tokens são armazenados em `OAuthToken` com suas datas de expiração.
* Periodicamente, o serviço verifica a validade do token e realiza refresh automático antes do vencimento para garantir acesso contínuo.

---

### 3. Sincronização dos Dados Bancários

* Um job agendado (cron, AWS Lambda, ou container) inicia o processo de sincronização (`SyncJob`) para cada provedor ativo.
* O serviço realiza chamadas gRPC para a API do provedor usando o `accessToken` para autenticação.
* Busca dados de contas bancárias (`AccountData`) e transações (`TransactionData`).
* Dados recebidos são adaptados e normalizados para o modelo interno do domínio, para uso pelos microsserviços FinBank.

---

### 4. Persistência dos Dados

* Dados financeiros normalizados são salvos em banco de dados (Aurora ou outro persistente).
* O status do `SyncJob` é atualizado conforme progresso (RUNNING, COMPLETED, FAILED).

---

### 5. Exposição de APIs para Consulta

* O serviço expõe endpoints REST e gRPC para outros microsserviços ou sistemas consumidores consultarem dados sincronizados.
* As APIs suportam filtros por conta, período, categoria e tipo de transação.

---

### 6. Tratamento de Erros e Alertas

* Em caso de falha na conexão ou erro na API, o serviço registra logs detalhados.
* O `SyncJob` é marcado como `FAILED` e pode gerar alertas para equipes de suporte.
* Políticas de retry são aplicadas para tentativas automáticas.

---

### 7. Segurança e Compliance

* Todas as comunicações com provedores externos são via TLS/SSL.
* Dados sensíveis são criptografados em repouso.
* Logs e dados pessoais seguem diretrizes LGPD, incluindo anonimização quando necessário.

---

### 8. Consultas e Relatórios

* APIs permitem consultas detalhadas dos dados sincronizados.
* Relatórios de sincronização (volume, erros, duração) são gerados para análise operacional e auditoria.
