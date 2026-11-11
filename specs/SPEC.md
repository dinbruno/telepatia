# Spec — outbox de notificação

Serviço NestJS que aceita um payload mockado, grava a intenção num outbox e só persiste no master depois que o webhook e o e-mail concluem.

Documentos desta pasta:

- `specs/SPEC.md` — esta spec, fonte de verdade.
- `specs/CHANGELOG.md` — histórico obrigatório de toda mudança (ver [Regra de changelog](#regra-de-changelog)).
- `specs/architecture.html` — diagramas Mermaid navegáveis e changelog renderizado. Abrir direto no navegador.

## Decisões fechadas

- O mecanismo é outbox. A recepção grava a intenção e responde sucesso.
- Um Postgres, database `notifications`, quatro tabelas: `outbox_events`, `master_records`, `dead_letters` e `error_logs`. A escrita da recepção e a do master não compartilham a mesma transação. O master é o último passo.
- Todo erro tratado, da API ou do worker, é gravado em `error_logs` no mesmo Postgres. Não há outro destino de erro além do log estruturado em stdout.
- Redis é infraestrutura obrigatória. A fila usa BullMQ em cima dele.
- A mesma chave de idempotência não cria outro registro e não enfileira outro envio. O retry é do job, não de um novo POST.
- Ordem do worker: webhook, Resend, master. Falha no webhook ou no Resend não escreve no master.
- API e worker saem da mesma imagem Docker, com comandos diferentes. O `Dockerfile` é obrigatório.

## Stack

- NestJS 12, camadas controller / service / repository
- Prisma, `class-validator`, `@nestjs/swagger`
- `@nestjs/bullmq` e Redis
- SDK `resend`
- Jest com `ts-jest` no lugar do Vitest
- Node 24 na imagem Docker

Variáveis de ambiente, só nomes: `DATABASE_URL`, `REDIS_HOST`, `REDIS_PORT`, `WEBHOOK_URL`, `RESEND_API_KEY`, `RESEND_FROM`, `PORT` (padrão `3000`), `OBSERVE_APP_KEY` e `OBSERVE_APP_SECRET` (opcionais). Entregar um `.env.example` com esses nomes e valores locais fictícios. Não sobrescrever um `.env` existente.

## Payload de entrada

O cliente envia JSON em `POST /notifications` com `Content-Type: application/json`. Corpo máximo de `100kb` (limite do body parser); acima disso, `413`.

Exemplo válido:

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

DTO `CreateNotificationDto`, validado com `class-validator`, `whitelist: true` e `forbidNonWhitelisted: true` (campo desconhecido é `400`):

| campo | tipo | regra |
| --- | --- | --- |
| `idempotencyKey` | string | obrigatório, não vazio, até 128 caracteres, `^[A-Za-z0-9._:-]+$` |
| `to` | string | obrigatório, e-mail válido |
| `subject` | string | obrigatório, não vazio, até 200 caracteres |
| `body` | string | obrigatório, não vazio, até 10000 caracteres |
| `metadata` | objeto | opcional, chaves e valores string, até 20 chaves |

O objeto validado é o que vai para `outbox_events.payload`.

## HTTP

Base local: `http://localhost:3000`. Swagger em `GET /docs`, com o DTO, as respostas e o formato de erro documentados.

### `POST /notifications` — endpoint de consumo do payload

É o único ponto de entrada da informação. Grava o outbox e enfileira o job.

| caso | status | corpo |
| --- | --- | --- |
| primeira recepção | `201` | `{ "idempotencyKey": "...", "status": "pending" }` |
| mesma chave de novo | `200` | `{ "idempotencyKey": "...", "status": "<status atual>" }`, sem novo job |
| payload inválido | `400` | formato de erro padrão |
| corpo acima de 100kb | `413` | formato de erro padrão |
| falha inesperada (Postgres ou Redis fora) | `500` | formato de erro padrão |

Corrida de duas requisições com a mesma chave: a violação de unicidade do `idempotencyKey` (Prisma `P2002`) é tratada como "mesma chave de novo" e responde `200`, sem novo job.

Exemplo:

```bash
curl -i -X POST http://localhost:3000/notifications \
  -H 'Content-Type: application/json' \
  -d '{"idempotencyKey":"appt-2026-11-12-p123-confirm","to":"paciente@example.com","subject":"Consulta confirmada","body":"Sua consulta foi confirmada."}'
```

### `GET /notifications/:idempotencyKey`

Consulta o andamento de uma notificação.

- `200` com `{ idempotencyKey, status, step, attemptCount, lastError, createdAt, updatedAt }`.
- `404` no formato de erro padrão quando a chave não existe.

### `GET /health`

Responde `200` com `{ "status": "ok" }`. Usado pelo `HEALTHCHECK` do Dockerfile e pela plataforma de hospedagem. Substitui o `GET /` de exemplo do scaffold (`AppController`/`AppService` viram o health).

### Formato de erro padrão

Toda resposta de erro sai do `AllExceptionsFilter` com este corpo:

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

`errorId` é o `id` da linha em `error_logs`, para rastrear o erro na base. Em `500`, `message` é `"Internal server error"`; stack e detalhes ficam só na base e no log.

## Tratamento e persistência de erros

Um único `AllExceptionsFilter` global (`APP_FILTER`) e um `ErrorLogRepository` gravam todos os erros. O worker usa o mesmo repositório.

| origem | quando grava | `statusCode` | `step` |
| --- | --- | --- | --- |
| `http` | toda resposta `>= 400` gerada pelo filtro | status HTTP | vazio |
| `worker` | toda tentativa que falha, inclusive as que ainda vão ser relançadas | vazio | `webhook`, `resend` ou `master` |

Regras:

- Não gravar `to` nem `body` no `context` (dado pessoal). O `context` leva `method`, `path`, `jobId`, `attempt` e, em `400`, as mensagens de validação.
- Se gravar o `error_logs` falhar, o filtro loga o erro original e a falha de gravação em stdout e responde mesmo assim, com `errorId` nulo. A falha do log nunca troca o status da resposta nem o resultado do job.
- `dead_letters` continua registrando só o esgotamento; `error_logs` registra cada falha.

## Infra

`docker-compose.yml` na raiz de `testing/`:

- Serviço `postgres`: imagem `postgres:16-alpine`, porta `5432`, database `notifications`, healthcheck com `pg_isready`.
- Serviço `redis`: imagem `redis:7-alpine`, porta `6379`, healthcheck com `redis-cli ping`.
- Serviço `api`: `build: .`, porta `3000`, `depends_on` com `condition: service_healthy` em `postgres` e `redis`, `env_file: .env`, `DATABASE_URL` e `REDIS_HOST` apontando para os nomes de serviço do compose.
- Serviço `worker`: mesma imagem, `command: ["node", "dist/worker.js"]`, mesmos `depends_on` e variáveis, sem porta.

Desenvolvimento local e testes de integração: `docker compose up -d postgres redis`. Stack completa: `docker compose up -d --build`.

### Dockerfile

`Dockerfile` na raiz de `testing/`, multi-stage, obrigatório para hospedagem:

```dockerfile
FROM node:24-alpine AS base
RUN corepack enable
WORKDIR /app

FROM base AS build
COPY package.json pnpm-lock.yaml ./
RUN pnpm install --frozen-lockfile
COPY . .
RUN pnpm prisma generate && pnpm build && pnpm prune --prod

FROM base AS runtime
ENV NODE_ENV=production
COPY --from=build --chown=node:node /app/node_modules ./node_modules
COPY --from=build --chown=node:node /app/dist ./dist
COPY --from=build --chown=node:node /app/prisma ./prisma
COPY --from=build --chown=node:node /app/package.json ./package.json
USER node
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=3s --start-period=20s CMD wget -qO- http://127.0.0.1:${PORT:-3000}/health || exit 1
CMD ["sh", "-c", "./node_modules/.bin/prisma migrate deploy && node dist/main.js"]
```

O runtime chama o binário do Prisma direto, sem `pnpm`, para o corepack não baixar nada no boot.

Regras:

- `prisma` fica em `dependencies`, não em `devDependencies`, porque o `migrate deploy` roda no runtime.
- A migração roda só no comando da API. O worker sobrescreve o `CMD` e não migra.
- O `HEALTHCHECK` só vale para a API; no serviço `worker` do compose, `healthcheck: disable: true`.
- A imagem roda como usuário `node`, nunca root.
- `.dockerignore` com `node_modules`, `dist`, `coverage`, `.env`, `.git` e `specs`.

### Hospedagem

A plataforma constrói a partir do `Dockerfile` e cria dois serviços da mesma imagem: API com o `CMD` padrão e healthcheck em `/health`; worker com `node dist/worker.js`. Postgres e Redis gerenciados entram por `DATABASE_URL`, `REDIS_HOST` e `REDIS_PORT`. A API escuta em `process.env.PORT`.

## Modelo

`outbox_events`

| campo | regra |
| --- | --- |
| `id` | uuid |
| `idempotencyKey` | único |
| `payload` | json do DTO |
| `status` | `pending`, `processing`, `committed`, `failed` |
| `step` | `webhook`, `resend`, `master` ou vazio |
| `attemptCount` | inteiro, começa em 0 |
| `lastError` | texto ou vazio |
| `createdAt`, `updatedAt` | timestamps |

`master_records`

| campo | regra |
| --- | --- |
| `id` | uuid |
| `idempotencyKey` | único, igual ao do outbox |
| `payload` | json persistido só após webhook e Resend |
| `createdAt` | timestamp |

`dead_letters`

| campo | regra |
| --- | --- |
| `id` | uuid |
| `name` | nome do job registrado na `WorkerFactory` |
| `idempotencyKey` | a chave do evento |
| `step` | passo em que esgotou |
| `error` | mensagem da falha |
| `payload` | json do evento |
| `createdAt` | timestamp |

`error_logs`

| campo | regra |
| --- | --- |
| `id` | uuid, devolvido como `errorId` |
| `source` | `http` ou `worker` |
| `idempotencyKey` | texto ou vazio |
| `step` | `webhook`, `resend`, `master` ou vazio |
| `statusCode` | inteiro ou vazio |
| `message` | texto |
| `stack` | texto ou vazio |
| `context` | json ou vazio, sem dados pessoais |
| `createdAt` | timestamp, com índice |

## Fluxo do worker

Entrada própria em `src/worker.ts` (compila para `dist/worker.js`): sobe um application context do Nest sem HTTP, só com a `WorkerFactory`.

Fila `notifications`. Job carrega só o `idempotencyKey`, com `jobId` igual à chave para o BullMQ também barrar duplicata.

1. Carrega o outbox. Status `committed` encerra sem efeito.
2. Marca `processing` e incrementa `attemptCount`.
3. `POST` em `WEBHOOK_URL`, timeout de 5s, header `Idempotency-Key` igual à chave, corpo abaixo. Sucesso é HTTP 2xx. Outro resultado ou timeout falha o passo `webhook`.
4. Envia o e-mail pelo SDK do Resend (`from: RESEND_FROM`, `to`, `subject`, `text: body`), com header `Idempotency-Key` igual à chave do registro. Falha de API falha o passo `resend`.
5. Grava `master_records` e marca o outbox `committed` no passo `master`, na mesma transação.

Corpo do webhook (passo 3):

```json
{
  "idempotencyKey": "appt-2026-11-12-p123-confirm",
  "to": "paciente@example.com",
  "subject": "Consulta confirmada",
  "metadata": { "patientId": "p-123", "appointmentId": "a-987", "source": "telepatia" }
}
```

Falha nos passos 3, 4 ou 5: master intocado, `lastError` e `step` preenchidos, uma linha em `error_logs` com `source: worker`, job relançado pelo BullMQ. `attempts: 5`, backoff exponencial a partir de 1s. Na última tentativa, status do outbox vai para `failed` e uma linha entra em `dead_letters`. Log estruturado com `idempotencyKey`, `step` e `error`.

## Factory do worker

`WorkerFactory` é um provider. Cada job futuro se registra com `name` e `processor`. O fluxo de notificação é o primeiro registro. A factory cria o worker BullMQ na fila correspondente e aplica a mesma política de tentativas, `error_logs` e dead-letter: a cada falha grava em `error_logs`; ao esgotar, grava origem (`name`, `step`, `idempotencyKey`, `error`) em `dead_letters` e loga.

## Pré-configuração do Jest

Remover o Vitest do caminho de teste. Apagar `vitest.config.ts` e `vitest.config.e2e.ts` quando o Jest estiver verde. Trocar em `tsconfig.json` o type `vitest/globals` por `jest`.

Dependências de desenvolvimento: `jest`, `ts-jest`, `@types/jest`.

`package.json`, scripts:

```json
{
  "test": "node --experimental-vm-modules node_modules/jest/bin/jest.js",
  "test:watch": "node --experimental-vm-modules node_modules/jest/bin/jest.js --watch",
  "test:cov": "node --experimental-vm-modules node_modules/jest/bin/jest.js --coverage",
  "test:e2e": "node --experimental-vm-modules node_modules/jest/bin/jest.js --config ./test/jest-e2e.json",
  "start:prod": "node dist/main.js",
  "start:worker": "node dist/worker.js"
}
```

`jest.config.ts` na raiz de `testing/`:

```ts
import type { Config } from 'jest';

const config: Config = {
  moduleFileExtensions: ['js', 'json', 'ts'],
  rootDir: '.',
  testRegex: '.*\\.spec\\.ts$',
  transform: {
    '^.+\\.(t|j)s$': [
      'ts-jest',
      {
        useESM: true,
        tsconfig: 'tsconfig.json',
      },
    ],
  },
  extensionsToTreatAsEsm: ['.ts'],
  moduleNameMapper: {
    '^(\\.{1,2}/.*)\\.js$': '$1',
  },
  testEnvironment: 'node',
  setupFiles: ['reflect-metadata'],
  clearMocks: true,
};

export default config;
```

`test/jest-e2e.json`:

```json
{
  "moduleFileExtensions": ["js", "json", "ts"],
  "rootDir": ".",
  "testEnvironment": "node",
  "testRegex": ".e2e-spec.ts$",
  "transform": {
    "^.+\\.(t|j)s$": [
      "ts-jest",
      {
        "useESM": true,
        "tsconfig": "tsconfig.json"
      }
    ]
  },
  "extensionsToTreatAsEsm": [".ts"],
  "moduleNameMapper": {
    "^(\\.{1,2}/.*)\\.js$": "$1"
  }
}
```

`package.json` precisa de `"type": "module"`, que o projeto já tem. Imports relativos nos testes terminam em `.js`, no mesmo padrão do `src`.

## Testes obrigatórios

Unitários, com Prisma, HTTP e Resend mockados. Redis mockado no unitário.

- Primeira recepção grava outbox `pending` e enfileira um job.
- A mesma chave não grava outro outbox e não enfileira outro job.
- Violação `P2002` na corrida responde como chave repetida, sem novo job.
- Webhook fora de 2xx não chama o Resend e não escreve no master. O passo registrado é `webhook`, com uma linha em `error_logs`.
- Resend com erro não escreve no master. O passo registrado é `resend`, com uma linha em `error_logs`.
- Webhook 2xx e Resend ok gravam o master e marcam o outbox `committed`.
- Na última tentativa, nasce uma `dead_letter` com `name`, `idempotencyKey`, `step` e `error`.
- DTO sem `idempotencyKey`, com email inválido ou com campo desconhecido falha a validação.
- `AllExceptionsFilter` grava `error_logs` e devolve o `errorId`; se a gravação falhar, a resposta sai com o status original e `errorId` nulo.

E2E, com Postgres e Redis do compose no ar, webhook e Resend mockados na borda:

- `POST /notifications` com payload válido responde `201` e o outbox fica `pending`.
- Repetir o POST com a mesma chave responde `200` e existe uma única linha de outbox.
- `POST` inválido responde `400`, com `errorId` que existe em `error_logs`.
- `GET /notifications/:idempotencyKey` responde `200` para chave existente e `404` para chave desconhecida.
- `GET /health` responde `200`.

O critério de pronto é `pnpm test` e `pnpm test:e2e` verdes, com o compose de Postgres e Redis de pé, `docker build .` concluindo e o container da API respondendo `/health`.

## Regra de changelog

Toda mudança no projeto — código, spec, infra, dependência ou teste — entra em `specs/CHANGELOG.md` no mesmo commit, na seção `[Unreleased]`, no formato [Keep a Changelog](https://keepachangelog.com/pt-BR/1.1.0/) (`Added`, `Changed`, `Fixed`, `Removed`, `Security`). Cada entrada cita a seção da spec ou o arquivo que motivou a mudança.

A seção Changelog de `specs/architecture.html` espelha o `CHANGELOG.md` e é atualizada no mesmo commit. Mudança que altera fluxo, tabela, endpoint ou infra também atualiza o diagrama correspondente no HTML.

Commit sem entrada no changelog não entra.
