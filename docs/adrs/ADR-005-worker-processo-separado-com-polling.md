# ADR-005 — Worker em Processo Separado com Polling

**Status:** Aceito  
**Data:** 2026-07-21  
**Participantes:** Larissa (Tech Lead), Diego (Engenheiro Sênior), Bruno (Engenheiro Pleno)

---

## Contexto

Os eventos registrados na `webhook_outbox` precisam ser lidos e despachados para os endpoints dos clientes. É necessário decidir: onde roda o consumidor da outbox, como ele é acionado e com qual frequência.

Opções principais: rodar dentro do processo da API (mesma instância), usar triggers do banco, ou rodar como processo Node.js separado em polling.

## Decisão

O worker roda como um **processo Node.js separado da API**, com um loop de polling a cada **2 segundos**.

- Entry point: `src/worker.ts` (novo arquivo, seguindo o padrão de `src/server.ts`).
- Script npm: `npm run worker`.
- O worker cria sua **própria instância de `PrismaClient`** (mesmo `DATABASE_URL`, instância nova por ser processo separado).
- A cada ciclo: busca eventos com status `PENDING` e `nextRetryAt <= NOW()`, em lote pequeno, ordenados por `createdAt ASC`, processa e atualiza status.
- Um único worker em execução garante ordenação implícita por `order_id` e `created_at`.
- Escalonamento para múltiplos workers é uma limitação conhecida e documentada (ver Consequências).

## Alternativas Consideradas

### 1. Worker in-process (mesmo processo da API)

Rodar o consumidor da outbox dentro da mesma instância Express da API, por exemplo via `setInterval`.

**Por que foi descartado:** Se a API reiniciar (deploy, crash), o worker para junto. Eventos ficam sem processamento até a API voltar e o ciclo recomeçar. [09:11] Diego: *"O worker tem que rodar como processo separado, não dentro da mesma instância da API. Senão se a API reinicia, perde o worker."*

### 2. Trigger de banco de dados

Usar trigger MySQL para notificar o worker quando um novo evento entra na outbox.

**Por que foi descartado:** MySQL não tem mecanismo nativo de `NOTIFY/LISTEN` como o PostgreSQL. Um trigger MySQL executa SQL dentro da transação, mas não tem como notificar um processo externo de forma confiável. Workarounds (escrever em arquivo, chamar endpoint) foram considerados frágeis. [09:09] Diego: *"MySQL não tem listener nativo tipo o NOTIFY/LISTEN do Postgres. [...] fica esquisito. Polling de 2 segundos atende o requisito de 'abaixo de 10 segundos' tranquilo."*

### 3. Múltiplos workers em paralelo

Escalar horizontalmente o worker para aumentar throughput.

**Por que foi adiado:** Múltiplos workers processando em paralelo quebrariam a garantia de ordenação de eventos por pedido (eventos do mesmo pedido poderiam ser entregues fora de ordem). Precisaria de particionamento por `order_id` ou locks pessimistas. [09:13] Diego: *"Isso é problema do futuro, não agora."* Documentado como limitação conhecida.

## Consequências

**Positivas:**
- Domínio de falha isolado: API e worker caem independentemente.
- 2s polling atende o SLA de entrega abaixo de 10 segundos.
- Sem infraestrutura adicional (não precisa de broker, apenas o MySQL existente).
- Único worker garante ordenação de eventos por `created_at` por enquanto.

**Negativas:**
- Dois processos para gerenciar e monitorar no deploy.
- Latência mínima de 2 segundos (floor de polling).
- Single point of failure: se o worker cair, entregas param até reinício.
- **Limitação conhecida**: ordenação de eventos por pedido só é garantida enquanto houver um único worker. Escalar para múltiplos workers exige trabalho adicional (particionamento ou locks por `order_id`).

## Referências

- Transcrição: [09:09]-[09:13] Diego, Bruno, Larissa; [09:11] Larissa; [09:28] Bruno; [09:30] Bruno
- Código: `src/server.ts` — padrão de entry point que o `src/worker.ts` vai seguir
- Código: `src/config/database.ts` — criação do `PrismaClient` singleton que o worker replicará como instância própria
- Relacionado: [ADR-001](ADR-001-outbox-pattern-no-mysql.md)
