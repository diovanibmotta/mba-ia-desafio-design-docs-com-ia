# FDD — Feature Design Document: Sistema de Webhooks de Notificação de Pedidos

**Versão:** 1.0  
**Data:** 2026-07-21  
**Autores:** Larissa (Tech Lead), Bruno (Engenheiro Pleno), Diego (Engenheiro Sênior)

---

## 1. Contexto e Motivação Técnica

O OMS atual não possui nenhum mecanismo de notificação externa. Clientes B2B são forçados a fazer polling periódico em `GET /api/v1/orders` para detectar mudanças de status — comportamento ineficiente e custoso para ambos os lados.

Este documento especifica a implementação do Sistema de Webhooks de Notificação de Pedidos: a feature que preenche esse vácuo, permitindo que clientes B2B recebam chamadas HTTP quando o status dos pedidos deles muda.

---

## 2. Objetivos Técnicos

- Entregar eventos de mudança de status com latência máxima de 10 segundos após a mudança.
- Garantir consistência absoluta: não pode existir mudança de status sem evento correspondente (e vice-versa).
- Suportar filtragem por tipo de status por endpoint de webhook.
- Autenticar cada entrega via HMAC-SHA256.
- Prover histórico de entregas e mecanismo de recuperação via DLQ.

---

## 3. Escopo e Exclusões

**Em escopo:**
- CRUD de endpoints de webhook
- Filtragem de eventos por status no momento da inserção na outbox
- Worker de despacho com retry e DLQ
- Autenticação HMAC-SHA256 com rotação de secret
- Endpoint de histórico de entregas
- Endpoint admin de replay de DLQ

**Fora de escopo:**
- Notificações por e-mail para clientes em caso de falhas ([09:37] Larissa)
- Rate limiting de envios por cliente ([09:39] Diego/Larissa)
- Dashboard visual de webhooks ([09:40] Larissa)
- Escalonamento para múltiplos workers ([09:13] Diego)
- Arquivamento de entradas entregues na outbox ([09:08] Diego)

---

## 4. Schema do Banco de Dados

Quatro novos modelos Prisma, seguindo as convenções existentes em `prisma/schema.prisma` (UUID `@db.Char(36)`, `@@map` snake_case, `@@index` em campos de query crítica).

### 4.1 WebhookEndpoint

Configuração de um endpoint de webhook registrado por um cliente.

```prisma
model WebhookEndpoint {
  id               String    @id @default(uuid()) @db.Char(36)
  customerId       String    @db.Char(36)
  url              String    @db.VarChar(2048)
  secret           String    @db.VarChar(255)
  previousSecret   String?   @db.VarChar(255)
  secretRotatedAt  DateTime?
  events           Json      // Array de OrderStatus: ex. ["SHIPPED", "DELIVERED"]
  active           Boolean   @default(true)
  createdAt        DateTime  @default(now())
  updatedAt        DateTime  @updatedAt

  customer         Customer  @relation(fields: [customerId], references: [id])
  outboxEvents     WebhookOutbox[]
  deliveryLogs     WebhookDeliveryLog[]
  deadLetters      WebhookDeadLetter[]

  @@index([customerId])
  @@index([active])
  @@map("webhook_endpoints")
}
```

### 4.2 WebhookOutbox

Fila de eventos aguardando despacho ou em retry.

```prisma
enum WebhookOutboxStatus {
  PENDING
  PROCESSING
  DELIVERED
  FAILED
}

model WebhookOutbox {
  id                  String              @id @default(uuid()) @db.Char(36)
  webhookEndpointId   String              @db.Char(36)
  eventType           String              @db.VarChar(100)  // ex: "order.status_changed"
  payload             Json                // Snapshot no momento da inserção
  status              WebhookOutboxStatus @default(PENDING)
  attempts            Int                 @default(0)
  nextRetryAt         DateTime?
  createdAt           DateTime            @default(now())
  processedAt         DateTime?

  webhookEndpoint     WebhookEndpoint     @relation(fields: [webhookEndpointId], references: [id])

  @@index([status, createdAt])
  @@index([nextRetryAt])
  @@map("webhook_outbox")
}
```

