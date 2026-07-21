# ADR-002 — Política de Retry com Backoff Exponencial e Dead Letter Queue

**Status:** Aceito  
**Data:** 2026-07-21  
**Participantes:** Larissa (Tech Lead), Bruno (Engenheiro Pleno), Diego (Engenheiro Sênior), Marcos (PM)

---

## Contexto

O worker que despacha webhooks vai encontrar falhas: cliente fora do ar, timeout de rede, erros HTTP 5xx. O sistema precisa de uma política clara para lidar com essas falhas — quantas vezes tentar novamente, com qual intervalo, e o que fazer quando todas as tentativas se esgotam.

O requisito de negócio é cobrir janelas de indisponibilidade razoáveis dos clientes B2B, incluindo manutenções planejadas. [09:16] Diego mencionou um caso real: *"cliente nosso com indisponibilidade de duas horas em manutenção planejada."*

## Decisão

**5 tentativas** com backoff em escala fixa:

| Tentativa | Intervalo após falha |
|-----------|---------------------|
| 1ª retry  | 1 minuto            |
| 2ª retry  | 5 minutos           |
| 3ª retry  | 30 minutos          |
| 4ª retry  | 2 horas             |
| 5ª retry  | 12 horas            |

Janela total: ~15 horas entre a primeira falha e a última tentativa.

Após esgotar as 5 tentativas, o evento é movido para a tabela `webhook_dead_letter` (DLQ), onde fica retido para análise e reprocessamento manual. O registro na `webhook_outbox` é removido ou marcado como `FAILED`.

Reprocessamento manual via endpoint admin: `POST /api/v1/admin/webhooks/dead-letter/:id/replay` (exige role `ADMIN`), que reinsere o evento na outbox com tentativas zeradas.

## Alternativas Consideradas

### 1. Apenas 3 tentativas

Proposta inicial de Bruno ([09:16]): ser mais agressivo e encerrar mais cedo.

**Por que foi descartado:** 3 tentativas com backoff curto seria insuficiente para cobrir uma janela de 2 horas de manutenção planejada. Diego: *"3 é pouco. Se o cliente teve indisponibilidade de manhã, a gente retentaria três vezes em 30 minutos e mataria."*

### 2. Retry indefinido com backoff

Continuar tentando para sempre, sem limite de tentativas.

**Por que foi descartado:** Eventos ficariam pendurados indefinitivamente se o cliente sumir permanentemente, poluindo a outbox e dificultando operação. [09:15] Diego: *"isso traz o problema de evento ficar pendurado pra sempre se o cliente sumiu."*

### 3. Marcar como "failed" na própria outbox em vez de tabela DLQ separada

Adicionar um status `FAILED` e manter o registro na `webhook_outbox`.

**Por que foi descartado:** Tabela separada mantém a outbox principal limpa e focada em eventos a processar. A DLQ serve como evidência de auditoria e facilita reprocessamento. [09:18] Diego: *"Mais limpa a leitura da outbox principal, e fica como evidence pra debug e reprocessamento."*

## Consequências

**Positivas:**
- Cobre até ~15 horas de indisponibilidade do cliente, incluindo manutenções planejadas.
- DLQ como evidência de falhas facilita debugging e suporte.
- Endpoint de replay permite recuperação manual sem acesso direto ao banco.
- Outbox principal permanece focada em eventos pendentes de processamento.

**Negativas:**
- Não há notificação automática ao cliente quando eventos entram na DLQ (email adiado para próxima fase, [09:37] Larissa).
- Reprocessamento é manual, exige intervenção de alguém com role ADMIN.
- Cronograma fixo (não verdadeiramente exponencial) é menos adaptativo a diferentes padrões de falha.

## Referências

- Transcrição: [09:15]-[09:18] Diego, Bruno, Larissa; [09:35]-[09:36] Diego, Sofia, Larissa
- Relacionado: [ADR-005](ADR-005-worker-processo-separado-com-polling.md)
