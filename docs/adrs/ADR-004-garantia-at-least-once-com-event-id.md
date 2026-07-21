# ADR-004 — Garantia At-Least-Once com X-Event-Id para Deduplicação

**Status:** Aceito  
**Data:** 2026-07-21  
**Participantes:** Diego (Engenheiro Sênior), Sofia (Engenheira de Segurança), Bruno (Engenheiro Pleno), Marcos (PM)

---

## Contexto

O worker pode entregar o mesmo evento mais de uma vez em cenários de falha: o cliente recebeu e processou a requisição, mas o timeout expirou antes da resposta chegar ao worker, que então tenta novamente. Ou o worker crasha após enviar mas antes de marcar o evento como entregue.

O sistema precisa definir claramente qual garantia de entrega oferece e como os clientes devem lidar com duplicatas.

[09:24] Diego: *"A gente vai garantir at-least-once. Pode acontecer de o cliente receber o mesmo evento duas vezes. Ele tem que estar preparado."*

## Decisão

Adotar garantia **at-least-once**: o sistema garante que cada evento será entregue pelo menos uma vez, mas não exatamente uma vez.

Para permitir deduplicação do lado do cliente:
- Cada evento recebe um **UUID gerado no momento da inserção na outbox** (`webhook_outbox.id`).
- Esse UUID é enviado no header `X-Event-Id` em toda tentativa de entrega do mesmo evento.
- O cliente usa o `X-Event-Id` para detectar e descartar duplicatas.

## Alternativas Consideradas

### 1. Garantia Exactly-Once

Implementar coordenação entre a plataforma e o cliente para garantir que cada evento seja processado exatamente uma vez — por exemplo, via protocolo de confirmação em duas fases.

**Por que foi descartado:** Exigiria um protocolo de handshake complexo entre os dois sistemas, aumentando drasticamente a complexidade de implementação em ambos os lados. Os precedentes da indústria (Stripe, GitHub, Shopify) adotam at-least-once com deduplicação por event_id. [09:25] Diego: *"Garantir exactly-once exigiria coordenação dos dois lados e fica muito mais complexo. At-least-once com event_id resolve 99% dos casos."*

### 2. At-Most-Once (fire-and-forget)

Tentar entregar uma única vez e não fazer retry em caso de falha.

**Por que não foi considerado:** Perderia eventos em falhas transitórias de rede, inaceitável para integração B2B que precisa de confiabilidade.

## Consequências

**Positivas:**
- Implementação significativamente mais simples que exactly-once.
- Padrão bem estabelecido e documentado no mercado.
- UUID fixo por evento permite deduplicação eficiente (busca por chave única no banco do cliente).
- Compatível com o padrão de retry definido no ADR-002.

**Negativas:**
- Empurra a responsabilidade de deduplicação para o cliente — cliente que não implementar deduplicação pode processar eventos duplicados.
- Requer documentação clara no portal do desenvolvedor ([09:26] Marcos: *"Eu posso documentar isso bem destacado no portal de desenvolvedor pros clientes."*).
- Em cenários de alta indisponibilidade do cliente, o mesmo evento pode chegar várias vezes ao longo de 15 horas.

## Referências

- Transcrição: [09:24]-[09:26] Diego, Sofia, Bruno, Marcos
- Relacionado: [ADR-002](ADR-002-retry-com-backoff-e-dlq.md)