### 4.3 WebhookDeadLetter

Eventos que esgotaram todas as tentativas de entrega.

```prisma
model WebhookDeadLetter {
  id                  String    @id @default(uuid()) @db.Char(36)
  originalOutboxId    String    @db.Char(36)  // ID do evento na outbox (histórico)
  webhookEndpointId   String    @db.Char(36)
  eventType           String    @db.VarChar(100)
  payload             Json
  lastError           String    @db.Text
  failedAt            DateTime  @default(now())
  replayedAt          DateTime?
  replayedById        String?   @db.Char(36)

  webhookEndpoint     WebhookEndpoint @relation(fields: [webhookEndpointId], references: [id])
  replayedBy          User?     @relation(fields: [replayedById], references: [id])

  @@index([webhookEndpointId])
  @@map("webhook_dead_letters")
}
```

### 4.4 WebhookDeliveryLog

Histórico de cada tentativa de entrega (sucesso ou falha).

```prisma
model WebhookDeliveryLog {
  id                  String    @id @default(uuid()) @db.Char(36)
  outboxId            String    @db.Char(36)
  webhookEndpointId   String    @db.Char(36)
  statusCode          Int?
  responseBody        String?   @db.Text
  responseTimeMs      Int
  success             Boolean
  error               String?   @db.Text
  deliveredAt         DateTime  @default(now())

  webhookEndpoint     WebhookEndpoint @relation(fields: [webhookEndpointId], references: [id])

  @@index([webhookEndpointId, deliveredAt])
  @@index([outboxId])
  @@map("webhook_delivery_logs")
}
```

---

## 5. Fluxos Detalhados

### 5.1 Criação do Evento na Outbox

Acionado dentro do método `changeStatus` de `src/modules/orders/order.service.ts`.

```
changeStatus(id, { toStatus, reason }, userId)
  │
  └─ prisma.$transaction(async (tx) => {
       1. findFirst order (com items) — já existente
       2. canTransition(from, to) — já existente
       3. debitStock/replenishStock — já existente
       4. tx.order.update(status) — já existente
       5. tx.orderStatusHistory.create — já existente
       6. [NOVO] publishWebhookEvent(tx, order, fromStatus, toStatus)
            │
            ├─ Query: webhook_endpoints WHERE customerId = order.customerId
            │          AND active = true
            │          AND JSON_CONTAINS(events, '"<toStatus>"')
            │
            └─ Para cada endpoint matching:
                 tx.webhookOutbox.create({
                   id: uuid(),           ← gerado aqui, será o X-Event-Id
                   webhookEndpointId: endpoint.id,
                   eventType: "order.status_changed",
                   payload: {            ← snapshot no momento da inserção [09:52]
                     event_id: <uuid>,
                     event_type: "order.status_changed",
                     timestamp: new Date().toISOString(),
                     data: {
                       order_id: order.id,
                       order_number: order.orderNumber,
                       customer_id: order.customerId,
                       from_status: fromStatus,
                       to_status: toStatus,
                       total_cents: order.totalCents
                     }
                   },
                   status: "PENDING",
                   attempts: 0
                 })
     })
```

**Ponto crítico:** `publishWebhookEvent` recebe o `tx` (Prisma transaction client) — não o `prisma` global. A inserção na outbox é atômica com a mudança de status. Se o `$transaction` der rollback por qualquer motivo, o evento some junto.

A filtragem por status acontece aqui (na inserção), não na hora do envio — se nenhum endpoint do customer quer aquele status, nenhuma linha é inserida na outbox. [09:34] Bruno.

### 5.2 Processamento pelo Worker

**Entry point:** `src/worker.ts` (novo arquivo, padrão análogo a `src/server.ts`).

