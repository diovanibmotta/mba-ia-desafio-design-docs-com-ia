# Da Reunião ao Documento: Design Docs Gerados por IA

## Sobre o Desafio

Este repositório contém o entregável do desafio do MBA de IA: transformar a transcrição de uma reunião técnica real em um pacote completo de documentação de design de software, usando IA como ferramenta principal de produção.

O cenário: um OMS (Order Management System) em Node.js/TypeScript/Express/Prisma/MySQL precisa de uma nova feature — um Sistema de Webhooks de Notificação de Pedidos. A decisão técnica já foi tomada em reunião entre tech lead, PM, dois engenheiros e a equipe de segurança. Minha tarefa foi transformar a transcrição dessa reunião (~55 minutos, 5 participantes) e o código-fonte existente em documentação técnica acionável — sem inventar um único requisito, decisão ou restrição que não tenha origem identificável na transcrição ou no código.

O que distingue este desafio de um simples "peça pra IA escrever a documentação": cada item documentado está rastreado ao timestamp exato da reunião ou ao arquivo de código de origem, no TRACKER. Se não tem origem, não entra.

---

## Ferramentas de IA Utilizadas

| Ferramenta | Papel no processo |
|-----------|-------------------|
| **Claude Code (claude-sonnet-4-6, CLI)** | Ferramenta principal. Exploração do código-fonte, leitura completa da transcrição, planejamento estruturado dos documentos, geração e refinamento iterativo de todo o conteúdo |

O Claude Code foi usado em modo interativo via CLI, com acesso direto ao sistema de arquivos do repositório. A IA leu o código-fonte do OMS (order.service.ts, schema.prisma, middleware de erros, auth middleware, etc.) antes de gerar qualquer conteúdo — isso garantiu que referências a arquivos e funções fossem reais, não inventadas.

---

## Workflow Adotado

### Ordem de produção e motivação

**1. Exploração paralela (antes de escrever qualquer doc)**

Dois agentes exploradores rodaram em paralelo: um leu a transcrição completa linha a linha; outro explorou o código-fonte em profundidade (order.service.ts, schema.prisma, todas as middlewares, padrões de erro, estrutura de módulos). Esse passo foi essencial — documentar sem ler o código garante referências incorretas a arquivos inexistentes.

**2. ADRs primeiro**

As 6 decisões discutidas na reunião viraram 6 ADRs antes de qualquer outro documento. Motivo: os ADRs são o esqueleto. RFC e FDD referenciam ADRs, não o contrário. Escrever o RFC sem os ADRs prontos levaria a duplicação ou inconsistência.

**3. RFC**

Com as decisões formalizadas, o RFC consolidou a proposta arquitetural em 2-4 páginas. As alternativas descartadas e as questões em aberto da reunião têm lugar natural aqui. Os ADRs já escritos são referenciados por link — sem repetir o conteúdo.

**4. FDD**

Com RFC e ADRs em mãos, o FDD desceu ao detalhe de implementação. A seção obrigatória "Integração com o sistema existente" foi escrita com os arquivos do código na memória do contexto: nenhuma referência foi inventada.

**5. PRD**

Produzido por último entre os documentos de conteúdo. Com RFC + FDD + ADRs prontos, o PRD virou essencialmente uma consolidação do ponto de vista de produto/negócio, sem duplicar detalhes técnicos.

**6. TRACKER**

Construído após todos os outros documentos estarem prontos. Varreu cada item dos docs e mapeou à origem — timestamp da transcrição ou arquivo de código. Qualquer linha sem origem identificável foi removida dos docs.

**7. README**

Último, quando o processo já estava completo e documentável com clareza.

---

## Prompts Customizados

### Prompt 1 — Exploração profunda da transcrição e código

Usado para orientar o agente explorador no primeiro passo do processo:

```
Explore C:\workspaces\mba\mba-ia-desafio-design-docs-com-ia codebase. Preciso:

1. Full directory tree de src/ com TODOS os arquivos
2. Schema Prisma completo
3. Módulo orders: TODOS os arquivos (service, controller, repository, DTOs, routes) — leia cada um completamente
4. Auth module: TODOS os arquivos
5. Padrões de tratamento de erros — leia as classes de erro, exception filters
6. Outros módulos (users, clients, products) — padrões principais
7. package.json dependencies
8. docker-compose.yml
9. Estrutura e padrões de testes
10. Diretório docs/ — o que existe atualmente

Reporte caminhos de arquivo, código completo do order service e transições de status, códigos de erro, padrões de middleware. Seja extremamente minucioso — isso será usado para escrever design docs que referenciam caminhos de arquivo reais.
```

### Prompt 2 — Produção do FDD com integração rastreável

Usado para orientar a geração do FDD com referências reais ao código:

```
Produza o FDD para o Sistema de Webhooks com as seguintes restrições rígidas:

SEÇÃO OBRIGATÓRIA "Integração com o sistema existente" deve:
- Nomear pelo menos 7 caminhos de arquivo REAIS do código (verificados previamente)
- Para cada arquivo: descrever EXATAMENTE como o módulo de webhooks vai se integrar
- Especificamente: como publishWebhookEvent() se encaixa no $transaction de order.service.ts
- Especificamente: como os novos erros estendem app-error.ts e http-errors.ts
- NÃO mencionar arquivos que não existem no repositório

RESTRIÇÕES DE CONTEÚDO:
- Não incluir: email notifications, rate limiting, dashboard visual, multi-worker scaling
- Payload snapshot na inserção, não no envio [09:52]
- customer_id vem do body/path, não do JWT [09:32]
- Filtro de eventos na inserção da outbox, não no envio [09:34]

MATRIZ DE ERROS: prefixo WEBHOOK_ em todos os códigos, seguindo padrão de INSUFFICIENT_STOCK, INVALID_STATUS_TRANSITION já no projeto.
```

