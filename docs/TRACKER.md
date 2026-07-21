# TRACKER — Rastreabilidade de Requisitos e Decisões

Este documento mapeia cada item registrado nos documentos de design à sua origem na transcrição da reunião (TRANSCRICAO.md) ou no código-fonte existente. Todo item sem origem identificável foi removido dos documentos.

**Legenda de Fonte:**
- `TRANSCRICAO` — origem na reunião técnica (TRANSCRICAO.md)
- `CODIGO` — origem no código-fonte do repositório

---

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|----|-----------|------|-------------------|-------|-------------|
| ADR-001 | docs/adrs/ADR-001-outbox-pattern-no-mysql.md | Decisão Arquitetural | Padrão Outbox no MySQL para criação atômica de eventos de webhook | TRANSCRICAO | [09:06] Diego |
| ADR-001-ALT-1 | docs/adrs/ADR-001-outbox-pattern-no-mysql.md | Alternativa Descartada | HTTP síncrono dentro da transação de mudança de status | TRANSCRICAO | [09:04] Bruno |
| ADR-001-ALT-2 | docs/adrs/ADR-001-outbox-pattern-no-mysql.md | Alternativa Descartada | Redis Streams ou message broker externo | TRANSCRICAO | [09:07] Diego |
| ADR-002 | docs/adrs/ADR-002-retry-com-backoff-e-dlq.md | Decisão Arquitetural | 5 retries com backoff 1m/5m/30m/2h/12h e DLQ em tabela separada | TRANSCRICAO | [09:17] Diego |
| ADR-002-ALT-1 | docs/adrs/ADR-002-retry-com-backoff-e-dlq.md | Alternativa Descartada | 3 tentativas (mais agressivo) | TRANSCRICAO | [09:16] Bruno |
| ADR-002-ALT-2 | docs/adrs/ADR-002-retry-com-backoff-e-dlq.md | Alternativa Descartada | Retry indefinido com backoff | TRANSCRICAO | [09:15] Diego |
| ADR-002-ALT-3 | docs/adrs/ADR-002-retry-com-backoff-e-dlq.md | Alternativa Descartada | Marcar como "failed" na própria outbox em vez de tabela DLQ separada | TRANSCRICAO | [09:18] Diego |
| ADR-003 | docs/adrs/ADR-003-hmac-sha256-com-secret-por-endpoint.md | Decisão Arquitetural | HMAC-SHA256 com secret única por endpoint e rotação com grace period de 24h | TRANSCRICAO | [09:20]-[09:22] Sofia |
| ADR-003-ALT-1 | docs/adrs/ADR-003-hmac-sha256-com-secret-por-endpoint.md | Alternativa Descartada | Secret global única para toda a plataforma | TRANSCRICAO | [09:21] Sofia |
| ADR-004 | docs/adrs/ADR-004-garantia-at-least-once-com-event-id.md | Decisão Arquitetural | Garantia at-least-once com X-Event-Id para deduplicação pelo cliente | TRANSCRICAO | [09:24]-[09:25] Diego |
| ADR-004-ALT-1 | docs/adrs/ADR-004-garantia-at-least-once-com-event-id.md | Alternativa Descartada | Garantia exactly-once com coordenação bilateral | TRANSCRICAO | [09:25] Diego |
| ADR-005 | docs/adrs/ADR-005-worker-processo-separado-com-polling.md | Decisão Arquitetural | Worker como processo Node.js separado com polling de 2 segundos | TRANSCRICAO | [09:09]-[09:11] Diego |
| ADR-005-ALT-1 | docs/adrs/ADR-005-worker-processo-separado-com-polling.md | Alternativa Descartada | Worker in-process (mesma instância da API) | TRANSCRICAO | [09:11] Diego |
| ADR-005-ALT-2 | docs/adrs/ADR-005-worker-processo-separado-com-polling.md | Alternativa Descartada | Trigger do banco de dados para notificar o worker | TRANSCRICAO | [09:09] Diego |
| ADR-006 | docs/adrs/ADR-006-reuso-padroes-existentes-do-projeto.md | Decisão Arquitetural | Reuso de AppError, Pino, validate middleware, auth middleware e estrutura de módulos | TRANSCRICAO | [09:27]-[09:30] Bruno |
| ADR-006-CODE-1 | docs/adrs/ADR-006-reuso-padroes-existentes-do-projeto.md | Restrição | Classe base AppError para hierarquia de erros do módulo de webhooks | CODIGO | src/shared/errors/app-error.ts |
| ADR-006-CODE-2 | docs/adrs/ADR-006-reuso-padroes-existentes-do-projeto.md | Restrição | Classes HTTP de erro existentes (NotFoundError, BadRequestError, etc.) | CODIGO | src/shared/errors/http-errors.ts |
| ADR-006-CODE-3 | docs/adrs/ADR-006-reuso-padroes-existentes-do-projeto.md | Restrição | Error middleware centralizado que trata AppError, ZodError e erros Prisma | CODIGO | src/middlewares/error.middleware.ts |
| ADR-006-CODE-4 | docs/adrs/ADR-006-reuso-padroes-existentes-do-projeto.md | Restrição | Middleware validate com Zod para validação de entrada | CODIGO | src/middlewares/validate.middleware.ts |
| ADR-006-CODE-5 | docs/adrs/ADR-006-reuso-padroes-existentes-do-projeto.md | Restrição | Middleware authenticate e requireRole para autenticação e autorização | CODIGO | src/middlewares/auth.middleware.ts |
| RFC-CONTEXTO-01 | docs/RFC.md | Contexto | Três clientes B2B (Atlas, MaxDistribuicao, Nova Cargo) pedindo notificação em tempo real | TRANSCRICAO | [09:00] Marcos |
| RFC-CONTEXTO-02 | docs/RFC.md | Contexto | Atlas ameaçando migrar para concorrente se feature não for entregue no trimestre | TRANSCRICAO | [09:00] Marcos |
| RFC-CONTEXTO-03 | docs/RFC.md | Restrição | Latência aceitável: abaixo de 10 segundos | TRANSCRICAO | [09:02] Marcos |
| RFC-CONTEXTO-04 | docs/RFC.md | Restrição | Webhook é exclusivamente outbound (do OMS para os clientes) | TRANSCRICAO | [09:02]-[09:03] Sofia, Marcos |
| RFC-ALT-01 | docs/RFC.md | Alternativa Descartada | Despacho HTTP síncrono dentro da transação de status | TRANSCRICAO | [09:04] Bruno, Larissa |
| RFC-ALT-02 | docs/RFC.md | Alternativa Descartada | Redis Streams / message broker externo | TRANSCRICAO | [09:07] Diego, Larissa |
| RFC-OPEN-01 | docs/RFC.md | Questão em Aberto | Rate limiting de saída por cliente: observar e decidir depois | TRANSCRICAO | [09:38]-[09:39] Diego, Larissa |
| RFC-OPEN-02 | docs/RFC.md | Questão em Aberto | E-mail de notificação em caso de falhas recorrentes: adiado para próxima fase | TRANSCRICAO | [09:37] Marcos, Larissa |
| RFC-OPEN-03 | docs/RFC.md | Questão em Aberto | Arquivamento de entradas entregues na outbox (mencionado 30 dias, sem mecanismo definido) | TRANSCRICAO | [09:08] Diego |
| PRD-CTX-01 | docs/PRD.md | Contexto | Clientes B2B fazem polling em GET /api/v1/orders — integração lenta e cara | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-02 | docs/PRD.md | Contexto | Requisito de negócio: entrega abaixo de 10 segundos é "tempo real" para os clientes | TRANSCRICAO | [09:02] Marcos |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastro de endpoints de webhook com URL e lista de status de evento | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Secret gerada automaticamente pelo sistema na criação do webhook | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Atualização de URL, lista de eventos e estado ativo/inativo | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Remoção de endpoint de webhook | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Listagem de endpoints de webhook por cliente | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Inserção de evento na outbox para cada endpoint ativo que tenha o status na sua lista | TRANSCRICAO | [09:33]-[09:34] Marcos, Bruno |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Inserção na outbox dentro da mesma transação SQL da mudança de status | TRANSCRICAO | [09:06] Diego, [09:41] Bruno |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Worker despacha HTTP POST com timeout de 10 segundos | TRANSCRICAO | [09:09] Diego, [09:42] Diego |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Retry com backoff: 1min, 5min, 30min, 2h, 12h (máx 5 tentativas) | TRANSCRICAO | [09:17] Diego |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Após 5 falhas, evento vai para Dead Letter Queue | TRANSCRICAO | [09:18] Diego |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Admin replica entrada de DLQ via endpoint POST /admin/webhooks/dead-letter/:id/replay | TRANSCRICAO | [09:18] Diego, [09:35]-[09:36] Sofia |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | Histórico de entregas consultável via GET /webhooks/:id/deliveries | TRANSCRICAO | [09:34]-[09:35] Marcos |
| PRD-FR-13 | docs/PRD.md | Requisito Funcional | Rotação de secret com grace period de 24 horas | TRANSCRICAO | [09:21] Sofia |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Latência máxima de 10 segundos da mudança de status até a entrega | TRANSCRICAO | [09:02] Marcos |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | URLs de webhook obrigatoriamente HTTPS; http:// rejeitado | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Assinatura HMAC-SHA256 em cada entrega no header X-Signature | TRANSCRICAO | [09:20] Sofia |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | Timeout de 10 segundos por tentativa de entrega | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | Payload máximo de 64KB por evento | TRANSCRICAO | [09:24] Diego, Larissa |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | Garantia at-least-once; clientes deduplicam pelo X-Event-Id | TRANSCRICAO | [09:24]-[09:25] Diego |
| PRD-NFR-07 | docs/PRD.md | Requisito Não Funcional | Payload é snapshot do estado no momento da mudança, não no envio | TRANSCRICAO | [09:52] Larissa, Diego |
| PRD-NFR-08 | docs/PRD.md | Requisito Não Funcional | Worker roda como processo separado com ciclo de vida independente da API | TRANSCRICAO | [09:11] Diego |
| PRD-ESCOPO-OUT-01 | docs/PRD.md | Fora de Escopo | E-mail de notificação de falhas adiado para próxima fase | TRANSCRICAO | [09:37] Larissa |
| PRD-ESCOPO-OUT-02 | docs/PRD.md | Fora de Escopo | Rate limiting de saída: observar e decidir depois | TRANSCRICAO | [09:39] Diego, Larissa |
| PRD-ESCOPO-OUT-03 | docs/PRD.md | Fora de Escopo | Dashboard visual: projeto separado do time de frontend | TRANSCRICAO | [09:40] Larissa |
| PRD-ESCOPO-OUT-04 | docs/PRD.md | Fora de Escopo | Garantia de ordenação com múltiplos workers: limitação conhecida, futuro | TRANSCRICAO | [09:13] Diego, Larissa |
| PRD-ESCOPO-OUT-05 | docs/PRD.md | Fora de Escopo | Arquivamento automático de eventos entregues da outbox | TRANSCRICAO | [09:08] Diego |
| FDD-FLOW-01 | docs/FDD.md | Fluxo | Evento inserido na outbox dentro da transação $transaction do changeStatus | TRANSCRICAO | [09:06] Diego, [09:41] Bruno |
| FDD-FLOW-02 | docs/FDD.md | Fluxo | Filtragem por status na inserção da outbox, não no envio | TRANSCRICAO | [09:34] Bruno |
| FDD-FLOW-03 | docs/FDD.md | Fluxo | Payload salvo como snapshot no momento da inserção, não renderizado no envio | TRANSCRICAO | [09:52] Larissa, Diego |
| FDD-FLOW-04 | docs/FDD.md | Fluxo | Worker polling a cada 2s, processa batch de 10 eventos, marca PROCESSING antes de enviar | TRANSCRICAO | [09:09]-[09:10] Diego, Larissa |
| FDD-FLOW-05 | docs/FDD.md | Fluxo | Backoff schedule: 1m/5m/30m/2h/12h após cada falha | TRANSCRICAO | [09:17] Diego |
| FDD-FLOW-06 | docs/FDD.md | Fluxo | DLQ replay: cria novo registro na outbox com attempts=0, registra replayedBy | TRANSCRICAO | [09:18] Diego, [09:36] Sofia |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | POST /api/v1/webhooks — cadastro de webhook | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | PATCH /api/v1/webhooks/:id — atualização parcial | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | DELETE /api/v1/webhooks/:id — remoção | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | GET /api/v1/webhooks?customerId=uuid — listagem | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato | POST /api/v1/webhooks/:id/rotate-secret — rotação de secret | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato | GET /api/v1/webhooks/:id/deliveries — histórico de entregas | TRANSCRICAO | [09:34]-[09:35] Marcos |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato | POST /api/v1/admin/webhooks/dead-letter/:id/replay — replay admin de DLQ | TRANSCRICAO | [09:18] Diego, [09:35]-[09:36] Sofia |
| FDD-PAYLOAD-01 | docs/FDD.md | Contrato | Headers X-Event-Id, X-Signature, X-Timestamp no envio ao cliente | TRANSCRICAO | [09:44] Diego |
| FDD-PAYLOAD-02 | docs/FDD.md | Contrato | Header X-Webhook-Id adicionado por sugestão de Sofia | TRANSCRICAO | [09:44] Sofia |
| FDD-PAYLOAD-03 | docs/FDD.md | Contrato | Payload JSON com event_id, event_type, timestamp, order_id, order_number, customer_id, from_status, to_status, total_cents | TRANSCRICAO | [09:43] Diego |
| FDD-PAYLOAD-04 | docs/FDD.md | Restrição | Items do pedido NÃO incluídos no payload; cliente busca detalhes em GET /orders/:id | TRANSCRICAO | [09:43] Diego |
| FDD-ERRO-01 | docs/FDD.md | Restrição | Prefixo WEBHOOK_ para todos os códigos de erro do módulo | TRANSCRICAO | [09:29] Larissa |
| FDD-ERRO-02 | docs/FDD.md | Restrição | WEBHOOK_NOT_FOUND para endpoint inexistente | TRANSCRICAO | [09:28]-[09:29] Bruno |
| FDD-ERRO-03 | docs/FDD.md | Restrição | WEBHOOK_INVALID_URL para URL sem HTTPS | TRANSCRICAO | [09:23] Sofia |
| FDD-RESILIENCE-01 | docs/FDD.md | Restrição | Timeout HTTP de 10 segundos por tentativa de entrega | TRANSCRICAO | [09:42] Diego |
| FDD-RESILIENCE-02 | docs/FDD.md | Restrição | Payload máximo de 64KB | TRANSCRICAO | [09:24] Diego, Larissa |
| FDD-ADMIN-01 | docs/FDD.md | Restrição | Role ADMIN obrigatória no endpoint de replay de DLQ | TRANSCRICAO | [09:35]-[09:36] Sofia, Larissa |
| FDD-ADMIN-02 | docs/FDD.md | Restrição | Replay registra userId do admin (auditoria) | TRANSCRICAO | [09:36] Sofia |
| FDD-INT-01 | docs/FDD.md | Integração | publishWebhookEvent() chamado dentro do $transaction do changeStatus | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-02 | docs/FDD.md | Integração | Novos erros extendem AppError com prefixo WEBHOOK_ | CODIGO | src/shared/errors/app-error.ts |
| FDD-INT-03 | docs/FDD.md | Integração | Novos erros estendem classes HTTP existentes (NotFoundError, BadRequestError) | CODIGO | src/shared/errors/http-errors.ts |
| FDD-INT-04 | docs/FDD.md | Integração | errorMiddleware existente trata AppError sem alteração | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INT-05 | docs/FDD.md | Integração | authenticate e requireRole('ADMIN') usados nas rotas de webhook | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INT-06 | docs/FDD.md | Integração | Schemas Zod + middleware validate para validação de entrada | CODIGO | src/middlewares/validate.middleware.ts |
| FDD-INT-07 | docs/FDD.md | Integração | buildControllers em app.ts extendido com WebhookRepository/Service/Controller | CODIGO | src/app.ts |
| FDD-INT-08 | docs/FDD.md | Integração | 4 novos modelos Prisma seguem convenções de schema existente (UUID, @@map, @@index) | CODIGO | prisma/schema.prisma |
| FDD-INT-09 | docs/FDD.md | Integração | Worker usa Pino com mesma config do projeto; instância própria por ser processo separado | CODIGO | src/shared/logger/index.ts |
| FDD-INT-10 | docs/FDD.md | Integração | Worker entry point modeled after src/server.ts | CODIGO | src/server.ts |
| FDD-UUID-01 | docs/FDD.md | Restrição | IDs da outbox são UUID, seguindo padrão do projeto (confirmado após encerramento) | TRANSCRICAO | [09:51] Larissa |
| FDD-LIMITE-01 | docs/FDD.md | Limitação Conhecida | Ordenação de eventos por pedido garantida apenas com single-worker | TRANSCRICAO | [09:12]-[09:13] Diego, Larissa |
| FDD-WORKER-01 | docs/FDD.md | Decisão | Worker cria PrismaClient próprio (mesmo DATABASE_URL, instância separada) | TRANSCRICAO | [09:30] Bruno |
| FDD-WORKER-02 | docs/FDD.md | Decisão | Entry point do worker: src/worker.ts, script npm run worker | TRANSCRICAO | [09:11] Larissa |
| PRD-SEGURANCA-01 | docs/PRD.md | Dependência | Revisão de segurança da Sofia (HMAC e geração de secret) antes do deploy | TRANSCRICAO | [09:46] Sofia |
| PRD-PRAZO-01 | docs/PRD.md | Dependência | Estimativa de 3 sprints incluindo revisão de Sofia | TRANSCRICAO | [09:46] Larissa |
| FDD-CRUD-AUTH-01 | docs/FDD.md | Restrição | customer_id passado no body/path, não extraído do JWT (JWT é do usuário operador, não do cliente) | TRANSCRICAO | [09:32] Larissa |
| FDD-CRUD-AUTH-02 | docs/FDD.md | Restrição | CRUD de configuração de webhook acessível com qualquer role autenticada (por ora) | TRANSCRICAO | [09:36]-[09:37] Sofia, Marcos |
