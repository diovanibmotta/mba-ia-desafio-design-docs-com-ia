# PRD — Sistema de Webhooks de Notificação de Pedidos

**Produto:** Order Management System (OMS)  
**Feature:** Webhooks de Notificação de Pedidos  
**Versão:** 1.0  
**Data:** 2026-07-21  
**Autores:** Marcos (PM), Larissa (Tech Lead)  
**Status:** Aprovado para desenvolvimento

---

## 1. Resumo e Contexto da Feature

O OMS serve clientes B2B que gerenciam grandes volumes de pedidos. Esses clientes precisam integrar seus sistemas internos (ERPs, plataformas logísticas, painéis de controle) com o status em tempo real dos pedidos na plataforma. Hoje, a única forma de obter essa informação é através de polling periódico no endpoint `GET /api/v1/orders`, o que é ineficiente, caro e causa latência desnecessária nas integrações.

Esta feature entrega um mecanismo de notificação proativa: quando o status de um pedido muda, o sistema dispara automaticamente uma chamada HTTP para os endpoints configurados pelo cliente — eliminando o polling e habilitando integrações verdadeiramente reativas.

---

## 2. Problema e Motivação

Três clientes B2B — **Atlas Comercial**, **MaxDistribuicao** e **Nova Cargo** — fizeram pedido formal de notificação em tempo real de mudanças de status de pedidos. O cenário atual:

- Clientes fazem polling em `GET /api/v1/orders` periodicamente para detectar mudanças.
- A integração é lenta (depende da frequência de polling) e gera carga desnecessária na API.
- A Atlas Comercial sinalizou formalmente que pode migrar para um concorrente caso a funcionalidade não seja entregue até o fim do trimestre.

O requisito de latência dos clientes: qualquer entrega **abaixo de 10 segundos** após a mudança de status é considerada "tempo real" para eles. O que importa é que não fiquem dependendo de atualização manual.

*Fonte: [09:00]-[09:02] Marcos*

---

## 3. Público-Alvo e Cenários de Uso

**Público-alvo:** Clientes B2B que integram seus sistemas internos com o OMS via API.

### Cenários de uso representativos

| Cenário | Cliente | Evento de interesse | Ação no sistema do cliente |
|---------|---------|--------------------|-----------------------------|
| Expedição automática | Atlas Comercial | `SHIPPED` | Aciona criação de etiqueta no sistema logístico |
| Sincronização financeira | MaxDistribuicao | `PAID` | Lança baixa financeira no ERP |
| Notificação ao consumidor final | Nova Cargo | `DELIVERED` | Dispara e-mail de confirmação para o destinatário |
| Estorno automático | Qualquer cliente | `CANCELLED` | Inicia fluxo de estorno no sistema de pagamentos |

---

## 4. Objetivos e Métricas de Sucesso

| Objetivo | Métrica | Meta |
|----------|---------|------|
| Eliminar dependência de polling | Redução no volume de chamadas a `GET /api/v1/orders` por clientes integrados | ≥ 80% de redução em 30 dias após adoção |
| Entrega confiável e rápida | Percentual de eventos entregues com sucesso na primeira tentativa, em até 10s | ≥ 95% |
| Zero perda de eventos | Eventos registrados vs. mudanças de status que deveriam gerar evento | 100% (garantia transacional) |
| Adoção pelos clientes B2B | Clientes que configuram pelo menos um webhook | 3 clientes (Atlas, MaxDistribuicao, Nova Cargo) até o fim do trimestre |

---

## 5. Escopo

### Em escopo

- Cadastro, edição, listagem e remoção de endpoints de webhook por cliente
- Filtragem de eventos por tipo de status (cliente escolhe quais status receber)
- Envio assíncrono de eventos via worker dedicado
- Autenticação de entrega via HMAC-SHA256 com secret por endpoint
- Rotação de secret com grace period de 24 horas
- Política de retry com backoff exponencial (até 5 tentativas)
- Dead Letter Queue (DLQ) para eventos que esgotam retries
- Endpoint admin para reprocessamento manual de DLQ
- Histórico de entregas por endpoint

### Fora de escopo

Os itens abaixo foram discutidos e **explicitamente excluídos** desta fase:

1. **Notificações por e-mail em caso de falhas recorrentes** — Marcos perguntou se o sistema poderia enviar e-mail ao cliente quando um webhook falha repetidamente. Adiado para próxima fase, após medir o impacto da feature em produção. *[09:37] Larissa*

2. **Rate limiting de envios por cliente** — Diego levantou a questão: se um cliente tem 50 pedidos mudando de status em um minuto, o sistema dispara 50 chamadas? Decisão: observar o comportamento em produção antes de implementar qualquer limitação de taxa. *[09:38]-[09:39] Diego, Larissa*

