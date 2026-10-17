# ADR-007 — Snapshot do payload renderizado no momento da inserção na outbox

## Status

Aceito

## Data

2026-10-17 (decisão tomada na reunião técnica de quinta-feira, 09:00, no encerramento da call)

## Decisores

- Larissa (Tech Lead) — decidiu
- Bruno (Engenheiro Pleno, Pedidos) — levantou a dúvida
- Diego (Engenheiro Sênior, Plataforma) — concordou

## Contexto

Levantada por Bruno já no fim da reunião, quando a call ia fechar ([09:51] Bruno): a linha da `webhook_outbox` deve **guardar o payload já renderizado** (o JSON completo que será enviado ao cliente) ou **guardar apenas o `order_id`** e renderizar o payload na hora do envio?

A diferença importa porque o pedido é um objeto mutável. Entre o momento em que o status muda e o momento em que o worker efetivamente envia a notificação — que pode ser de 2 segundos no caso feliz ou de horas em caso de retry ([ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md)) — o pedido pode sofrer novas alterações.

Se o payload for renderizado no envio, o cliente pode receber uma notificação do evento "PAID → PROCESSING" contendo dados que já refletem um estado posterior do pedido. Larissa descreveu esse cenário como "caso esquisito" ([09:52] Larissa).

## Decisão

**O evento guarda o payload já renderizado, como snapshot, no momento da inserção na outbox.**

Propriedades acordadas:

- O payload é renderizado **na inserção**, não no envio ([09:52] Larissa).
- Justificativa explícita: "se o pedido mudar depois, o evento ainda reflete o estado de quando o status mudou" ([09:52] Larissa).
- Diego concordou com o termo: "concordo, snapshot na inserção" ([09:52] Diego), e Bruno fechou: "beleza, snapshot. Decidido" ([09:52] Bruno).

## Alternativas consideradas

### 1. Guardar apenas o `order_id` e renderizar no envio

Manter na outbox somente a referência ao pedido e montar o payload no momento da chamada HTTP.

**Trade-off que motivou o descarte:** o payload enviado refletiria o estado **atual** do pedido, não o estado no momento da mudança de status. Um evento de "PAID → PROCESSING" enfileirado e entregue horas depois (após retries) poderia chegar ao cliente com dados de um pedido já `SHIPPED`, criando uma notificação internamente inconsistente e de difícil explicação para o cliente ([09:52] Larissa). A alternativa também exigiria uma leitura adicional do pedido no caminho crítico do worker, a cada tentativa.

### 2. Guardar o `order_id` e um subconjunto mínimo de campos

Meio-termo: persistir só os campos que não mudam, renderizando o resto no envio.

**Trade-off que motivou o descarte:** não foi discutida explicitamente na reunião. Seria descartada pelo mesmo motivo da alternativa 1 — qualquer campo lido no envio carrega o risco de divergir do estado no momento da mudança de status —, além de adicionar complexidade ao definir o que é "imutável" no pedido. Registrada aqui como alternativa plausível, mas não debatida na call.

## Consequências

### Positivas

- **Consistência temporal do evento:** o payload entregue corresponde exatamente ao estado do pedido no instante da mudança de status, independentemente de quanto tempo a entrega demore ([09:52] Larissa).
- **Idempotência e reprodutibilidade:** reprocessar um evento da DLQ ([ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md)) envia exatamente o mesmo payload, o que torna o replay previsível.
- **Caminho de envio mais leve:** o worker não precisa reler o pedido nem seus relacionamentos para montar a notificação.
- **Aderente ao payload enxuto:** o snapshot é renderizado a partir dos campos já disponíveis na transação do `changeStatus`, incluindo os dados básicos da order ([09:43] Diego).

### Negativas

- **Payload duplicado no banco:** a outbox armazena o JSON completo de cada evento, o que aumenta o consumo de storage em relação a guardar só o `order_id` — agravado pelo fato de o arquivamento após 30 dias estar fora de escopo ([09:08] Diego).
- **Payload congelado, mesmo se incorreto:** se a renderização na inserção tiver um defeito, o evento carrega o defeito para sempre; não há como "re-renderizar" no envio.
- **Requer renderização dentro da transação:** a montagem do payload passa a fazer parte da transação crítica do `changeStatus`, somando trabalho a uma transação já pesada ([09:04] Bruno).
- **Rigidez a evolução do schema de payload:** uma mudança no formato do payload não retroage sobre eventos já enfileirados, que continuarão no formato antigo até serem entregues.

### Trade-off explícito

Aceita-se **duplicar o payload no banco** e **assumir trabalho adicional dentro da transação** em troca de **um evento temporalmente consistente** — que descreve o pedido como ele estava quando o status mudou, e não como ele está quando o evento finalmente é entregue.

## Referências

- Transcrição: [09:04] Bruno, [09:08] Diego, [09:43] Diego, [09:51] Bruno, [09:52] Larissa, [09:52] Diego, [09:52] Bruno
- Código: `src/modules/orders/order.service.ts` (`changeStatus` — origem dos dados renderizados), `src/modules/orders/order.repository.ts` (`OrderWithRelations` — campos disponíveis na transação), `prisma/schema.prisma` (modelo `Order`)