```
bootstrap worker:
  1. Criar PrismaClient próprio (mesmo DATABASE_URL, instância separada) [09:30] Bruno
  2. Criar logger Pino (mesma configuração de src/shared/logger/index.ts)
  3. Iniciar loop de polling

Poll loop (a cada 2 segundos):
  1. Query: SELECT * FROM webhook_outbox
            WHERE status = 'PENDING'
              AND (nextRetryAt IS NULL OR nextRetryAt <= NOW())
            ORDER BY createdAt ASC
            LIMIT 10

  2. Para cada evento:
     a. UPDATE webhook_outbox SET status = 'PROCESSING' WHERE id = ?
     b. Buscar webhookEndpoint (para obter url e secret)
     c. Construir headers:
          X-Event-Id: <outbox.id>
          X-Signature: sha256=<HMAC-SHA256(secret, JSON.stringify(payload))>
          X-Timestamp: <ISO 8601 atual>
          X-Webhook-Id: <webhookEndpointId>
          Content-Type: application/json
     d. HTTP POST para endpoint.url com timeout de 10s [09:42] Diego
     e. Registrar resultado em webhook_delivery_logs

  3. Em caso de sucesso (resposta 2xx):
     - UPDATE webhook_outbox SET status = 'DELIVERED', processedAt = NOW()

  4. Em caso de falha (não-2xx, timeout, erro de rede):
     - attempts++
     - Se attempts < 5: calcular nextRetryAt por tabela de backoff,
                        UPDATE status = 'PENDING', nextRetryAt = ?
     - Se attempts >= 5: mover para webhook_dead_letter,
                         DELETE from webhook_outbox (ou UPDATE status = FAILED)
```

**Backoff schedule:**

| Tentativa (attempts) | nextRetryAt (offset do momento da falha) |
|----------------------|------------------------------------------|
| 1                    | + 1 minuto                               |
| 2                    | + 5 minutos                              |
| 3                    | + 30 minutos                             |
| 4                    | + 2 horas                                |
| 5 (última)           | + 12 horas                               |

**Recuperação após crash do worker:** eventos com status `PROCESSING` com `updatedAt` mais antigo que o timeout (ex: 30s) devem ser revertidos para `PENDING` na inicialização do worker.

### 5.3 Assinatura HMAC-SHA256

```typescript
import { createHmac } from 'crypto';

function signPayload(secret: string, body: string): string {
  return 'sha256=' + createHmac('sha256', secret).update(body).digest('hex');
}

// Uso no worker:
const bodyString = JSON.stringify(outboxEvent.payload);
const signature  = signPayload(endpoint.secret, bodyString);
// Header: X-Signature: sha256=<hex>
```

Durante o grace period de rotação (24h após `secretRotatedAt`):
- O worker assina com a `secret` atual (nova).
- O `previousSecret` continua disponível para o cliente verificar caso ainda use a chave antiga.
- Após 24h, `previousSecret` é nulificado.

### 5.4 Replay de DLQ

```
POST /api/v1/admin/webhooks/dead-letter/:id/replay
  │  (requer authenticate + requireRole('ADMIN'))
  │
  ├─ Buscar WebhookDeadLetter por id (lança WEBHOOK_DEAD_LETTER_NOT_FOUND se não existir)
  ├─ Criar novo registro em webhook_outbox com:
  │    - id: novo uuid()
  │    - webhookEndpointId: deadLetter.webhookEndpointId
  │    - eventType: deadLetter.eventType
  │    - payload: deadLetter.payload  ← mesmo snapshot original
  │    - status: PENDING
  │    - attempts: 0
  ├─ Atualizar webhook_dead_letter:
  │    - replayedAt: NOW()
  │    - replayedById: req.user.id  ← auditoria [09:36] Sofia
  └─ Retornar { outboxId: novoId, status: "PENDING" }
```

---

## 6. Contratos Públicos

### 6.1 POST /api/v1/webhooks — Cadastrar Webhook

Cria um endpoint de webhook para um cliente.

**Auth:** `authenticate`

**Request:**
```json
POST /api/v1/webhooks
Content-Type: application/json
Authorization: Bearer <token>

{
  "customerId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "url": "https://erp.atlascomercial.com.br/webhooks/oms",
  "events": ["SHIPPED", "DELIVERED", "CANCELLED"]
}
```

**Validações:**
- `customerId`: UUID válido, customer deve existir
- `url`: string URL válida, obrigatoriamente `https://` ([09:23] Sofia)
- `events`: array não-vazio de valores de `OrderStatus`