3. **Dashboard visual de webhooks** — Marcos perguntou sobre painel visual para o cliente acompanhar seus webhooks. Descartado: é projeto separado do time de frontend, fora do escopo desta feature de API. *[09:40] Larissa*

4. **Garantia de ordenação global entre pedidos** — Com um único worker, eventos do mesmo pedido chegam em ordem. Se no futuro múltiplos workers forem necessários, a garantia de ordenação exige redesign. Documentado como limitação conhecida, não como feature a implementar agora. *[09:13] Diego, Larissa*

5. **Arquivamento automático de eventos entregues** — Diego mencionou arquivamento após 30 dias, mas foi classificado como fora de escopo desta fase. *[09:08] Diego*

---

## 6. Requisitos Funcionais

| ID | Requisito | Fonte |
|----|-----------|-------|
| RF-01 | O sistema deve permitir o cadastro de endpoints de webhook associados a um cliente (`customerId`), informando URL e lista de status de pedido que deseja receber | [09:31] Marcos |
| RF-02 | O sistema deve gerar automaticamente uma `secret` única por endpoint no momento do cadastro, retornando-a apenas uma vez | [09:31] Marcos |
| RF-03 | O cliente deve poder atualizar a URL, a lista de eventos e o estado ativo/inativo de um webhook existente | [09:33] Bruno |
| RF-04 | O cliente deve poder remover um endpoint de webhook | [09:33] Bruno |
| RF-05 | O cliente deve poder listar todos os endpoints de webhook cadastrados para um `customerId` | [09:33] Bruno |
| RF-06 | Quando o status de um pedido muda, o sistema deve inserir um evento na fila (`webhook_outbox`) para cada endpoint ativo do cliente que tenha aquele status na sua lista de eventos | [09:33]-[09:34] Marcos, Bruno |
| RF-07 | A inserção do evento na fila deve ocorrer na mesma transação SQL que atualiza o status do pedido, garantindo atomicidade | [09:06] Diego, [09:41] Bruno |
| RF-08 | O worker deve disparar chamadas HTTP POST para os endpoints configurados, com timeout de 10 segundos por tentativa | [09:09] Diego, [09:42] Diego |
| RF-09 | Em caso de falha de entrega, o sistema deve tentar novamente seguindo o backoff: 1min, 5min, 30min, 2h, 12h (máximo 5 tentativas) | [09:17] Diego |
| RF-10 | Após 5 tentativas sem sucesso, o evento deve ser movido para a Dead Letter Queue (`webhook_dead_letter`) | [09:18] Diego |
| RF-11 | Administradores devem poder reprocessar manualmente entradas da DLQ via endpoint `POST /api/v1/admin/webhooks/dead-letter/:id/replay` | [09:18] Diego, [09:35]-[09:36] Larissa, Sofia |
| RF-12 | O cliente deve poder consultar o histórico das últimas entregas de um webhook (`GET /api/v1/webhooks/:id/deliveries`) | [09:34]-[09:35] Marcos |
| RF-13 | O cliente deve poder rotacionar a secret de um endpoint, com a secret anterior permanecendo válida por 24 horas | [09:21] Sofia |

---

## 7. Requisitos Não Funcionais

| ID | Requisito | Fonte |
|----|-----------|-------|
| RNF-01 | Latência máxima de entrega: até 10 segundos após a mudança de status | [09:02] Marcos |
| RNF-02 | URLs de webhook devem ser obrigatoriamente HTTPS; URLs `http://` devem ser rejeitadas com erro de validação | [09:23] Sofia |
| RNF-03 | Cada entrega deve incluir assinatura `HMAC-SHA256` no header `X-Signature` | [09:20] Sofia |
| RNF-04 | Cada tentativa de entrega deve ter timeout de 10 segundos | [09:42] Diego |
| RNF-05 | O payload máximo de cada evento é 64KB; eventos maiores devem ser rejeitados | [09:24] Diego, Larissa |
| RNF-06 | A garantia de entrega é at-least-once; clientes devem deduplicar usando `X-Event-Id` | [09:24]-[09:25] Diego |
| RNF-07 | O payload armazena um snapshot do estado do pedido no momento da mudança de status, não no momento do envio | [09:52] Larissa, Diego |
| RNF-08 | O worker roda como processo separado da API, com ciclo de vida independente | [09:11] Diego |

---

## 8. Decisões e Trade-offs Principais

