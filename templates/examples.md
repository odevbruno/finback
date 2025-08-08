# 🚀 Guia Completo: Deploy e Código Base para Backend Fintech com AWS + NestJS + Kafka

---

## 1. Infraestrutura AWS via Terraform (resumo)

* VPC, Subnets, Security Groups
* Aurora PostgreSQL Cluster + Instances
* MSK Cluster (Kafka)
* DynamoDB tabelas
* Lambda functions
* Cognito User Pool + App Client
* Redis (Elasticache) — *por simplicidade pode usar Redis local em dev*
* S3 Buckets
* Batch Jobs para conciliação
* IAM Roles, Policies e Secrets Manager para credenciais

*(Já falamos do Terraform detalhado, que cria tudo isso)*

---

## 2. Projeto NestJS (Monorepo) Setup Básico

* `apps/` para microsserviços (ex: auth-service, pix-service, account-service)
* `libs/` para utilitários compartilhados (ex: logger, dto, exceptions)
* Cada serviço com estrutura:

  * `domain/` (entidades, interfaces)
  * `application/` (use cases)
  * `infrastructure/` (AWS SDK clients, repositórios, adapters)
  * `interfaces/` (controllers, gateways, eventos)

---

## 3. Exemplo: **auth-service** - Cognito + Redis

### a) Provider Redis (libs/redis/redis.module.ts)

```typescript
import { Module } from '@nestjs/common';
import { RedisModule as NestRedisModule } from '@nestjs-modules/ioredis';

@Module({
  imports: [
    NestRedisModule.forRoot({
      config: {
        host: process.env.REDIS_HOST,
        port: +process.env.REDIS_PORT,
      },
    }),
  ],
  exports: [NestRedisModule],
})
export class RedisModule {}
```

### b) Serviço Cognito (apps/auth-service/src/infrastructure/cognito.service.ts)

```typescript
import { Injectable } from '@nestjs/common';
import {
  CognitoIdentityProviderClient,
  InitiateAuthCommand,
  RespondToAuthChallengeCommand,
} from '@aws-sdk/client-cognito-identity-provider';

@Injectable()
export class CognitoService {
  private client = new CognitoIdentityProviderClient({ region: process.env.AWS_REGION });
  private userPoolClientId = process.env.COGNITO_CLIENT_ID;

  async login(username: string, password: string) {
    const command = new InitiateAuthCommand({
      AuthFlow: 'USER_PASSWORD_AUTH',
      ClientId: this.userPoolClientId,
      AuthParameters: { USERNAME: username, PASSWORD: password },
    });
    const response = await this.client.send(command);

    if (response.ChallengeName === 'SMS_MFA' || response.ChallengeName === 'SOFTWARE_TOKEN_MFA') {
      return { challenge: response.ChallengeName, session: response.Session };
    }
    return { tokens: response.AuthenticationResult };
  }

  async respondToMfa(session: string, code: string) {
    const command = new RespondToAuthChallengeCommand({
      ClientId: this.userPoolClientId,
      ChallengeName: 'SMS_MFA', // ou SOFTWARE_TOKEN_MFA
      Session: session,
      ChallengeResponses: { SMS_MFA_CODE: code },
    });
    const response = await this.client.send(command);
    return response.AuthenticationResult;
  }
}
```

### c) Serviço Redis para sessão MFA (apps/auth-service/src/infrastructure/session.service.ts)

```typescript
import { Injectable } from '@nestjs/common';
import { RedisService } from '@nestjs-modules/ioredis';

@Injectable()
export class SessionService {
  private client = this.redisService.getClient();

  constructor(private readonly redisService: RedisService) {}

  async saveMfaSession(userId: string, session: string) {
    await this.client.set(`mfa-session:${userId}`, session, 'EX', 300);
  }

  async getMfaSession(userId: string): Promise<string | null> {
    return this.client.get(`mfa-session:${userId}`);
  }

  async clearMfaSession(userId: string) {
    await this.client.del(`mfa-session:${userId}`);
  }
}
```

### d) Controller simplificado (apps/auth-service/src/interfaces/auth.controller.ts)

```typescript
import { Controller, Post, Body } from '@nestjs/common';
import { CognitoService } from '../infrastructure/cognito.service';
import { SessionService } from '../infrastructure/session.service';

@Controller('auth')
export class AuthController {
  constructor(
    private readonly cognitoService: CognitoService,
    private readonly sessionService: SessionService,
  ) {}

  @Post('login')
  async login(@Body() body: { username: string; password: string }) {
    const result = await this.cognitoService.login(body.username, body.password);
    if (result.challenge) {
      await this.sessionService.saveMfaSession(body.username, result.session);
      return { mfaRequired: true, challenge: result.challenge };
    }
    return { tokens: result.tokens };
  }

  @Post('mfa')
  async respondMfa(@Body() body: { username: string; code: string }) {
    const session = await this.sessionService.getMfaSession(body.username);
    if (!session) throw new Error('Sessão MFA inválida ou expirada');
    const tokens = await this.cognitoService.respondToMfa(session, body.code);
    await this.sessionService.clearMfaSession(body.username);
    return { tokens };
  }
}
```

---

## 4. Exemplo: **pix-service** - Kafka + DynamoDB + Lambda

### a) Kafka Provider (libs/kafka/kafka.provider.ts)

```typescript
import { Kafka } from 'kafkajs';

export const kafkaProvider = {
  provide: 'KAFKA_CLIENT',
  useFactory: () => {
    return new Kafka({
      clientId: 'finbank-pix-service',
      brokers: [
        process.env.KAFKA_BROKER_1!,
        process.env.KAFKA_BROKER_2!,
      ],
      ssl: true,
      sasl: {
        mechanism: 'scram-sha-512',
        username: process.env.KAFKA_USERNAME!,
        password: process.env.KAFKA_PASSWORD!,
      },
    });
  },
};
```

### b) Producer e Consumer (apps/pix-service/src/infrastructure/pix-producer.service.ts e pix-consumer.service.ts)

Veja o exemplo detalhado do passo anterior, com producer e consumer KafkaJS.

### c) Lambda Function handler básico (apps/pix-service/src/infrastructure/lambda.handler.ts)

```typescript
import { Handler, Context, Callback } from 'aws-lambda';

export const handler: Handler = async (event, context: Context, callback: Callback) => {
  // Parse evento DynamoDB Stream (PIX transações)
  for (const record of event.Records) {
    if (record.eventName === 'INSERT') {
      const newImage = record.dynamodb?.NewImage;
      // lógica para processar transação PIX
    }
  }
  callback(null, `Processado ${event.Records.length} registros`);
};
```

---

## 5. CI/CD (GitHub Actions exemplo básico)

```yaml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: 18
      - run: npm install
      - run: npm run lint
      - run: npm run test -- --coverage
      - name: Upload coverage
        uses: actions/upload-artifact@v3
        with:
          name: coverage-report
          path: coverage/
```

---

## 6. Próximos Passos

* Completar casos de uso domain-driven para cada serviço
* Criar testes unitários e de integração completos
* Configurar monitoramento CloudWatch + alerts
* Automatizar deploy Terraform via pipeline
