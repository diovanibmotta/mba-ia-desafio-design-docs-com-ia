# RFC — Sistema de Webhooks de Notificação de Pedidos

| Campo      | Valor                                                     |
|------------|-----------------------------------------------------------|
| **Autor**  | Larissa (Tech Lead)                                       |
| **Status** | Em Revisão                                                |
| **Data**   | 2026-07-21                                                |
| **Revisores** | Marcos (PM), Bruno (Engenheiro Pleno), Diego (Engenheiro Sênior), Sofia (Engenheira de Segurança) |

---

## TL;DR

Três clientes B2B estão integrando com o OMS via polling, o que é custoso e lento. Proposta: implementar webhooks outbound — chamadas HTTP disparadas pelo sistema quando o status de um pedido muda. A solução usa o padrão Transactional Outbox no MySQL existente para garantir consistência, um worker em processo separado para despachar os eventos, HMAC-SHA256 para autenticação de cada entrega e garantia at-least-once com deduplicação por `X-Event-Id`.

---

## Contexto e Problema

Os clientes B2B **Atlas Comercial**, **MaxDistribuicao** e **Nova Cargo** precisam ser notificados quando o status dos pedidos deles muda no OMS. Atualmente, eles consomem o endpoint `GET /api/v1/orders` periodicamente para verificar mudanças — prática conhecida como polling.

Isso gera dois problemas concretos:
1. **Custo e latência para os clientes**: integração lenta e cara, dependente de frequência de polling.
2. **Risco de negócio**: a Atlas Comercial formalizou que pode migrar para um concorrente caso a plataforma não entregue notificação em tempo real até o fim do trimestre.

O requisito de latência é claro: qualquer entrega abaixo de 10 segundos da mudança de status é aceitável para os clientes. Os webhooks são exclusivamente **outbound** (do OMS para os clientes), sem recepção de eventos externos.

---

## Proposta Técnica

### Visão Geral da Solução

```
[order.service.ts]
  changeStatus()
  └─ prisma.$transaction()
       ├─ UPDATE orders SET status = ...
       ├─ INSERT order_status_history
       ├─ UPDATE products.stock_quantity (se aplicável)
       └─ INSERT webhook_outbox (evento snapshot)

[webhook worker — processo separado]
  Poll a cada 2s ← webhook_outbox (PENDING)
  │
  ├─ HTTP POST → endpoint do cliente
  │    Headers: X-Event-Id, X-Signature (HMAC-SHA256), X-Timestamp, X-Webhook-Id
  │
  ├─ Sucesso (2xx) → marca DELIVERED, registra em webhook_delivery_logs
  └─ Falha → retry com backoff (1m/5m/30m/2h/12h)
               └─ após 5 tentativas → webhook_dead_letter
```

### Componentes Principais

**1. Transactional Outbox**  
O evento de webhook é escrito na tabela `webhook_outbox` dentro da mesma transação SQL que muda o status do pedido. O payload é um snapshot do estado do pedido no momento da mudança — não é renderizado na hora do envio. Isso garante consistência total: não existe mudança de status sem evento correspondente, e eventos "fantasma" são impossíveis. *(Ver [ADR-001](adrs/ADR-001-outbox-pattern-no-mysql.md))*

**2. Worker em Processo Separado**  
Um processo Node.js independente (`src/worker.ts`) faz polling na `webhook_outbox` a cada 2 segundos, buscando eventos pendentes. Roda com `npm run worker`, cria sua própria instância de `PrismaClient` e tem ciclo de vida isolado da API. *(Ver [ADR-005](adrs/ADR-005-worker-processo-separado-com-polling.md))*

**3. Retry com Backoff e DLQ**  
Falhas de entrega são retentadas até 5 vezes com intervalos crescentes. Eventos que esgotam as tentativas vão para `webhook_dead_letter`. Administradores podem reprocessar entradas da DLQ via endpoint admin. *(Ver [ADR-002](adrs/ADR-002-retry-com-backoff-e-dlq.md))*

**4. Autenticação HMAC-SHA256**  
Cada endpoint de webhook recebe uma secret única gerada pelo sistema. O worker assina o corpo de cada requisição com `HMAC-SHA256(secret, body)` e envia a assinatura no header `X-Signature`. Suporte a rotação de secret com grace period de 24h. *(Ver [ADR-003](adrs/ADR-003-hmac-sha256-com-secret-por-endpoint.md))*

**5. At-Least-Once com X-Event-Id**  
Cada evento na outbox recebe um UUID gerado na inserção. Esse UUID é enviado no header `X-Event-Id` em toda tentativa de entrega. Clientes usam o `X-Event-Id` para deduplicar recebimentos. *(Ver [ADR-004](adrs/ADR-004-garantia-at-least-once-com-event-id.md))*