**Response 201:**
```json
{
  "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "customerId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "url": "https://erp.atlascomercial.com.br/webhooks/oms",
  "secret": "wh_live_xK9mP2nQrT7vZ4bL8cD1eF6gH3iJ5k",
  "events": ["SHIPPED", "DELIVERED", "CANCELLED"],
  "active": true,
  "createdAt": "2026-07-21T10:00:00.000Z"
}
```

> A `secret` é retornada **somente na criação**. Não é possível recuperá-la depois. Em chamadas subsequentes, o campo `secret` não aparece na resposta.

**Errors:**
- `400 WEBHOOK_INVALID_URL` — URL não começa com `https://`
- `404 WEBHOOK_CUSTOMER_NOT_FOUND` — customer não existe

---

### 6.2 PATCH /api/v1/webhooks/:id — Atualizar Webhook

**Auth:** `authenticate`

**Request:**
```json
PATCH /api/v1/webhooks/a1b2c3d4-e5f6-7890-abcd-ef1234567890
Content-Type: application/json
Authorization: Bearer <token>

{
  "url": "https://erp.atlascomercial.com.br/webhooks/oms-v2",
  "events": ["SHIPPED", "DELIVERED"],
  "active": false
}
```

Todos os campos são opcionais (patch parcial).

**Response 200:**
```json
{
  "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "customerId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "url": "https://erp.atlascomercial.com.br/webhooks/oms-v2",
  "events": ["SHIPPED", "DELIVERED"],
  "active": false,
  "updatedAt": "2026-07-21T11:00:00.000Z"
}
```

**Errors:**
- `404 WEBHOOK_NOT_FOUND`
- `400 WEBHOOK_INVALID_URL`

---

### 6.3 DELETE /api/v1/webhooks/:id — Remover Webhook

**Auth:** `authenticate`

**Response:** `204 No Content`

**Errors:**
- `404 WEBHOOK_NOT_FOUND`

---

### 6.4 GET /api/v1/webhooks?customerId=uuid — Listar Webhooks

**Auth:** `authenticate`

**Query params:** `customerId` (obrigatório), `page` (default 1), `pageSize` (default 20, max 100)

**Response 200:**
```json
{
  "data": [
    {
      "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "customerId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
      "url": "https://erp.atlascomercial.com.br/webhooks/oms",
      "events": ["SHIPPED", "DELIVERED", "CANCELLED"],
      "active": true,
      "createdAt": "2026-07-21T10:00:00.000Z",
      "updatedAt": "2026-07-21T10:00:00.000Z"
    }
  ],
  "total": 1,
  "page": 1,
  "pageSize": 20
}
```

---

### 6.5 POST /api/v1/webhooks/:id/rotate-secret — Rotacionar Secret

**Auth:** `authenticate`

**Response 200:**
```json
{
  "secret": "wh_live_nR8sU1wYpX5qA3mB9dE2fG7hI4jK6l",
  "previousSecretExpiresAt": "2026-07-22T10:00:00.000Z"
}
```

Internamente: move `secret` para `previousSecret`, gera nova `secret`, seta `secretRotatedAt = NOW()`.

**Errors:**
- `404 WEBHOOK_NOT_FOUND`

---

### 6.6 GET /api/v1/webhooks/:id/deliveries — Histórico de Entregas

**Auth:** `authenticate`

**Query params:** `page` (default 1), `pageSize` (default 20, max 100)

**Response 200:**
```json
{
  "data": [
    {
      "id": "d1e2f3a4-b5c6-7890-abcd-ef0123456789",
      "outboxId": "e1f2a3b4-c5d6-7890-abcd-ef0123456789",
      "statusCode": 200,
      "responseTimeMs": 342,
      "success": true,
      "deliveredAt": "2026-07-21T10:00:02.411Z"
    },
    {
      "id": "f1a2b3c4-d5e6-7890-abcd-ef0123456789",
      "outboxId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "statusCode": 503,
      "responseTimeMs": 10001,
      "success": false,
      "error": "Request timeout after 10000ms",
      "deliveredAt": "2026-07-21T10:02:15.000Z"
    }
  ],
  "total": 2,
  "page": 1,
  "pageSize": 20
}
```

