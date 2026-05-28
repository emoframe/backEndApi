# Atualizações da Emoframe API no container e publicação via Nginx

Este documento descreve como uma mudança de funcionalidade sai do código da Emoframe API, entra no container Docker e fica disponível publicamente em:

```text
https://emoframe.marcusrodrigues.dev/api/
```

## Visão geral do fluxo

A API roda em Node.js/Express dentro de um container Docker. O container não é exposto diretamente para a internet: o `docker-compose.yml` publica a porta somente no loopback do VPS, usando `127.0.0.1:${PORT}:${PORT}`.

O Nginx fica como ponto público de entrada. Ele recebe as requisições HTTPS do domínio `emoframe.marcusrodrigues.dev` e encaminha para a API local em `http://127.0.0.1:3000`.

Fluxo simplificado:

```text
Cliente
  -> https://emoframe.marcusrodrigues.dev/api/...
  -> Cloudflare
  -> Nginx no VPS
  -> http://127.0.0.1:3000/api/...
  -> Container api
  -> Container db/postgres
```

## Onde as funcionalidades entram no código

As rotas públicas são registradas em `emoframe-api/src/app.ts`.

Hoje a aplicação registra:

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

Cada instrumento segue o mesmo desenho:

```text
src/routes/<instrumento>.routes.ts
  -> middleware de autenticação
  -> middleware de validação Zod
  -> controller
  -> service
  -> banco via Drizzle/Postgres
```

Exemplo público:

```text
POST /api/sus/submissions
```

No deploy, esse endpoint é acessado como:

```text
POST https://emoframe.marcusrodrigues.dev/api/sus/submissions
```

## Como o container recebe uma atualização

O `Dockerfile` usa build em três etapas:

1. `deps`: instala dependências com `npm ci`.
2. `builder`: copia o código e roda `npm run build`, gerando `dist`.
3. `runner`: cria a imagem final com dependências de produção, `dist` e `package.json`.

Isso significa que mudanças em `src/` não entram automaticamente no container que já está rodando. Para uma nova funcionalidade chegar em produção, a imagem precisa ser reconstruída e o container precisa ser recriado.

No VPS, dentro de `/repositories/emoframe-api` ou do diretório equivalente do projeto:

```bash
docker compose up -d --build api
```

Para garantir que banco, migrations e API estejam consistentes, o fluxo recomendado é:

```bash
docker compose up -d --build
```

Esse comando reconstrói as imagens necessárias, sobe o Postgres, executa o serviço `migrate` e recria a API com o novo `dist`.

## Atualizações que alteram o banco

O projeto usa Drizzle. O schema fica em:

```text
emoframe-api/src/db/schema.ts
```

O `docker-compose.yml` possui um serviço `migrate` que roda:

```bash
npm run db:push
```

Esse comando aplica o schema Drizzle no Postgres usando as variáveis:

```text
POSTGRES_USER
POSTGRES_PASSWORD
POSTGRES_DB
POSTGRES_HOST=db
POSTGRES_PORT=5432
```

Quando a mudança envolver tabelas, colunas ou tipos:

```bash
docker compose up -d --build db
docker compose up --build migrate
docker compose up -d --build api
```

Ou, para o fluxo completo:

```bash
docker compose up -d --build
```

Depois confira se os containers estão saudáveis:

```bash
docker compose ps
```

E veja logs se algo falhar:

```bash
docker compose logs migrate
docker compose logs api
```

## Como o Nginx transmite a API para fora

A configuração documentada em `DEPLOY_API_NGINX_VPS.md` usa dois comportamentos importantes.

Primeiro, o health check público:

```nginx
location = /api/health {
    proxy_pass http://127.0.0.1:3000/health;
}
```

Isso transforma:

```text
https://emoframe.marcusrodrigues.dev/api/health
```

em:

```text
http://127.0.0.1:3000/health
```

Segundo, as rotas da API:

