# Emoframe API

API Node.js/Express para registrar submissões de instrumentos de avaliação da Emoframe, com validação via Zod e persistência em PostgreSQL usando Drizzle ORM.

## Stack

- Node.js com TypeScript
- Express
- Zod
- Drizzle ORM
- PostgreSQL
- Docker Compose

## Rotas

Health check:

```text
GET /health
```

Instrumentos disponíveis:

```text
/api/sus
/api/panas
/api/leap
/api/sam
/api/eaz
/api/gds
/api/brums
/api/px
/api/gamex
/api/iuxrv
```

As rotas protegidas esperam autenticação por bearer token e bloqueiam o papel `guest`.

Exemplo:

```bash
curl -i -X POST http://localhost:3000/api/sus/submissions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer SEU_TOKEN_AQUI" \
  -H "user-role: user" \
  -d '{
    "applicationId": "app-test",
    "evaluationId": "eval-test",
    "externalUserId": "user-test",
    "answers": {
      "use_frequency": 5,
      "use_complex": 2,
      "use_easy": 4,
      "need_help": 1,
      "function_integration": 4,
      "inconsistency": 2,
      "learning_curve": 4,
      "jumbled": 1,
      "confidence": 5,
      "learn_system": 4
    }
  }'
```

## Configuração

Crie o arquivo `.env` a partir do exemplo:

```bash
cp .env.example .env
```

Variáveis esperadas:

```text
POSTGRES_USER=
POSTGRES_DB=
POSTGRES_PASSWORD=
PORT=
TEST_BEARER_TOKEN=
```

Para execução local fora do Docker, a aplicação também usa as configurações de conexão com o Postgres definidas em `src/db/connection.ts`.

## Desenvolvimento

Instale as dependências:

```bash
npm install
```

Rode em modo desenvolvimento:

```bash
npm run dev
```

Gere o build TypeScript:

```bash
npm run build
```

Inicie a versão compilada:

```bash
npm start
```

## Banco de dados

O schema principal fica em:

```text
src/db/schema.ts
```

Gerar migrations Drizzle:

```bash
npm run db:generate
```

Aplicar o schema no banco configurado:

```bash
npm run db:push
```

## Docker

Subir a API, o Postgres e o serviço de migração:

```bash
docker compose up -d --build
```

Verificar containers:

```bash
docker compose ps
```

Ver logs:

```bash
docker compose logs api
docker compose logs migrate
```

## Deploy com Nginx

O guia de atualização do container e publicação via Nginx está em:
[Guia](docs/CONTAINER_NGINX.md)