### Prompt 3 — Validação de rastreabilidade do TRACKER

Usado para garantir integridade do tracker antes da entrega:

```
Para cada linha do TRACKER, verifique:
1. Se fonte = TRANSCRICAO: o timestamp [hh:mm] e o nome do falante estão corretos?
   Compare com TRANSCRICAO.md — não invente timestamps.
2. Se fonte = CODIGO: o caminho de arquivo existe no repositório?
   Liste os arquivos que NÃO existem e remova as linhas correspondentes.
3. Há itens dos documentos (PRD, RFC, FDD, ADRs) que não têm linha no TRACKER?
   Se sim, adicione ou justifique a exclusão.

Cobertura mínima exigida:
- 80% dos itens identificáveis têm linha no TRACKER
- 70% das linhas têm fonte TRANSCRICAO com timestamp válido
- 5+ linhas têm fonte CODIGO com caminho real
```

---

## Iterações e Ajustes

### Iteração 1 — Remoção de itens fora de escopo do PRD

Na primeira geração do PRD, a seção de Requisitos Funcionais incluía "Envio de e-mail para clientes quando webhooks falham consecutivamente" como RF-14. A IA não percebeu que isso foi explicitamente descartado pela Larissa em [09:37]: *"Email está fora de escopo dessa fase."* Corrigi removendo o item e adicionando-o explicitamente na seção "Fora de escopo".

Lição: prompts vagos como "gere o PRD baseado na transcrição" incluem tudo que foi *mencionado*, não só o que foi *decidido*. O prompt precisou especificar: "itens explicitamente descartados ou adiados devem aparecer em 'Fora de escopo', não em requisitos."

### Iteração 2 — RFC duplicando detalhe do FDD

Na primeira versão do RFC, a seção "Proposta Técnica" incluía o schema Prisma completo dos 4 novos modelos e a assinatura completa da função `publishWebhookEvent`. Isso é nível FDD, não RFC.

Corrigi limitando o RFC à visão arquitetural (o que e por quê) e movendo toda a especificação de implementação para o FDD. O RFC menciona o padrão Outbox, o worker, o HMAC — mas não descreve campos de tabela nem assinaturas de função.

### Iteração 3 — TRACKER com timestamps incorretos

Na primeira geração automática do TRACKER, algumas linhas tinham timestamps como `[09:15]` atribuídos ao falante errado (ex: Bruno em vez de Diego). A IA estava extrapolando, não citando com precisão.

Corrigi relendo a transcrição e verificando cada timestamp/falante manualmente. A regra final: se não consigo confirmar o timestamp exato, uso o intervalo (`[09:15]-[09:18]`) em vez de inventar precisão.

### Iteração 4 — Referências a arquivos inexistentes no FDD

A primeira versão do FDD mencionava `src/modules/orders/order.entity.ts` na seção de integração — arquivo que não existe no projeto (o projeto usa Prisma sem camadas de entidade separadas). Detectado via busca no código antes de finalizar.

A regra aplicada: antes de mencionar qualquer caminho de arquivo em qualquer documento, verificar que o arquivo existe no repositório. Todos os caminhos no FDD foram verificados.

---

## Como Navegar a Entrega

### Estrutura dos arquivos

```
docs/
  PRD.md          — Contexto de produto, requisitos funcionais e não funcionais
  RFC.md          — Proposta técnica arquitetural, alternativas, questões abertas
  FDD.md          — Especificação de implementação detalhada
  TRACKER.md      — Rastreabilidade de todos os itens aos documentos fonte
  adrs/
    ADR-001-outbox-pattern-no-mysql.md
    ADR-002-retry-com-backoff-e-dlq.md
    ADR-003-hmac-sha256-com-secret-por-endpoint.md
    ADR-004-garantia-at-least-once-com-event-id.md
    ADR-005-worker-processo-separado-com-polling.md
    ADR-006-reuso-padroes-existentes-do-projeto.md
TRANSCRICAO.md    — Fonte primária (não alterado)
src/              — Código-fonte do OMS (não alterado)
```

### Ordem sugerida de leitura

1. **TRANSCRICAO.md** — entenda a reunião original (opcional, mas recomendado)
2. **docs/adrs/** — leia os 6 ADRs em ordem para entender cada decisão isolada
3. **docs/RFC.md** — veja como as decisões se consolidam numa proposta arquitetural
4. **docs/FDD.md** — mergulhe no detalhe de implementação
5. **docs/PRD.md** — visão de produto e negócio
6. **docs/TRACKER.md** — verifique a rastreabilidade de qualquer item que gerar dúvida

### Para engenheiros iniciando a implementação

O ponto de partida está descrito em `docs/FDD.md`, seção "Integração com o sistema existente":
- A mudança no `src/modules/orders/order.service.ts` (seção 10.1)
- O schema das 4 novas tabelas Prisma (seção 4)
- Os 7 endpoints com payloads de exemplo (seção 6)
