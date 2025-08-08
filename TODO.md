# 📝 TODO - Desenvolvimento Completo FinBank Backend

---

## 0. Documentação Técnica Completa (Antes do Código)

- [ ] Criar diagramas de arquitetura gerais e específicos por microsserviço (Mermaid, Draw.io)  
- [ ] Modelagem DDD: bounded contexts, agregados, entidades, value objects  
- [ ] Definir eventos Kafka com schema, contratos e fluxo de mensagens  
- [ ] Descrever casos de uso detalhados para cada microsserviço, com regras de negócio e fluxos  
- [ ] Documentar políticas de segurança, compliance e auditoria (LGPD, BACEN, PCI DSS)  
- [ ] Criar guia de contribuição e padrões de codificação para o time  
- [ ] Validar roadmap de entregas e milestones

---

## 1. Planejamento e Setup Inicial

- [ ] Criar repositório monorepo no GitHub  
- [ ] Configurar padrão de branches (main, develop, feature/*, hotfix/*)  
- [ ] Criar templates para PRs e issues alinhados com o guia de contribuição  
- [ ] Definir padrão de commits: **Conventional Commits**  
- [ ] Configurar ferramentas de lint (ESLint + Prettier) e padronização de código  
- [ ] Configurar CI inicial no GitHub Actions com:  
  - [ ] Execução de testes unitários  
  - [ ] Análise estática (SonarCloud/SonarQube)  
  - [ ] Verificação de lint  
  - [ ] Geração de cobertura de testes mínima de 90%  
- [ ] Definir stack tecnológica (NestJS, TypeScript, AWS SDK, KafkaJS, Terraform)  
- [ ] Configurar docker-compose para ambiente local (Postgres, Kafka, Redis, DynamoDB local, etc)

---

## 2. Estrutura do Monorepo e Arquitetura

- [ ] Criar pastas `apps/` e `libs/`  
- [ ] Para cada microserviço, estruturar DDD + Arquitetura Hexagonal:  
  - [ ] `domain/` (entidades, agregados, interfaces de domínio)  
  - [ ] `application/` (casos de uso, serviços de aplicação)  
  - [ ] `infrastructure/` (implementações concretas, adaptadores AWS, repositórios)  
  - [ ] `interfaces/` (controllers REST/gRPC, gateways externos, middlewares)  
- [ ] Criar projetos base para todos os microsserviços:  
  - [ ] auth-service  
  - [ ] account-service  
  - [ ] loan-service  
  - [ ] pix-service  
  - [ ] bank-reconciliation-service  
  - [ ] open-banking-service  
  - [ ] legacy-migration-service  
- [ ] Configurar build, start, test, lint para cada serviço  
- [ ] Criar libs compartilhadas para utilitários, DTOs, logging, erros customizados  

---

## 3. Desenvolvimento de Microsserviços (por ordem de prioridade)

### 3.1 auth-service

- [ ] Modelar entidades (Usuário, Sessão, MFA) no domínio  
- [ ] Integrar com AWS Cognito para autenticação, registro e MFA  
- [ ] Implementar casos de uso: registro, login com MFA, reset de senha  
- [ ] Criar controllers REST para endpoints  
- [ ] Testes unitários e integração (mock AWS Cognito)  
- [ ] Implementar cache e sessão com Redis  

### 3.2 account-service

- [ ] Modelar entidades: Conta, Saldo, Transação  
- [ ] Implementar casos de uso: criação de conta, depósito, saque, transferência interna  
- [ ] Persistência em Aurora PostgreSQL  
- [ ] Publicar eventos Kafka para transações realizadas  
- [ ] Testes unitários e integração  

### 3.3 loan-service

- [ ] Modelar entidades: Proposta, Contrato, Parcela  
- [ ] Casos de uso: solicitação, aprovação (regras + score), pagamento de parcela  
- [ ] Integrar com account-service para débitos/créditos  
- [ ] Implementar auditoria imutável com AWS QLDB  
- [ ] Testes unitários e integração  

### 3.4 pix-service

- [ ] Configurar DynamoDB Streams para capturar transações PIX  
- [ ] Criar AWS Lambda para processamento de eventos e publicação em Kafka  
- [ ] Casos de uso: envio, recebimento e consulta status da transação PIX  
- [ ] Persistência em DynamoDB com consistência eventual  
- [ ] Testes end-to-end  

### 3.5 bank-reconciliation-service

- [ ] Receber e armazenar arquivos CNAB no S3  
- [ ] Processar arquivos via AWS Batch para conciliação bancária  
- [ ] Atualizar dados conciliados em Aurora  
- [ ] Gerar relatórios e alertas para inconsistências  
- [ ] Testes de integração e validação  

### 3.6 open-banking-service

- [ ] Consumir APIs externas Open Banking via gRPC e REST  
- [ ] Implementar adaptação dos dados para o modelo interno  
- [ ] Implementar autenticação OAuth2 com refresh tokens  
- [ ] Testes de integração e segurança  

### 3.7 legacy-migration-service

- [ ] Consumir serviços SOAP legados  
- [ ] Criar APIs REST modernas para os mesmos fluxos  
- [ ] Implementar adapters para tradução SOAP → REST  
- [ ] Testes de contrato  

---

## 4. Infraestrutura

- [ ] Criar scripts Terraform para provisionar recursos:  
  - [ ] MSK (Kafka) cluster  
  - [ ] Aurora PostgreSQL  
  - [ ] DynamoDB tabelas  
  - [ ] Cognito User Pool  
  - [ ] S3 buckets  
  - [ ] Lambda functions  
  - [ ] AWS Batch jobs  
- [ ] Configurar roles e permissões IAM específicas para cada serviço  
- [ ] Configurar parâmetros e segredos (AWS Parameter Store / Secrets Manager)  
- [ ] Configurar CloudWatch para logs, métricas, dashboards e alertas  

---

## 5. Testes e Qualidade

- [ ] Cobertura mínima de 90% em testes unitários e integração  
- [ ] Implementar testes end-to-end integrando microsserviços e eventos  
- [ ] Testar cenários assíncronos e de falha de mensagens Kafka  
- [ ] Testar performance e carga com volumes simulados  
- [ ] Análise estática e revisão obrigatória em PRs  
- [ ] Documentar APIs via Swagger / OpenAPI  

---

## 6. Segurança e Compliance

- [ ] Implementar criptografia em trânsito (TLS) e repouso (AWS KMS)  
- [ ] Garantir conformidade com LGPD: tratamento, consentimento, anonimização e eliminação de dados  
- [ ] Controlar acesso com RBAC e políticas granulares  
- [ ] Implementar auditoria imutável para operações críticas com AWS QLDB  
- [ ] Monitorar logs de acesso, tentativas suspeitas e anomalias  

---

## 7. Deploy e Monitoramento

- [ ] Configurar pipelines CI/CD no GitHub Actions para todos os microsserviços  
- [ ] Deploy automático para ambientes de staging  
- [ ] Deploy manual para produção com controle de acesso e aprovação  
- [ ] Monitorar serviços via CloudWatch dashboards e alarms  
- [ ] Configurar alertas em SNS, Slack ou sistemas de incidentes  
- [ ] Centralizar logs e rastreamento distribuído (ex: OpenTelemetry)  

---

# Finalização

- [ ] Documentar todo processo de desenvolvimento, deploy e monitoramento  
- [ ] Criar guias de onboarding para novos desenvolvedores  
- [ ] Planejar roadmap de melhorias e features futuras  
