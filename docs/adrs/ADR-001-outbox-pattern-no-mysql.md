# ADR-001 — Padrão Outbox no MySQL para Criação Atômica de Eventos

**Status:** Aceito  
**Data:** 2026-07-21  
**Participantes:** Larissa (Tech Lead), Bruno (Engenheiro Pleno), Diego (Engenheiro Sênior)

---

## Contexto

O sistema precisa notificar clientes externos (B2B) quando o status de um pedido muda. A pergunta central é: como garantir que o evento de notificação seja criado **sempre e somente** quando a mudança de status acontece, sem inconsistências entre o estado do banco e o evento gerado?

O método `changeStatus` em `src/modules/orders/order.service.ts` já executa uma transação complexa: atualiza o status na tabela `orders`, insere registro em `order_status_history` e, dependendo da transição, debita ou repõe estoque em `products`. Qualquer mecanismo de notificação precisa se encaixar nesse fluxo sem comprometer a atomicidade.

A opção mais simples seria fazer o envio HTTP do webhook dentro do próprio service, de forma síncrona. Outra opção seria usar um sistema de mensageria externo (Redis Streams, RabbitMQ, Kafka).

## Decisão

Adotar o **padrão Transactional Outbox** com a tabela `webhook_outbox` persistida no mesmo banco MySQL existente.

Quando o status de um pedido muda, dentro da mesma transação SQL (`prisma.$transaction`) que já atualiza `orders`, insere em `order_status_history` e ajusta `products.stockQuantity`, também se insere uma linha em `webhook_outbox` com o evento serializado (snapshot do payload no momento da inserção). Um worker separado é responsável por ler essa tabela e despachar as chamadas HTTP — decisão detalhada no ADR-005.

## Alternativas Consideradas

### 1. Despacho HTTP síncrono dentro da transação

Realizar o envio HTTP para o endpoint do cliente como parte do próprio `changeStatus`, antes de commitar a transação.

**Por que foi descartado:** Qualquer lentidão ou indisponibilidade do cliente externo travaria a mudança de status para outros pedidos. Pior: se o cliente estiver fora do ar, seria necessário fazer rollback de uma mudança de status legítima ou aceitar que o status mudou mas o cliente não foi notificado — ambas opções inaceitáveis. [09:04] Bruno: *"Se o cliente tiver fora do ar, o que a gente faz, dá rollback na mudança de status? Não dá."*

### 2. Redis Streams / Message Broker externo

Publicar um evento num broker externo (Redis Streams, RabbitMQ, Kafka) após a mudança de status.

**Por que foi descartado:** Requer subir e operar infraestrutura adicional. O time é pequeno e o MySQL já existe. Qualquer broker externo adiciona complexidade operacional sem resolver o problema que o outbox resolve: a atomicidade entre a mudança de status e a criação do evento. [09:07] Diego: *"Subir Redis Cluster pra isso é overengineering. Outbox no MySQL existente resolve."*

## Consequências

**Positivas:**
- Atomicidade garantida: se a transação commitar, o evento existe na outbox; se der rollback, o evento some junto. Não há possibilidade de inconsistência.
- Sem infraestrutura adicional: mesmo banco, mesmo Prisma client, mesma stack.
- Modelo conhecido pelo time: segue exatamente o padrão de transações já usado no projeto.

**Negativas:**
- Latência mínima de entrega determinada pelo ciclo de polling do worker (2 segundos). Não há entrega instantânea.
- A tabela `webhook_outbox` cresce ao longo do tempo e precisará de estratégia de arquivamento (fora do escopo desta fase, [09:08] Diego).
- O worker precisa ser um processo separado com ciclo de vida próprio (ver ADR-005).

## Referências

- Transcrição: [09:04] Bruno, [09:06] Diego, [09:07] Diego, [09:07] Larissa, [09:08] Diego
- Código: `src/modules/orders/order.service.ts` — método `changeStatus`, bloco `prisma.$transaction`
- Código: `prisma/schema.prisma` — convenções de modelo seguidas pelas novas tabelas
- Relacionado: [ADR-005](ADR-005-worker-processo-separado-com-polling.md)
