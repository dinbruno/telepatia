# Changelog

Todas as mudanças do projeto ficam aqui, no formato [Keep a Changelog](https://keepachangelog.com/pt-BR/1.1.0/) e com [SemVer](https://semver.org/lang/pt-BR/).
Regra: toda mudança entra em `[Unreleased]` no mesmo commit e é espelhada em `specs/architecture.html` (ver `SPEC.md` › Regra de changelog).

## [Unreleased]

## [0.1.0] - 2026-11-11

Refinamento da spec. Nenhum código de produção alterado.

### Added

- `SPEC.md` › Payload de entrada: exemplo JSON, tabela do DTO com limites de tamanho, `whitelist`/`forbidNonWhitelisted` e limite de 100kb no corpo.
- `SPEC.md` › HTTP: `POST /notifications` como endpoint de consumo, com tabela de status (`201`, `200`, `400`, `413`, `500`), exemplo `curl` e tratamento da corrida `P2002`.
- `SPEC.md` › HTTP: `GET /notifications/:idempotencyKey` para consulta de status e `GET /health` para healthcheck.
- `SPEC.md` › Formato de erro padrão com `errorId`.
- `SPEC.md` › Tratamento e persistência de erros: `AllExceptionsFilter` global e `ErrorLogRepository`; erros da API e do worker gravados no mesmo Postgres.
- `SPEC.md` › Modelo: tabela `error_logs`; campo `name` em `dead_letters`.
- `SPEC.md` › Dockerfile: multi-stage em Node 24, usuário `node`, `HEALTHCHECK`, `prisma migrate deploy` no boot da API; `.dockerignore`.
- `SPEC.md` › Infra: serviços `api` e `worker` no `docker-compose.yml`, mesma imagem.
- `SPEC.md` › Hospedagem: dois serviços da mesma imagem, Postgres e Redis gerenciados por variável.
- `SPEC.md` › Fluxo do worker: entrada `src/worker.ts`, `jobId` igual à chave, timeout de 5s e corpo do webhook.
- `SPEC.md` › Testes obrigatórios: casos de `error_logs`, `P2002`, campo desconhecido, `GET` de status e `/health`.
- `SPEC.md` › Regra de changelog.
- `specs/architecture.html`: diagramas Mermaid navegáveis (arquitetura, sequências, estados, ER, deploy) e changelog.
- `specs/CHANGELOG.md`: este arquivo.

### Changed

- `SPEC.md` movido da raiz de `testing/` para `specs/SPEC.md`.
- `SPEC.md` › Decisões fechadas: quatro tabelas em vez de duas; Dockerfile obrigatório.
- `SPEC.md` › Stack: variáveis `PORT`, `OBSERVE_APP_KEY` e `OBSERVE_APP_SECRET`.
- `SPEC.md` › Pré-configuração do Jest: scripts `start:prod` com `dist/main.js` e `start:worker`.
- `AGENTS.md`: regra de changelog obrigatório.