**6. Integração com Padrões Existentes**  
O módulo de webhooks (`src/modules/webhooks/`) segue exatamente a estrutura dos módulos existentes. Erros usam a hierarquia `AppError` com prefixo `WEBHOOK_`. Logging via Pino. Validação via Zod. Autenticação via `authenticate` e `requireRole`. *(Ver [ADR-006](adrs/ADR-006-reuso-padroes-existentes-do-projeto.md))*

---

## Alternativas Consideradas

### Alternativa 1: Despacho HTTP Síncrono Dentro da Transação

Incluir a chamada HTTP ao cliente diretamente no método `changeStatus`, antes do commit da transação.

**Trade-off que levou ao descarte:** Acoplamento direto entre a confiabilidade da mudança de status e a disponibilidade do cliente externo. Se o cliente estiver lento ou fora do ar, a mudança de status de pedidos seria bloqueada ou revertida — consequência inaceitável. A alternativa também tornaria impossível o retry sem reverter a transação. Descartada em [09:04] por Bruno e Larissa.

### Alternativa 2: Redis Streams / Message Broker Externo

Publicar eventos em um broker externo (Redis Streams, RabbitMQ, Kafka) após a mudança de status, com um consumidor do broker fazendo o despacho.

**Trade-off que levou ao descarte:** Requer provisionamento e operação de infraestrutura adicional. O time é pequeno e a complexidade operacional supera o benefício. Mais importante: um broker externo não resolve o problema de atomicidade entre a mudança de status e a criação do evento — seria necessário um outbox de qualquer forma, ou aceitar a possibilidade de perda de eventos. O MySQL existente resolve o problema com menos partes móveis. Descartada em [09:07] por Diego e Larissa.

---

## Questões em Aberto

### 1. Rate Limiting de Saída por Cliente

Se um cliente tem 50 pedidos mudando de status em um minuto, o worker disparará 50 chamadas HTTP para o mesmo endpoint em rápida sucessão. Isso pode sobrecarregar endpoints de clientes menos robustos.

**Situação:** Acordado observar o comportamento em produção antes de implementar qualquer limite. Se o volume causar problemas, rate limiting por endpoint será considerado na próxima iteração. *[09:38]-[09:39] Diego, Larissa.*

### 2. Notificação Proativa de Falhas Recorrentes

Quando um webhook falha repetidamente (ex: 3 tentativas consecutivas sem sucesso), seria útil notificar o responsável técnico do cliente por e-mail, para que ele investigue o endpoint antes de o evento chegar à DLQ.

**Situação:** Explicitamente fora de escopo desta fase. Marcos pode documentar no portal do desenvolvedor que clientes devem monitorar o endpoint `/api/v1/webhooks/:id/deliveries` para detectar falhas. *[09:37] Larissa.*

### 3. Estratégia de Arquivamento da Outbox

Eventos com status `DELIVERED` acumulam na `webhook_outbox` ao longo do tempo. Diego mencionou arquivamento após 30 dias, mas sem definição de mecanismo.

**Situação:** Fora de escopo desta fase. A tabela deve ser monitorada em produção e uma estratégia de archiving/purge será definida antes que o volume se torne problema. *[09:08] Diego.*

---

## Impacto e Riscos

| Risco | Probabilidade | Impacto | Mitigação |
|-------|--------------|---------|-----------|
| Worker como único ponto de falha para entregas | Médio | Alto | Supervisor de processo, health check, alertas de monitoramento |
| Outbox crescendo sem controle em alta volumetria | Baixo (curto prazo) | Médio | Monitorar tamanho da tabela, planejar archiving |
| Garantia de ordenação perdida se houver múltiplos workers | Baixo (por decisão de design) | Médio | Documentado como limitação; escala horizontal requer redesign |
| Vazamento de secret do cliente | Baixo | Alto | Secrets por endpoint, rotação disponível, revisão de segurança |

**Impacto na API existente:** Mínimo. A única alteração no código existente é a adição de uma chamada a `publishWebhookEvent()` dentro do bloco `$transaction` do método `changeStatus` em `src/modules/orders/order.service.ts`.

---

## Decisões Relacionadas

- [ADR-001](adrs/ADR-001-outbox-pattern-no-mysql.md) — Padrão Outbox no MySQL
- [ADR-002](adrs/ADR-002-retry-com-backoff-e-dlq.md) — Retry com Backoff e DLQ
- [ADR-003](adrs/ADR-003-hmac-sha256-com-secret-por-endpoint.md) — HMAC-SHA256 com Secret por Endpoint
- [ADR-004](adrs/ADR-004-garantia-at-least-once-com-event-id.md) — At-Least-Once com X-Event-Id
- [ADR-005](adrs/ADR-005-worker-processo-separado-com-polling.md) — Worker em Processo Separado com Polling
- [ADR-006](adrs/ADR-006-reuso-padroes-existentes-do-projeto.md) — Reuso dos Padrões Existentes do Projeto