```nginx
location /api/ {
    proxy_pass http://127.0.0.1:3000;
}
```

O detalhe importante é que o `proxy_pass` está sem barra final. Assim, o Nginx preserva o caminho `/api/...` quando encaminha para o Express.

Exemplo:

```text
https://emoframe.marcusrodrigues.dev/api/sus/submissions
```

chega na aplicação como:

```text
http://127.0.0.1:3000/api/sus/submissions
```

Se o `proxy_pass` fosse `http://127.0.0.1:3000/` com barra final, o prefixo `/api/` poderia ser removido e a API receberia caminhos errados, como `/sus/submissions`.

## Quando precisa mexer no Nginx

Na maioria das atualizações de funcionalidade, não precisa alterar o Nginx. Basta manter as novas rotas dentro do prefixo `/api`.

Exemplos que não exigem mudança no Nginx:

```text
POST /api/sus/submissions
POST /api/novo-instrumento/submissions
GET /api/novo-recurso
```

Casos que podem exigir ajuste no Nginx:

- mudar o domínio público;
- mudar a porta local da API;
- remover o prefixo `/api` da aplicação;
- criar outro serviço público no mesmo domínio;
- publicar WebSocket, upload grande ou timeout maior;
- mudar o endpoint público de health check.

Sempre que alterar Nginx:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

## Checklist de deploy de funcionalidade

1. Atualizar o código no servidor.

```bash
git pull
```

2. Conferir variáveis de ambiente.

```bash
cat .env
```

As principais são:

```text
PORT=3000
POSTGRES_USER=...
POSTGRES_PASSWORD=...
POSTGRES_DB=...
TEST_BEARER_TOKEN=...
```

3. Reconstruir e subir os containers.

```bash
docker compose up -d --build
```

4. Conferir estado.

```bash
docker compose ps
```

5. Conferir logs.

```bash
docker compose logs api
docker compose logs migrate
```

6. Testar a API dentro do VPS.

```bash
curl -i http://127.0.0.1:3000/health
```

7. Testar pelo caminho público.

```bash
curl -i https://emoframe.marcusrodrigues.dev/api/health
```

8. Testar uma rota real com autenticação.

```bash
curl -i -X POST https://emoframe.marcusrodrigues.dev/api/sus/submissions \
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

## Estados esperados

Health interno:

```bash
curl -i http://127.0.0.1:3000/health
```

Deve retornar `HTTP 200` com:

```json
{"status":"OK","service":"api-micro"}
```

Health público:

```bash
curl -i https://emoframe.marcusrodrigues.dev/api/health
```

Deve retornar `HTTP 200` com o mesmo JSON.

Rota `GET` em submissions:

```bash
curl -i https://emoframe.marcusrodrigues.dev/api/sus/submissions
```

Pode retornar `404`, porque a rota foi definida apenas como `POST`.

Rota `POST` sem token:

```bash
curl -i -X POST https://emoframe.marcusrodrigues.dev/api/sus/submissions \
  -H "Content-Type: application/json" \
  -d '{}'
```

Deve retornar `401`, indicando que a requisição chegou na API e foi bloqueada pelo middleware de autenticação.

## Pontos de atenção

- O container final não usa `src/`; ele usa `dist`. Por isso sempre rode build/rebuild depois de alterar código TypeScript.
- A porta do Compose deve continuar presa em `127.0.0.1` para evitar exposição direta da API.
- O Nginx deve preservar `/api/...` nas rotas normais.
- O endpoint `/api/health` é um caso especial no Nginx, porque a aplicação internamente expõe `/health`.
- `TEST_BEARER_TOKEN` precisa estar definido no ambiente do container para as rotas protegidas funcionarem.
- `user-role: guest` recebe `403` pelo middleware de autenticação.
- Mudanças de banco devem ser validadas com logs do `migrate` antes de considerar o deploy finalizado.
