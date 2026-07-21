# ADR-003 — Autenticação HMAC-SHA256 com Secret por Endpoint

**Status:** Aceito  
**Data:** 2026-07-21  
**Participantes:** Sofia (Engenheira de Segurança), Larissa (Tech Lead), Bruno (Engenheiro Pleno), Diego (Engenheiro Sênior)

---

## Contexto

O worker envia payloads com dados de pedidos para endpoints externos dos clientes. O cliente precisa conseguir verificar duas coisas: (1) que a requisição veio de fato da nossa plataforma, e (2) que o payload não foi adulterado no caminho. Sem algum mecanismo de autenticação, qualquer agente malicioso poderia forjar chamadas para o endpoint do cliente.

Além disso, a solução precisa ser implementável pelos clientes com bibliotecas padrão, sem exigir infraestrutura especial.

[09:19] Sofia: *"O cliente tem que conseguir validar que a requisição veio realmente da gente, e que ninguém adulterou o payload no meio."*

## Decisão

Usar **HMAC-SHA256** para assinar o corpo da requisição. O sistema:

1. Mantém uma **secret única por endpoint de webhook** (não uma secret global da plataforma).
2. Gera a secret server-side no momento de criação do webhook e retorna ao cliente **apenas uma vez**.
3. Computa `HMAC-SHA256(secret, requestBody)` antes de cada envio.
4. Envia a assinatura no header `X-Signature: sha256=<hex>`.
5. Suporta **rotação de secret** com grace period de 24 horas: ao rotacionar, a secret anterior permanece válida por 24h para o cliente migrar seus sistemas sem downtime.

O cliente verifica a assinatura do lado dele usando a secret compartilhada.

TLS (HTTPS) é obrigatório em todos os endpoints (validação na criação do webhook, [09:23] Sofia). A secret é gerada e gerenciada pela plataforma — o cliente não a escolhe.

## Alternativas Consideradas

### 1. Secret global única para toda a plataforma

Uma única secret compartilhada entre todos os endpoints de webhook de todos os clientes.

**Por que foi descartado:** Se um cliente vazar a secret (em log de aplicação, por exemplo — caso real citado por Diego em [09:22]), todos os outros clientes seriam comprometidos. [09:21] Sofia: *"Não é uma secret global da nossa plataforma. Senão se vaza uma, vaza tudo."*

### 2. mTLS (mutual TLS / certificados de cliente)

Autenticação mútua via certificados TLS.

**Por que foi descartado:** Não discutido formalmente, mas HMAC-SHA256 é o padrão de mercado adotado por Stripe, GitHub e outros serviços de webhook B2B. Certificados de cliente adicionam complexidade operacional considerável para os clientes e exigem infraestrutura de PKI. [09:20] Sofia: *"HMAC-SHA256 é o padrão de mercado, todo cliente sério tem biblioteca pra isso."*

## Consequências

**Positivas:**
- Padrão de mercado: todos os clientes B2B sérios têm biblioteca disponível para verificar HMAC-SHA256.
- Isolamento por endpoint: vazamento de secret de um cliente não afeta os outros.
- Rotação com grace period permite migração sem downtime.
- Proteção contra replay attacks parcial via `X-Timestamp` (cliente pode rejeitar timestamps muito antigos).

**Negativas:**
- A secret precisa ser armazenada no banco de forma segura (não em plaintext, ou ao menos bem protegida).
- Clientes precisam implementar a verificação; falha na implementação do lado deles não é detectável.
- Processo de rotação adiciona complexidade: durante o grace period, o worker precisa reconhecer que a assinatura com a secret anterior ainda é válida no lado do cliente.
- Sofia exige revisão de código do mecanismo de HMAC e geração de secret antes do deploy ([09:46]).

## Referências

- Transcrição: [09:19]-[09:22] Sofia, Bruno, Diego; [09:23] Sofia; [09:46] Sofia