---

### 6.7 POST /api/v1/admin/webhooks/dead-letter/:id/replay — Replay de DLQ

**Auth:** `authenticate` + `requireRole('ADMIN')`

**Response 200:**
```json
{
  "outboxId": "b2c3d4e5-f6a7-8901-bcde-f01234567891",
  "status": "PENDING"
}
```

**Errors:**
- `404 WEBHOOK_DEAD_LETTER_NOT_FOUND`
- `403 FORBIDDEN` — role insuficiente

---

### 6.8 Payload de Entrega ao Cliente

O cliente recebe um `HTTP POST` com o seguinte formato:

**Headers enviados:**
```
Content-Type: application/json
X-Event-Id: e1f2a3b4-c5d6-7890-abcd-ef0123456789
X-Signature: sha256=a3f8b2c1d9e4f7a6b5c0d8e3f2a1b9c4d7e6f5a4b3c2d1e0f9a8b7c6d5e4f3
X-Timestamp: 2026-07-21T10:00:00.412Z
X-Webhook-Id: a1b2c3d4-e5f6-7890-abcd-ef1234567890
```

**Body:**
```json
{
  "event_id": "e1f2a3b4-c5d6-7890-abcd-ef0123456789",
  "event_type": "order.status_changed",
  "timestamp": "2026-07-21T10:00:00.000Z",
  "data": {
    "order_id": "c3d4e5f6-a7b8-9012-cdef-012345678901",
    "order_number": "ORD-000042",
    "customer_id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
    "from_status": "PROCESSING",
    "to_status": "SHIPPED",
    "total_cents": 15000
  }
}
```

> Items do pedido não são incluídos no payload. Se o cliente precisar de detalhes, deve chamar `GET /api/v1/orders/:id`. [09:43] Diego.

**Resposta esperada do cliente:** qualquer HTTP 2xx. O worker trata como falha qualquer status não-2xx, timeout (>10s) ou erro de conexão.

---

## 7. Matriz de Erros

Todos os erros seguem o padrão `{ "error": { "code": "...", "message": "..." } }` via `errorMiddleware` existente (`src/middlewares/error.middleware.ts`).

| Código | HTTP | Descrição | Quando ocorre |
|--------|------|-----------|---------------|
| `WEBHOOK_NOT_FOUND` | 404 | Endpoint de webhook não encontrado | GET/PATCH/DELETE com id inválido |
| `WEBHOOK_INVALID_URL` | 400 | URL não é HTTPS ou não é URL válida | POST/PATCH com URL inválida |
| `WEBHOOK_URL_REQUIRED` | 400 | Campo `url` obrigatório ausente | POST sem `url` |
| `WEBHOOK_EVENTS_REQUIRED` | 400 | Lista de eventos vazia ou ausente | POST sem `events` |
| `WEBHOOK_CUSTOMER_NOT_FOUND` | 404 | Customer referenciado não existe | POST com `customerId` inválido |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | Entrada da DLQ não encontrada | Replay com id inválido |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 400 | Payload do evento excede 64KB | Inserção na outbox com payload gigante |
| `WEBHOOK_DELIVERY_TIMEOUT` | — | Timeout de 10s excedido (interno, registrado no log) | Worker: cliente não respondeu |

---

## 8. Estratégias de Resiliência

| Aspecto | Estratégia | Valor |
|---------|-----------|-------|
| Timeout HTTP | Abortar requisição após | 10 segundos [09:42] |
| Limite de payload | Rejeitar inserção se payload > | 64KB [09:24] |
| Retries | Máximo de tentativas por evento | 5 tentativas [09:15] |
| Backoff | Schedule fixo | 1m / 5m / 30m / 2h / 12h [09:17] |
| DLQ | Destino após retries esgotados | `webhook_dead_letters` |
| Recovery | Worker reiniciado: eventos PROCESSING > 30s | Revertidos para PENDING |
| Batch size | Eventos processados por ciclo | 10 (configurável) |
| Poll interval | Frequência de leitura da outbox | 2 segundos [09:09] |