| Decisão | Trade-off | Referência |
|---------|-----------|------------|
| Outbox no MySQL (não Redis/broker externo) | Menos infraestrutura, mesma consistência; sem sub-segundo de latência | [ADR-001](adrs/ADR-001-outbox-pattern-no-mysql.md) |
| Worker com polling de 2s | Simplicidade vs. reatividade imediata | [ADR-005](adrs/ADR-005-worker-processo-separado-com-polling.md) |
| At-least-once com X-Event-Id | Implementação simples vs. responsabilidade de dedup no cliente | [ADR-004](adrs/ADR-004-garantia-at-least-once-com-event-id.md) |
| 5 retries com backoff fixo | Cobre até 15h de downtime vs. eventos pendurados indefinitivamente | [ADR-002](adrs/ADR-002-retry-com-backoff-e-dlq.md) |
| Secret por endpoint (não global) | Blast radius limitado vs. gerenciamento por endpoint | [ADR-003](adrs/ADR-003-hmac-sha256-com-secret-por-endpoint.md) |

---

## 9. Dependências

| Dependência | Tipo | Descrição |
|------------|------|-----------|
| Banco de dados MySQL existente | Técnica | As novas tabelas (`webhook_outbox`, `webhook_endpoints`, etc.) são criadas no mesmo banco |
| Método `changeStatus` do `OrderService` | Técnica | Ponto de integração onde a inserção na outbox é adicionada |
| Middleware de autenticação JWT existente | Técnica | `authenticate` e `requireRole('ADMIN')` usados sem modificação |
| Revisão de segurança da Sofia | Processo | Sofia revisará o código de HMAC e geração de secret antes do deploy |
| Portal do desenvolvedor (Marcos) | Produto | Marcos documentará o contrato de at-least-once e a verificação de HMAC para os clientes |

---

## 10. Riscos e Mitigação

| Risco | Probabilidade | Impacto | Mitigação |
|-------|--------------|---------|-----------|
| Worker parado interrompe todas as entregas | Médio | Alto | Supervisor de processo (pm2/systemd), alerta de monitoramento em ausência de polling, SLA de resposta de incidente |
| Endpoint do cliente consistentemente lento, degradando throughput do worker | Baixo | Médio | Timeout de 10s por tentativa limita o impacto; observar se rate limiting se torna necessário |
| Vazamento de secret por cliente (ex: em logs de aplicação) | Baixo | Alto | Secrets por endpoint limitam blast radius; suporte a rotação disponível; Sofia revisará geração de secret |
| Volume de pedidos maior que o esperado sobrecarregando a outbox | Baixo (curto prazo) | Médio | Índices na tabela garantem queries eficientes; monitorar crescimento da tabela |

---

## 11. Critérios de Aceitação

| ID | Critério |
|----|---------|
| CA-01 | Cliente consegue cadastrar, editar, listar e remover endpoints de webhook via API autenticada |
| CA-02 | Mudança de status de pedido resulta em evento na outbox dentro da mesma transação SQL |
| CA-03 | Worker entrega o evento ao endpoint do cliente em até 10 segundos após a mudança de status |
| CA-04 | Falhas de entrega resultam em retry conforme o backoff definido (1m/5m/30m/2h/12h) |
| CA-05 | Após 5 falhas, o evento aparece na DLQ e não está mais pendente na outbox |
| CA-06 | Admin com role `ADMIN` consegue reprocessar entrada da DLQ; roles inferiores recebem `403` |
| CA-07 | A assinatura `X-Signature` no header de cada entrega pode ser verificada com `HMAC-SHA256(secret, body)` |
| CA-08 | URL com `http://` é rejeitada com `400 WEBHOOK_INVALID_URL` |
| CA-09 | Rotação de secret mantém a secret anterior válida por exatamente 24h |
| CA-10 | Eventos de status não listados na configuração do webhook não geram entradas na outbox |
| CA-11 | Histórico de entregas é acessível via `GET /api/v1/webhooks/:id/deliveries` |

---

## 12. Estratégia de Testes e Validação

### Testes de integração (suite existente em `tests/`)

- Fluxo completo: criar webhook → mudar status do pedido → verificar evento na outbox
- Atomicidade: simular falha após mudança de status, verificar que outbox não tem evento órfão
- Filtragem: status não listado não deve gerar entrada na outbox

### Testes do worker

- Sucesso de entrega: mock de endpoint HTTP, verificar `status = DELIVERED` e log
- Retry: mock retornando 503, verificar incremento de `attempts` e `nextRetryAt`
- DLQ: simular 5 falhas, verificar que evento vai para `webhook_dead_letters`
- Replay: verificar que replay cria novo registro na outbox com `attempts = 0`

### Validação de segurança (Sofia)

- Revisão do código de geração de secret (entropia, formato)
- Revisão do código de HMAC-SHA256 (algoritmo, encoding)
- Teste de rotação com grace period de 24h
- Verificar que secret não aparece em logs (redact do Pino)

Sofia reservou 2 dias úteis para revisão antes do deploy em produção. *[09:46] Sofia*

### Prazo estimado

3 sprints incluindo a revisão de segurança da Sofia. *[09:46] Larissa*
