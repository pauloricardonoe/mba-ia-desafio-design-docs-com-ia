# ADR-005 — Garantia at-least-once com `X-Event-Id` para deduplicação no cliente

## Status

Aceito

## Data

2026-10-17 (decisão tomada na reunião técnica de quinta-feira, 09:00)

## Decisores

- Diego (Engenheiro Sênior, Plataforma) — propôs
- Larissa (Tech Lead) — fechou a decisão
- Sofia (Engenheira de Segurança) — levantou o custo da responsabilidade transferida
- Marcos (Product Manager) — assumiu a comunicação ao cliente

## Contexto

Com retry automático ([ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md)) e reprocessamento manual de DLQ, a entrega **vai** duplicar em algum momento: um cliente pode receber a resposta HTTP depois do timeout já ter disparado o retry, e um replay manual recoloca deliberadamente um evento na outbox.

É preciso decidir qual garantia de entrega a plataforma oferece e como o cliente lida com duplicatas.

As opções canônicas são *at-most-once*, *at-least-once* e *exactly-once*. Sofia levantou imediatamente o ponto sensível da opção escolhida: "isso joga responsabilidade pro cliente" ([09:25] Sofia).

## Decisão

**Garantia at-least-once, com um `event_id` (UUID) propagado no header `X-Event-Id` para que o cliente deduplique do lado dele.**

Propriedades acordadas:

- O `event_id` é um **UUID gerado no momento em que o evento entra na `webhook_outbox`** e é único por evento ([09:25] Diego).
- Ele é enviado no header **`X-Event-Id`** ([09:25] Diego).
- A plataforma **não** garante entrega única: o cliente precisa estar preparado para receber o mesmo evento mais de uma vez e deduplicar pelo `event_id` ([09:24] Diego, [09:25] Diego).
- O comportamento é aceito como padrão de mercado: "Stripe faz assim, GitHub faz assim. Garantir exactly-once exigiria coordenação dos dois lados e fica muito mais complexo. At-least-once com event_id resolve 99% dos casos" ([09:25] Diego).
- A responsabilidade transferida ao cliente será **comunicada explicitamente**: Marcos se comprometeu a documentar isso bem destacado no portal do desenvolvedor ([09:26] Marcos).

## Alternativas consideradas

### 1. Exactly-once

Garantir que cada evento seja entregue exatamente uma vez ao cliente.

**Trade-off que motivou o descarte:** exigiria coordenação dos dois lados — a plataforma precisaria de confirmação explícita do cliente de que ele processou o evento, e o cliente precisaria participar de um protocolo de deduplicação ou de ack. Isso é muito mais complexo do que o problema justifica e ainda assim não eliminaria todos os casos de borda. Diego resumiu: "at-least-once com event_id resolve 99% dos casos" ([09:25] Diego).

### 2. At-most-once (sem retry)

Enviar uma única vez; se falhar, o evento é perdido.

**Trade-off que motivou o descarte:** contradiz diretamente o requisito de negócio. Os clientes precisam ser notificados de mudanças de status de pedidos — perder um evento significa que o cliente nunca saberá que o status mudou. Toda a política de retry ([ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md)) existe justamente para não perder eventos, o que é incompatível com at-most-once.

## Consequências

### Positivas

- **Simplicidade do lado da plataforma:** não é necessário manter estado de confirmação por cliente nem um protocolo de ack bilateral.
- **Alinhamento com o mercado:** clientes B2B que já integram com Stripe e GitHub reconhecem o padrão de `X-Event-Id` e deduplicação no receptor ([09:25] Diego).
- **Nenhum evento é silenciosamente perdido:** combinada com o retry e a DLQ, a garantia at-least-once assegura que todo evento ou é entregue ou termina na DLQ.
- **Compatível com replay manual:** recolocar um evento da DLQ na outbox é seguro, porque o cliente deduplica pelo `event_id`.

### Negativas

- **Transfere responsabilidade ao cliente:** o cliente precisa implementar deduplicação por `event_id`. Sofia registrou esse custo explicitamente ([09:25] Sofia).
- **Risco de processamento duplicado no cliente:** se o cliente não implementar a deduplicação corretamente, pode processar o mesmo evento duas vezes — consequência do lado dele, mas com origem na nossa garantia de entrega.
- **Exige comunicação/documentação ativa:** sem um aviso destacado no portal, o cliente pode assumir entrega única e se comportar incorretamente ([09:26] Marcos).
- **`event_id` precisa ser estável entre tentativas:** a geração do UUID na inserção (e não a cada tentativa de envio) é uma invariante obrigatória; gerá-lo no envio quebraria a deduplicação.

### Trade-off explícito

Aceita-se **transferir a responsabilidade de deduplicação ao cliente** e **aceitar duplicatas na entrega** em troca de **uma implementação muito mais simples do lado da plataforma** e **alinhamento com o padrão de mercado que os clientes já conhecem**. O ponto sensível (responsabilidade do cliente) foi levantado em reunião e mitigado por comunicação explícita no portal do desenvolvedor ([09:25] Sofia, [09:26] Marcos).

## Referências

- Transcrição: [09:24] Diego, [09:25] Bruno, [09:25] Diego, [09:25] Sofia, [09:26] Marcos, [09:26] Larissa
- Código: `prisma/schema.prisma` (padrão de chaves UUID), `src/modules/orders/order.service.ts` (origem do evento), `src/shared/http/response.ts` (padrão de resposta da API)