---

## 9. Observabilidade

### 9.1 Logs (Pino)

O worker usa Pino (`src/shared/logger/index.ts`) em sua instância própria. Logs estruturados em JSON, com `requestId` (= `X-Event-Id`) como campo de correlação.

**Eventos de log relevantes:**

```jsonc
// Ciclo de polling
{ "level": "debug", "msg": "poll cycle", "found": 3 }

// Tentativa de entrega
{ "level": "info", "msg": "webhook delivery attempt",
  "eventId": "uuid", "webhookId": "uuid", "attempt": 1,
  "url": "https://...", "statusCode": 200, "durationMs": 342 }

// Falha com retry agendado
{ "level": "warn", "msg": "webhook delivery failed, scheduling retry",
  "eventId": "uuid", "attempt": 2, "nextRetryAt": "2026-07-21T10:05:00Z",
  "error": "connect ECONNREFUSED" }

// Evento movido para DLQ
{ "level": "error", "msg": "webhook max retries exhausted, moving to DLQ",
  "eventId": "uuid", "webhookId": "uuid", "attempts": 5 }

// Replay de DLQ
{ "level": "info", "msg": "DLQ entry replayed",
  "deadLetterId": "uuid", "replayedBy": "userId", "newOutboxId": "uuid" }
```

### 9.2 Métricas (para implementação futura de APM)

| Métrica | Tipo | Descrição |
|---------|------|-----------|
| `webhook_deliveries_total` | Counter | Total de tentativas por `status` (success/failure) |
| `webhook_delivery_duration_ms` | Histogram | Tempo de resposta do cliente |
| `webhook_outbox_pending_count` | Gauge | Eventos aguardando entrega |
| `webhook_dead_letter_count` | Gauge | Eventos na DLQ sem replay |
| `webhook_retry_attempts_total` | Counter | Total de retries por tentativa (1-5) |

### 9.3 Tracing

O `X-Event-Id` (UUID do evento na outbox) serve como correlation ID:
- Presente no payload enviado ao cliente (`event_id`)
- Presente no header `X-Event-Id` de cada entrega
- Presente em todos os logs do worker relacionados ao evento
- Presente em `webhook_delivery_logs.outboxId`

---

## 10. Integração com o Sistema Existente

### 10.1 `src/modules/orders/order.service.ts`

Ponto de integração principal. O método `changeStatus` (linha ~126) executa um `prisma.$transaction()` que atualiza `orders`, insere em `order_status_history` e ajusta `products.stockQuantity`. A integração consiste em adicionar, ao final desse bloco de transação, a chamada:

```typescript
await publishWebhookEvent(tx, order, fromStatus, toStatus);
```

A função `publishWebhookEvent` é importada de `src/modules/webhooks/webhook.outbox.ts` e recebe o `tx` (Prisma.TransactionClient) — não o `prisma` global — garantindo que a inserção na outbox seja atômica com a mudança de status. Nenhuma outra linha do arquivo precisa ser modificada.

### 10.2 `src/shared/errors/app-error.ts` e `src/shared/errors/http-errors.ts`

Os novos erros do módulo de webhooks estendem a hierarquia existente:

```typescript
// Exemplos de novas classes em src/shared/errors/http-errors.ts
export class WebhookNotFoundError extends NotFoundError {
  constructor() {
    super('Webhook');
    this.errorCode = 'WEBHOOK_NOT_FOUND';
  }
}

export class WebhookInvalidUrlError extends BadRequestError {
  constructor() {
    super('URL must be HTTPS', 'WEBHOOK_INVALID_URL');
  }
}
```

O `errorMiddleware` existente em `src/middlewares/error.middleware.ts` já trata qualquer instância de `AppError` — nenhuma alteração é necessária nesse arquivo.

### 10.3 `src/middlewares/auth.middleware.ts`

Todas as rotas do módulo de webhooks usam o middleware `authenticate` existente. O endpoint de replay de DLQ adiciona `requireRole('ADMIN')`:

