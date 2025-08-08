# 📝 TODO - Desenvolvimento Completo FinBank Backend

## 1. Planejamento e Setup Inicial

- [ ] Criar repositório monorepo no GitHub  
- [ ] Configurar padrão de branches (main, develop, feature/*, hotfix/*)  
- [ ] Configurar templates para PRs e issues  
- [ ] Definir padrão de commits: **Conventional Commits**  
- [ ] Configurar ferramentas de lint (ESLint + Prettier) e formato de código  
- [ ] Configurar CI inicial no GitHub Actions com:  
  - [ ] Execução de testes unitários  
  - [ ] Análise estática (SonarCloud/SonarQube)  
  - [ ] Verificação de lint  
  - [ ] Geração de cobertura de testes  
- [ ] Definir stack tecnológica (NestJS, TypeScript, AWS SDK, KafkaJS, Terraform)  
- [ ] Configurar docker-compose para ambiente local (Postgres, Kafka, Redis, DynamoDB local, etc)

---

## 2. Estrutura do Monorepo

- [ ] Criar pastas `apps/` e `libs/`  
- [ ] Para cada microserviço, criar estrutura DDD + Hexagonal:  
  - [ ] `domain/` (entidades, serviços, interfaces)  
  - [ ] `application/` (casos de uso, serviços de aplicação)  
  - [ ] `infrastructure/` (implementações concretas, AWS clients, repositórios)  
  - [ ] `interfaces/` (controllers, gateways, middlewares)  
- [ ] Criar projeto base para:  
  - [ ] auth-service  
  - [ ] account-service  
  - [ ] loan-service  
  - [ ] pix-service  
  - [ ] bank-reconciliation-service  
  - [ ] open-banking-service  
  - [ ] legacy-migration-service  
- [ ] Configurar build e scripts para cada app (build, start, test, lint)  
- [ ] Configurar compartilhamento de código/utilitários em libs (ex: logger, DTOs, exceptions)  

---

## 3. Desenvolvimento de Microsserviços (por ordem de prioridade)

### 3.1 auth-service

- [ ] Modelar entidades de usuário e sessão (Domain)  
- [ ] Integrar com AWS Cognito para autenticação e MFA  
- [ ] Implementar casos de uso:  
  - [ ] Registrar usuário  
  - [ ] Login com MFA  
  - [ ] Resetar senha  
- [ ] Criar controllers REST para endpoints  
- [ ] Testes unitários e integração (mock Cognito)  
- [ ] Setup de cache e sessão com Redis  

### 3.2 account-service

- [ ] Modelar entidades: Conta, Saldo, Transação  
- [ ] Casos de uso:  
  - [ ] Criar conta (validação CPF/CNPJ)  
  - [ ] Depositar valor  
  - [ ] Sacar valor (validação saldo)  
  - [ ] Transferência interna  
- [ ] Persistência com Aurora PostgreSQL  
- [ ] Publicar eventos no Kafka para cada transação realizada  
- [ ] Testes e validações  

### 3.3 loan-service

- [ ] Modelar entidades: Proposta, Contrato, Parcela  
- [ ] Casos de uso:  
  - [ ] Solicitar empréstimo  
  - [ ] Aprovar empréstimo (regras de negócio + score)  
  - [ ] Registrar pagamento de parcela  
- [ ] Integração com account-service para débito/crédito  
- [ ] Auditoria com AWS QLDB  
- [ ] Testes unitários e integração  

### 3.4 pix-service

- [ ] Configurar streams DynamoDB para capturar transações PIX  
- [ ] Criar AWS Lambda para processamento de eventos  
- [ ] Casos de uso: envio, recebimento, status da transação  
- [ ] Persistência em DynamoDB + integração com Kafka  
- [ ] Testes end-to-end  

### 3.5 bank-reconciliation-service

- [ ] Processar arquivos CNAB armazenados no S3  
- [ ] Batch processing com AWS Batch para conciliação  
- [ ] Atualizar base Aurora com resultados  
- [ ] Notificação em caso de inconsistências  
- [ ] Testes de integração  

### 3.6 open-banking-service

- [ ] Consumir APIs do Open Banking via gRPC e REST  
- [ ] Implementar adaptação de dados para domínio interno  
- [ ] Autenticação via OAuth2 e refresh token  
- [ ] Testes e validações  

### 3.7 legacy-migration-service

- [ ] Consumir serviços SOAP legados  
- [ ] Criar APIs REST modernas para os mesmos dados/fluxos  
- [ ] Implementar adapter para migração gradual  
- [ ] Testes de contrato  

---

## 4. Infraestrutura

- [ ] Criar scripts Terraform para provisionar:  
  - [ ] MSK (Kafka) cluster  
  - [ ] Aurora PostgreSQL  
  - [ ] DynamoDB tabelas  
  - [ ] Cognito User Pool  
  - [ ] S3 Buckets  
  - [ ] Lambda functions  
  - [ ] AWS Batch jobs  
- [ ] Configurar roles e permissões IAM para cada serviço  
- [ ] Configurar parâmetros e segredos (AWS Parameter Store / Secrets Manager)  
- [ ] Configurar CloudWatch para logs, métricas e alertas  

---

## 5. Testes e Qualidade

- [ ] Cobertura mínima: 90% em unitários e integração  
- [ ] Testes automatizados rodando no CI (GitHub Actions)  
- [ ] Revisão obrigatória de PRs com aprovação  
- [ ] Análise estática com SonarQube  
- [ ] Documentação automática de API com Swagger / OpenAPI  

---

## 6. Segurança e Compliance

- [ ] Criptografia em trânsito (TLS) e repouso (KMS)  
- [ ] Garantir conformidade com LGPD (tratamento e consentimento de dados)  
- [ ] Implementar controle de acesso baseado em funções (RBAC)  
- [ ] Auditoria imutável com QLDB para transações críticas  
- [ ] Monitorar logs de acesso e eventos suspeitos  

---

## 7. Deploy e Monitoramento

- [ ] Configurar pipelines CI/CD no GitHub Actions  
- [ ] Deploy automático para staging  
- [ ] Deploy manual para produção com aprovação  
- [ ] Monitoramento com CloudWatch dashboards e alarms  
- [ ] Alertas via SNS/Slack para incidentes  
- [ ] Logs centralizados e rastreamento distribuído  
