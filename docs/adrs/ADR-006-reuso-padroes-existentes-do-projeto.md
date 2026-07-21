# ADR-006 — Reuso dos Padrões Existentes do Projeto

**Status:** Aceito  
**Data:** 2026-07-21  
**Participantes:** Larissa (Tech Lead), Bruno (Engenheiro Pleno), Diego (Engenheiro Sênior)

---

## Contexto

O projeto tem padrões estabelecidos de módulo, tratamento de erros, validação, logging e autenticação. A feature de webhooks pode seguir esses padrões ou introduzir abordagens novas. A decisão afeta consistência da codebase, curva de aprendizado e manutenção.

[09:27] Bruno: *"A gente tem um padrão claro na codebase. Cada domínio é um módulo em src/modules com controller, service, repository, routes e schemas. Webhook vai seguir igual."*

## Decisão

**Reuso máximo dos padrões existentes**. O módulo de webhooks segue exatamente a mesma estrutura dos módulos existentes:

**Estrutura de módulo:**
- Novo diretório `src/modules/webhooks/` com: `webhook.controller.ts`, `webhook.service.ts`, `webhook.repository.ts`, `webhook.routes.ts`, `webhook.schemas.ts`
- Worker: `src/worker.ts` como entry point, lógica de processamento em `src/modules/webhooks/webhook.processor.ts`

**Tratamento de erros:**
- Novos erros estendem a hierarquia de `AppError` (`src/shared/errors/app-error.ts`)
- Novos erros estendem as classes HTTP existentes (`src/shared/errors/http-errors.ts`)
- Prefixo `WEBHOOK_` para todos os códigos de erro do módulo (ex: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`)
- O `errorMiddleware` existente (`src/middlewares/error.middleware.ts`) já trata `AppError`, `ZodError` e erros Prisma — sem necessidade de alteração

**Validação:**
- Schemas Zod para todos os contratos de entrada
- Middleware `validate` (`src/middlewares/validate.middleware.ts`) aplicado em todas as rotas

**Logging:**
- Pino logger (`src/shared/logger/index.ts`) — reutilizado na API; worker cria nova instância com mesma configuração
- Sem introdução de nova biblioteca de logging

**Autenticação e autorização:**
- Middleware `authenticate` (`src/middlewares/auth.middleware.ts`) em todas as rotas protegidas
- `requireRole('ADMIN')` no endpoint de replay de DLQ, seguindo o mesmo padrão do módulo de usuários

**Injeção de dependência:**
- `buildControllers` em `src/app.ts` é extendido com `WebhookRepository`, `WebhookService`, `WebhookController`
- Router montado via `buildApiRouter` em `src/routes/index.ts`

## Alternativas Consideradas

### 1. Introduzir nova biblioteca de logging ou framework de erros

Usar algo como Winston, Bunyan ou estrutura de erro diferente para o módulo de webhooks.

**Por que foi descartado:** Divergência desnecessária da codebase estabelecida. Adiciona dependência nova sem benefício claro, aumenta a superfície de conhecimento que o time precisa manter. [09:29] Bruno: *"O logger, que é Pino, já tá no projeto inteiro. Não vamos botar nada novo."*

## Consequências

**Positivas:**
- Zero curva de aprendizado: o time conhece todos os padrões usados.
- `errorMiddleware` funciona sem alterações para erros do módulo de webhooks.
- Validação via Zod captura erros de URL, campos faltando e tipos inválidos na borda da requisição.
- Consistência facilita code review e manutenção futura.
- Código do módulo de webhooks é imediatamente legível para qualquer membro do time.

**Negativas:**
- Acopla o módulo de webhooks aos padrões atuais do projeto — mudanças arquiteturais futuras no projeto afetam o módulo de webhooks também.

## Referências

- Transcrição: [09:27]-[09:30] Bruno, Diego, Larissa; [09:29] Larissa (prefixo WEBHOOK_)
- Código: `src/shared/errors/app-error.ts` — classe base AppError
- Código: `src/shared/errors/http-errors.ts` — classes HTTP de erro
- Código: `src/middlewares/error.middleware.ts` — middleware centralizado de erros
- Código: `src/middlewares/validate.middleware.ts` — middleware Zod
- Código: `src/middlewares/auth.middleware.ts` — authenticate + requireRole
- Código: `src/shared/logger/index.ts` — instância Pino
- Código: `src/app.ts` — ponto de DI e montagem de routers