```typescript
// Em src/modules/webhooks/webhook.routes.ts
router.patch('/:id', authenticate, validate({ params, body }), controller.update);
router.post('/dead-letter/:id/replay', authenticate, requireRole('ADMIN'), controller.replay);
```

Esse padrão é idêntico ao usado em `src/modules/users/user.routes.ts` para proteger `GET /users/:id`.

### 10.4 `src/middlewares/validate.middleware.ts`

Os schemas Zod do módulo de webhooks são aplicados via middleware `validate`, exatamente como nos outros módulos:

```typescript
// Em src/modules/webhooks/webhook.schemas.ts
export const createWebhookSchema = z.object({
  customerId: z.string().uuid(),
  url: z.string().url().refine(u => u.startsWith('https://'), {
    message: 'URL must use HTTPS'
  }),
  events: z.array(z.nativeEnum(OrderStatus)).min(1),
});
```

### 10.5 `src/app.ts`

A função `buildControllers` é extendida com as novas dependências do módulo de webhooks:

```typescript
// Adição em src/app.ts
const webhookRepository = new WebhookRepository(prisma);
const webhookService    = new WebhookService(webhookRepository, prisma);
const webhookController = new WebhookController(webhookService);
```

O router é montado em `src/routes/index.ts`:

```typescript
router.use('/webhooks', buildWebhookRouter(controllers.webhooks));
```

### 10.6 `prisma/schema.prisma`

Os 4 novos modelos seguem exatamente as convenções do schema existente: `@id @default(uuid()) @db.Char(36)`, `@@map("snake_case")`, `createdAt @default(now())`, `updatedAt @updatedAt`, `@@index` nos campos de filtro mais usados.

### 10.7 `src/shared/logger/index.ts`

O worker (`src/worker.ts`) importa o mesmo `createLogger` e instancia seu próprio logger:

```typescript
import { createLogger } from './shared/logger';
const logger = createLogger(); // mesma config, nova instância — worker é processo separado
```

---

## 11. Critérios de Aceite Técnicos

- [ ] Inserção na outbox e mudança de status estão na mesma transação — teste de rollback confirma atomicidade
- [ ] Evento com `status = PENDING` é processado pelo worker em até 4 segundos do commit (2s poll + overhead)
- [ ] Falha de entrega incrementa `attempts` e agenda `nextRetryAt` conforme tabela de backoff
- [ ] Após 5 falhas consecutivas, evento aparece em `webhook_dead_letters` e some da outbox
- [ ] Replay cria novo registro na outbox com `attempts = 0`
- [ ] Replay registra `replayedBy` com o `userId` do admin
- [ ] URL com `http://` (sem S) é rejeitada com `400 WEBHOOK_INVALID_URL`
- [ ] Assinatura `X-Signature` pode ser verificada pelo cliente com `HMAC-SHA256(secret, body)`
- [ ] Rotação de secret mantém `previousSecret` válido por 24h
- [ ] Evento não é inserido na outbox se nenhum endpoint do customer tem aquele status na lista `events`
- [ ] Payload não inclui items do pedido (apenas campos listados em seção 6.8)
- [ ] Replay de DLQ requer role `ADMIN`; retorna `403` para roles inferiores

---

## 12. Riscos e Mitigação

| Risco | Probabilidade | Impacto | Mitigação |
|-------|--------------|---------|-----------|
| Worker parado = entregas paradas | Médio | Alto | Supervisor de processo (systemd/pm2), alert em gap de polling |
| Outbox cresce sem arquivamento | Médio (longo prazo) | Médio | Índice em `(status, createdAt)` mantém queries rápidas; archiving planeja após observar volumetria |
| Ordering entre pedidos diferentes não garantida | Baixo (aceitável) | Baixo | Documentado; clientes B2B precisam de ordering por pedido, não global |
| Secret armazenada em plaintext | Médio | Alto | Revisar antes do deploy: hash da secret ou criptografia em repouso; Sofia revisará ([09:46]) |

---

*Para a proposta técnica em nível arquitetural, ver [RFC.md](RFC.md). Para as decisões isoladas, ver [docs/adrs/](adrs/).*
