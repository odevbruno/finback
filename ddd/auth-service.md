# auth-service - Documentação Técnica Completa

---

## 1. Diagrama de Classes (DDD)

### 1.1 Entidades

* **User**

  * id: UUID
  * username: string
  * email: string
  * passwordHash: string
  * status: enum (ACTIVE, BLOCKED, PENDING_CONFIRMATION)
  * roles: Role[] (admin, user, etc)
  * createdAt: Date
  * updatedAt: Date

* **Session**

  * id: UUID
  * userId: UUID
  * jwtToken: string
  * refreshToken: string
  * expiresAt: Date

* **MfaChallenge** (Value Object)

  * challengeType: enum (SMS, TOTP)
  * challengeCode: string
  * sessionId: UUID
  * expiresAt: Date

### 1.2 Interfaces (Ports)

* **IUserRepository**

  * findByUsername(username: string): Promise\<User | null>
  * create(user: User): Promise<User>
  * update(user: User): Promise<void>

* **ISessionRepository**

  * create(session: Session): Promise<Session>
  * findByToken(token: string): Promise\<Session | null>
  * delete(sessionId: UUID): Promise<void>

* **IMfaService**

  * sendChallenge(user: User, type: ChallengeType): Promise<MfaChallenge>
  * verifyChallenge(challenge: MfaChallenge, code: string): Promise<boolean>

---

## 2. Fluxos Detalhados - Auth Service

### 2.1 Fluxo de Criação de Conta

1. O cliente faz uma requisição HTTP POST para o endpoint `/register` enviando **username**, **email** e **password**.
2. O serviço `auth-service` consulta o repositório de usuários (`UserRepo`) para verificar se o username já existe.
3. Se o usuário já existir, o serviço retorna uma resposta HTTP 409 Conflict para o cliente, indicando que o nome já está em uso.
4. Se o usuário não existir:

   * O serviço chama a API do AWS Cognito para criar o usuário no pool de usuários.
   * Se a criação no Cognito for bem-sucedida, o serviço cria o registro do usuário no banco local (via `UserRepo`).
   * Após criar o registro local, o serviço retorna uma resposta HTTP 201 Created para o cliente, confirmando a criação da conta.

---

### 2.2 Fluxo de Login com MFA

1. O cliente faz uma requisição HTTP POST para o endpoint `/login` enviando **username** e **password**.
2. O serviço `auth-service` inicia o processo de autenticação chamando o Cognito (`InitiateAuth`).
3. O Cognito responde:

   * Se o login requer MFA, retorna um desafio com `ChallengeName` e um token `Session`.
   * Se o login não requer MFA, retorna diretamente os tokens JWT.
4. Caso o MFA seja requerido:

   * O serviço salva o token `Session` no cache Redis, associado ao usuário.
   * O serviço responde ao cliente com HTTP 200 MFA Required, informando que o código MFA deve ser enviado.
   * O cliente envia o código MFA via POST no endpoint `/mfa` com o username e o código.
   * O serviço busca o token `Session` no Redis.
   * O serviço chama o Cognito para validar o código MFA (`RespondToAuthChallenge`).
   * Se o código for válido, o Cognito retorna os tokens JWT.
   * O serviço limpa a sessão MFA no Redis.
   * O serviço retorna os tokens JWT ao cliente com HTTP 200 OK.
5. Caso o login não exija MFA, o serviço retorna os tokens JWT diretamente para o cliente com HTTP 200 OK.

---

## 3. Mapa de Eventos Kafka

O **auth-service** pode publicar eventos para os outros microsserviços para sinalizar ações importantes, tais como:

| Evento                  | Tópico Kafka         | Descrição                             | Consumidores                        |
| ----------------------- | -------------------- | ------------------------------------- | ----------------------------------- |
| UserRegistered          | `user-registered`    | Usuário criado com sucesso            | account-service, loan-service       |
| UserLoggedIn            | `user-logged-in`     | Login bem-sucedido                    | audit-service, notification-service |
| UserMfaChallengeStarted | `user-mfa-challenge` | Início do desafio MFA para um usuário | audit-service                       |

### Exemplo do payload JSON:

```json
{
  "userId": "uuid-do-usuario",
  "username": "usuario123",
  "timestamp": "2025-08-07T20:00:00Z",
  "eventType": "UserRegistered"
}
```

---

## 4. Observações importantes

* A autenticação principal utiliza **AWS Cognito** para gerenciar credenciais e MFA.
* A sessão temporária para MFA é armazenada no **Redis** para rápida consulta e expiração automática.
* A comunicação com Kafka deve ser assíncrona e idempotente.
* A responsabilidade do auth-service é **não armazenar senhas**, apenas a delegação para o Cognito.
