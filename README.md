# Outbox de notificação

Serviço NestJS que aceita um payload, grava a intenção num outbox e só persiste no master depois que o webhook e o e-mail concluem. API e worker saem da mesma imagem. O Postgres guarda outbox, master, dead letters e erros. O Redis segura a fila BullMQ.

Spec em `0.1.0` (2026-11-11). Essa versão fecha o desenho e não altera código de produção: o que está em `src/` ainda é o scaffold. Os comandos de compose, worker, Prisma e Jest abaixo são os da spec e passam a valer na implementação.

## Documentos

| arquivo | papel |
| --- | --- |
| [specs/SPEC.md](specs/SPEC.md) | fonte de verdade |
| [specs/architecture.html](specs/architecture.html) | diagramas Mermaid (arquitetura, recepção, worker, estados, erros, modelo, deploy) e changelog. Abrir no navegador |
| [specs/CHANGELOG.md](specs/CHANGELOG.md) | histórico. Toda mudança entra em `[Unreleased]` no mesmo commit e é espelhada no HTML |

## Fluxo

1. `POST /notifications` valida o payload, grava `outbox_events` com status `pending` e enfileira um job. A mesma `idempotencyKey` responde `200` com o status atual e não enfileira de novo. Corrida com `P2002` tem o mesmo efeito.
2. O worker (`src/worker.ts`) lê a fila `notifications` e executa, nesta ordem: webhook (`WEBHOOK_URL`, timeout de 5s), e-mail via Resend, gravação de `master_records` com o outbox em `committed`.
3. Falha no webhook, no Resend ou no master deixa o master intocado. O BullMQ tenta 5 vezes, com backoff exponencial a partir de 1s. Na última, o outbox vai para `failed` e nasce uma linha em `dead_letters`.
4. Toda resposta HTTP `>= 400` e toda falha do worker grava `error_logs`. O `errorId` da resposta é o `id` dessa linha.

Estados do outbox: `pending` → `processing` → `committed` ou `failed`.

## Endpoints

Base local: `http://localhost:3000`. Swagger em `GET /docs`.

| rota | função | respostas |
| --- | --- | --- |
| `POST /notifications` | grava o outbox e enfileira | 201, 200 (chave repetida), 400, 413, 500 |
| `GET /notifications/:idempotencyKey` | status do outbox | 200, 404 |
| `GET /health` | healthcheck do container | 200 `{ "status": "ok" }` |

Corpo máximo de 100kb. Campo desconhecido é `400` (`whitelist` e `forbidNonWhitelisted`).

```json
{
  "idempotencyKey": "appt-2026-11-12-p123-confirm",
  "to": "paciente@example.com",
  "subject": "Consulta confirmada",
  "body": "Sua consulta foi confirmada para 12/11 às 14h.",
  "metadata": {
    "patientId": "p-123",
    "appointmentId": "a-987",
    "source": "telepatia"
  }
}
```

`idempotencyKey`: obrigatória, até 128 caracteres, `^[A-Za-z0-9._:-]+$`. `to` é e-mail. `subject` até 200 caracteres. `body` até 10000. `metadata` é opcional, no máximo 20 chaves string.

Erro padrão, gerado pelo `AllExceptionsFilter`:

```json
{
  "statusCode": 400,
  "error": "Bad Request",
  "message": ["to must be an email"],
  "path": "/notifications",
  "timestamp": "2026-11-11T18:00:00.000Z",
  "errorId": "0b8f6c1e-3d1a-4c55-9a43-2e7f1f0c9d10"
}
```

Em `500`, `message` é `"Internal server error"`. Stack fica em `error_logs` e no stdout.

## Stack

NestJS 12 (controller, service, repository), Prisma, `class-validator`, `@nestjs/swagger`, BullMQ, Redis 7, SDK `resend`, Jest com `ts-jest`, Node 24 na imagem.

Postgres 16, database `notifications`, quatro tabelas: `outbox_events`, `master_records`, `dead_letters`, `error_logs`. A ligação entre elas é a `idempotencyKey`, sem foreign key.

## Variáveis

Copiar `.env.example` para `.env` na raiz. Não sobrescrever um `.env` que já exista.

| nome | uso |
| --- | --- |
| `DATABASE_URL` | Postgres |
| `REDIS_HOST`, `REDIS_PORT` | Redis da fila |
| `WEBHOOK_URL` | destino do passo 1 do worker |
| `RESEND_API_KEY`, `RESEND_FROM` | e-mail |
| `PORT` | porta da API, padrão `3000` |
| `OBSERVE_APP_KEY`, `OBSERVE_APP_SECRET` | opcionais, NestJS Observe |

## Como rodar

Na raiz do repositório, com Docker no ar.

Infra só para desenvolvimento e testes de integração:

```powershell
docker compose up -d postgres redis
pnpm install
pnpm start:dev
```

Worker, noutro terminal, depois do build:

```powershell
pnpm build
pnpm start:worker
```

Stack completa (API e worker na mesma imagem; só a API roda `prisma migrate deploy` e responde `/health`):

```powershell
docker compose up -d --build
```

Exemplo de recepção:

```powershell
curl.exe -i -X POST http://localhost:3000/notifications -H "Content-Type: application/json" -d "{\"idempotencyKey\":\"appt-2026-11-12-p123-confirm\",\"to\":\"paciente@example.com\",\"subject\":\"Consulta confirmada\",\"body\":\"Sua consulta foi confirmada.\"}"
```

Na hospedagem, a plataforma constrói o `Dockerfile` e sobe dois serviços da mesma imagem. A API usa o `CMD` padrão. O worker usa `node dist/worker.js`. Postgres e Redis gerenciados entram pelas variáveis acima.

## Testes

Jest no lugar do Vitest. Unitários mockam Prisma, HTTP, Resend e Redis. E2E sobe com Postgres e Redis do compose e mocka webhook e Resend na borda.

```powershell
pnpm test
pnpm test:e2e
pnpm test:cov
```

Pronto quando `pnpm test` e `pnpm test:e2e` estão verdes, o compose de Postgres e Redis está de pé, `docker build .` conclui e o container da API responde `GET /health`. A lista de casos está em `specs/SPEC.md`, seção Testes obrigatórios.
