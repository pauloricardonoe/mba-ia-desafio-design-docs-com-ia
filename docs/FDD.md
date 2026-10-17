# FDD — Feature Design Document: Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| **Feature** | Webhooks outbound de mudança de status de pedido |
| **Status** | Pronto para implementação |
| **Data** | 2026-10-17 |
| **Documento pai** | `docs/RFC.md` |
| **Decisões** | `docs/adrs/` (ADR-001 a ADR-007) |

---

## 1. Contexto e motivação técnica

A aplicação não possui hoje nenhum mecanismo de eventos, filas ou notificações externas. Clientes B2B fazem polling no `GET /orders` para detectar mudanças de status, o que é lento e caro do lado deles.

O ponto de origem de toda notificação é o método `OrderService.changeStatus` (`src/modules/orders/order.service.ts:126`), que executa uma transação com três efeitos: atualiza `orders.status`, insere em `order_status_history` e debita ou repõe estoque. Essa transação **já é pesada**, e é justamente por isso que a notificação não pode ser síncrona: um cliente lento travaria a mudança de status de outros pedidos, e um cliente indisponível forçaria rollback da mudança de status ([09:04] Bruno).

A solução adotada é o **padrão Outbox no MySQL** ([ADR-001](adrs/ADR-001-outbox-no-mysql.md)): o evento é persistido na mesma transação que muda o status, e um **worker em processo separado** ([ADR-002](adrs/ADR-002-worker-em-processo-separado-com-polling.md)) consome a outbox por polling e executa as chamadas HTTP.

### 1.1 Objetivos técnicos

| ID | Objetivo | Como é verificado |
| --- | --- | --- |
| FDD-OT-01 | Atomicidade entre mudança de status e registro do evento | Nenhum commit de status sem linha correspondente na outbox (teste de rollback) |
| FDD-OT-02 | Latência de entrega abaixo de 10s no caso feliz | Piso estrutural de 2s (polling) + tempo de chamada HTTP |
| FDD-OT-03 | Entrega resiliente a indisponibilidade do cliente por até ~15h | 5 tentativas com backoff 1m/5m/30m/2h/12h |
| FDD-OT-04 | Autenticidade e integridade verificáveis pelo cliente | Assinatura HMAC-SHA256 no header `X-Signature` |
| FDD-OT-05 | Deduplicação possível pelo cliente | `X-Event-Id` estável entre tentativas do mesmo evento |
| FDD-OT-06 | Zero infraestrutura nova | Reuso do MySQL, do PrismaClient e do Pino existentes |
| FDD-OT-07 | Consistência de padrões com o restante do projeto | Módulo `src/modules/webhooks` na mesma estrutura dos demais |

### 1.2 Escopo e exclusões

**Dentro do escopo:**

- Publicação de eventos de mudança de status na outbox, dentro da transação do `changeStatus`.
- Worker em processo separado com polling de 2 segundos, retry com backoff e DLQ.
- CRUD de configuração de webhooks, histórico de entregas e rotação de secret.
- Endpoint administrativo de replay de DLQ.
- Assinatura HMAC-SHA256, filtro de eventos por status e validações de schema.

**Fora do escopo (decidido na reunião):**

| Item | Origem |
| --- | --- |
| E-mail de aviso ao cliente quando o webhook falha repetidamente | [09:37] Larissa — "fora de escopo dessa fase" |
| Rate limiting de envio por cliente | [09:39] Larissa — "observar e decidir depois" |
| Dashboard visual / painel do cliente | [09:40] Larissa — "projeto separado do time de frontend" |
| Arquivamento das linhas entregues da outbox (após ~30 dias) | [09:08] Diego — "fora do escopo dessa feature" |
| Escala horizontal do worker / ordering com múltiplos workers | [09:13] Diego — "problema do futuro" |
| Webhooks inbound (cliente → plataforma) | [09:02] Marcos — "só saindo da gente pra eles" |

## 2. Modelo de dados

Duas tabelas novas, seguindo o padrão do projeto (chaves UUID `Char(36)`, `createdAt`/`updatedAt`).

### FDD-MODELO-01 — `webhook_endpoints` — configuração de webhook do cliente

| Campo | Tipo | Notas |
| --- | --- | --- |
| `id` | `Char(36)` PK | UUID, padrão do projeto ([09:51] Larissa) |
| `customerId` | `Char(36)` FK → `customers.id` | Vínculo com o cliente |
| `url` | `VarChar(2048)` | Deve ser `https` ([09:23] Sofia) |
| `secret` | `VarChar(255)` | Secret atual, por endpoint ([09:21] Sofia) |
| `previousSecret` | `VarChar(255)` nullable | Secret em grace period de 24h ([09:21] Sofia) |
| `previousSecretExpiresAt` | `DateTime` nullable | Fim do grace period |
| `subscribedStatuses` | `Json` | Lista de `OrderStatus` de interesse ([09:33] Marcos) |
| `active` | `Boolean` default `true` | Estado ativo ([09:21] Bruno) |
| `createdAt` / `updatedAt` | `DateTime` | Padrão do projeto |

Índice em `customerId`.

### FDD-MODELO-02 — `webhook_outbox` — fila de eventos

| Campo | Tipo | Notas |
| --- | --- | --- |
| `id` | `Char(36)` PK | UUID ([09:51] Larissa) |
| `eventId` | `Char(36)` | UUID exposto no header `X-Event-Id` ([09:25] Diego) |
| `webhookId` | `Char(36)` FK → `webhook_endpoints.id` | Endpoint de destino |
| `orderId` | `Char(36)` | Pedido de origem |
| `eventType` | `VarChar(64)` | `order.status_changed` |
| `payload` | `Json` | Payload **renderizado na inserção** ([ADR-007](adrs/ADR-007-snapshot-do-payload-na-insercao.md)) |
| `status` | `Enum` | `PENDING`, `PROCESSING`, `FAILED`, `DELIVERED` ([09:08] Diego) |
| `attempts` | `Int` default `0` | Contador de tentativas |
| `nextAttemptAt` | `DateTime` | Próximo momento elegível para tentativa |
| `lastError` | `Text` nullable | Motivo da última falha |
| `createdAt` / `updatedAt` | `DateTime` | |

Índices: `status`, `createdAt`, e composto `(status, nextAttemptAt)` para a query do worker ([09:08] Diego).

> O enum de status usa os nomes `PENDING`/`PROCESSING`/`FAILED`/`DELIVERED`, correspondentes a pendente/processando/falhou/entregue ([09:08] Diego).

### FDD-MODELO-03 — `webhook_dead_letter` — eventos esgotados

| Campo | Tipo | Notas |
| --- | --- | --- |
| `id` | `Char(36)` PK | UUID |
| `outboxId` | `Char(36)` | Evento original |
| `webhookId` | `Char(36)` | Endpoint de destino |
| `payload` | `Json` | Evidência para debug ([09:18] Diego) |
| `failureReason` | `Text` | Motivo da falha ([09:18] Diego) |
| `attempts` | `Int` | Total de tentativas realizadas |
| `failedAt` | `DateTime` | Timestamp ([09:18] Diego) |
| `replayedAt` | `DateTime` nullable | Auditoria de replay |
| `replayedById` | `Char(36)` nullable | Quem executou o replay ([09:36] Sofia) |

### FDD-MODELO-04 — `webhook_deliveries` — histórico de entregas

| Campo | Tipo | Notas |
| --- | --- | --- |
| `id` | `Char(36)` PK | UUID |
| `webhookId` | `Char(36)` | Endpoint |
| `eventId` | `Char(36)` | Correlação com o evento |
| `success` | `Boolean` | Sucesso/falha ([09:34] Marcos) |
| `responseStatus` | `Int` nullable | Status HTTP da resposta |
| `responseBody` | `Text` nullable | Corpo da resposta (truncado) |
| `durationMs` | `Int` | Tempo de resposta ([09:34] Marcos) |
| `attempt` | `Int` | Número da tentativa |
| `createdAt` | `DateTime` | |

## 3. Fluxos detalhados

### FDD-FLUXO-01 — Criação do evento na outbox (dentro da transação do `changeStatus`)

```
1. OrderService.changeStatus(id, { toStatus, reason }, userId)
2.   prisma.$transaction(tx => { ... })                       // transação já existente
3.     valida transição via canTransition(from, to)           // order.status.ts
4.     debitStock / replenishStock                            // como hoje
5.     tx.order.update({ status: to })                        // como hoje
6.     tx.orderStatusHistory.create({ ... })                  // como hoje
7.     publishWebhookEvent(tx, order, from, to)   ◄── NOVO
8.         seleciona webhook_endpoints ativos do customer
9.              WHERE customerId = order.customerId AND active = true
10.        filtra por subscribedStatuses contendo `to`         // filtro NA INSERÇÃO [09:34]
11.        para cada endpoint elegível:
12.            renderiza payload (snapshot) e gera eventId (UUID)
13.            tx.webhookOutbox.create({ status: PENDING, attempts: 0, ... })
14.  })  // commit atômico
```

Regras:

- **Filtro na inserção, não no envio.** Se nenhum webhook ativo do customer quer aquele status, nenhuma linha é inserida — "economiza linha na tabela" ([09:34] Bruno, [09:34] Diego).
- **Snapshot na inserção (FDD-FLUXO-05).** O `payload` é renderizado aqui e não é recalculado no envio ([ADR-007](adrs/ADR-007-snapshot-do-payload-na-insercao.md), [09:52] Larissa).
- **Atomicidade.** Se a inserção na outbox falhar, a transação inteira sofre rollback. Não pode existir status mudado sem evento ([09:40] Bruno, [09:41] Diego).
- **Assinatura da função.** `publishWebhookEvent(tx: Prisma.TransactionClient, order, fromStatus, toStatus)` — função pura que recebe o client de transação, sem injetar um repository inteiro no `OrderService` ([09:41] Bruno, [09:41] Diego).

### FDD-FLUXO-02 — Processamento pelo worker

```
loop a cada 2 segundos:                                       // [09:09] Diego
  eventos = SELECT * FROM webhook_outbox
              WHERE status = 'PENDING' AND nextAttemptAt <= NOW()
              ORDER BY createdAt ASC
              LIMIT <batch pequeno>                            // [09:08] Diego
  para cada evento:
     UPDATE status = 'PROCESSING'
     inicio = now()
     try:
        resposta = fetch(endpoint.url, {
           method: 'POST',
           headers: { ... },                                    // ver seção 4.3
           body: JSON.stringify(evento.payload),
           signal: AbortSignal.timeout(10_000)                  // [09:42] Diego
        })
        registra webhook_deliveries(success, status, durationMs)
        if resposta.ok (2xx):  status = 'DELIVERED'
        else:                  trata como falha
     catch (timeout / erro de rede):
        registra webhook_deliveries(success = false, durationMs)
        trata como falha
```

**Ordenação (FDD-FLUXO-06).** Com um único worker lendo em ordem de `createdAt`, o cliente recebe os eventos de um pedido em ordem. A garantia é **por `order_id` e apenas enquanto houver um único worker** — não é ordering global ([09:12] Diego, [09:13] Larissa).

### FDD-FLUXO-03 — Retry com backoff exponencial

Ao falhar, o evento **não** é descartado nem marcado como entregue. Ele volta para `PENDING` com `attempts + 1` e um `nextAttemptAt` calculado pela progressão acordada ([09:17] Diego):

| Tentativa | Espera após a falha | Momento acumulado |
| --- | --- | --- |
| 1ª falha | 1 minuto | 1 min |
| 2ª falha | 5 minutos | 6 min |
| 3ª falha | 30 minutos | 36 min |
| 4ª falha | 2 horas | ~2h36min |
| 5ª falha | 12 horas | ~14h36min |
| após a 5ª falha | — | **vai para a DLQ** |

A janela total entre a primeira falha e a última tentativa é de **quase 15 horas** ([09:17] Diego), validada pelo negócio como aceitável ([09:17] Marcos).

**Decisão sobre respostas não-2xx:** qualquer resposta fora da faixa 2xx é tratada como falha e entra no ciclo de retry. O timeout de 10 segundos ([09:42] Diego) é tratado da mesma forma.

### FDD-FLUXO-04 — Dead letter queue e replay

Quando `attempts` atinge 5 sem sucesso:

```
1. INSERT INTO webhook_dead_letter
     (payload, failureReason, attempts, failedAt)              // [09:18] Diego
2. UPDATE webhook_outbox.status = 'FAILED'
```

O evento sai do caminho quente do worker e passa a viver na DLQ, servindo como evidência para debug ([09:18] Diego).

**Replay manual** (`POST /api/v1/admin/webhooks/dead-letter/:id/replay`):

```
1. authenticate  →  requireRole('ADMIN')                        // [09:36] Larissa, Sofia
2. localiza o registro na DLQ  (404 WEBHOOK_DEAD_LETTER_NOT_FOUND)
3. INSERT novo evento em webhook_outbox com status = PENDING
     mantendo o MESMO eventId (dedup do cliente continua válida)
4. UPDATE webhook_dead_letter.replayedAt = now(),
     replayedById = req.user.id                                 // [09:36] Sofia (auditoria)
```

O replay recoloca o evento na outbox como pendente ([09:18] Diego). Como o `eventId` é preservado, um cliente que já tenha recebido o evento consegue deduplicá-lo ([ADR-005](adrs/ADR-005-garantia-at-least-once-com-x-event-id.md)).

## 4. Contratos públicos

Base path: `/api/v1` (**FDD-CONTRATO-08**). Todos os endpoints exigem JWT válido via `authenticate` (`src/middlewares/auth.middleware.ts`), exceto onde indicado.

### FDD-CONTRATO-01 — `POST /api/v1/webhooks` — cadastrar webhook

Autenticado. Qualquer role autenticada ([09:37] Sofia). A secret é **gerada pela plataforma** e devolvida apenas na criação ([09:31] Marcos).

**Request:**

```json
{
  "customerId": "3f1c9c1e-6a1e-4f5a-9b1a-2c7d8e9f0a11",
  "url": "https://integracao.atlas.com.br/webhooks/oms",
  "subscribedStatuses": ["SHIPPED", "DELIVERED"]
}
```

**Response `201 Created`:**

```json
{
  "id": "8b2d4f60-1c33-4a2e-8f7d-5a6b7c8d9e0f",
  "customerId": "3f1c9c1e-6a1e-4f5a-9b1a-2c7d8e9f0a11",
  "url": "https://integracao.atlas.com.br/webhooks/oms",
  "secret": "whsec_9f8e7d6c5b4a39281706f5e4d3c2b1a0",
  "subscribedStatuses": ["SHIPPED", "DELIVERED"],
  "active": true,
  "createdAt": "2026-10-17T12:00:00.000Z"
}
```

| Status | Semântica |
| --- | --- |
| `201` | Webhook criado; secret devolvida |
| `400` | Payload inválido (`VALIDATION_ERROR`) ou URL não-https (`WEBHOOK_INVALID_URL`) |
| `401` | Token ausente/inválido |
| `404` | `customerId` inexistente (`NOT_FOUND`) |

### FDD-CONTRATO-02 — `GET /api/v1/webhooks?customerId=...` — listar webhooks

Autenticado. Retorna os webhooks de um cliente, no formato paginado padrão do projeto — **FDD-CONTRATO-09** (`src/shared/http/response.ts`).

**Response `200 OK`:**

```json
{
  "data": [
    {
      "id": "8b2d4f60-1c33-4a2e-8f7d-5a6b7c8d9e0f",
      "customerId": "3f1c9c1e-6a1e-4f5a-9b1a-2c7d8e9f0a11",
      "url": "https://integracao.atlas.com.br/webhooks/oms",
      "subscribedStatuses": ["SHIPPED", "DELIVERED"],
      "active": true,
      "createdAt": "2026-10-17T12:00:00.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```

> A secret **não** é devolvida na listagem — apenas na criação e na rotação.

| Status | Semântica |
| --- | --- |
| `200` | Lista retornada |
| `401` | Token ausente/inválido |

### FDD-CONTRATO-03 — `PATCH /api/v1/webhooks/:id` — editar webhook

Autenticado. Permite alterar `url`, `subscribedStatuses` e `active` ([09:33] Bruno).

**Request:**

```json
{
  "url": "https://integracao.atlas.com.br/webhooks/oms-v2",
  "subscribedStatuses": ["PAID", "SHIPPED", "DELIVERED"],
  "active": true
}
```

**Response `200 OK`:** objeto do webhook atualizado (mesmo formato do item da listagem).

| Status | Semântica |
| --- | --- |
| `200` | Atualizado |
| `400` | Payload inválido ou URL não-https (`WEBHOOK_INVALID_URL`) |
| `401` | Token ausente/inválido |
| `404` | Webhook inexistente (`WEBHOOK_NOT_FOUND`) |

### FDD-CONTRATO-04 — `DELETE /api/v1/webhooks/:id` — remover webhook

Autenticado ([09:33] Bruno).

| Status | Semântica |
| --- | --- |
| `204` | Removido, sem corpo |
| `401` | Token ausente/inválido |
| `404` | Webhook inexistente (`WEBHOOK_NOT_FOUND`) |

### FDD-CONTRATO-05 — `POST /api/v1/webhooks/:id/rotate-secret` — rotacionar secret

Autenticado. Gera uma nova secret e mantém a antiga válida por **24 horas** ([09:21] Sofia).

**Response `200 OK`:**

```json
{
  "id": "8b2d4f60-1c33-4a2e-8f7d-5a6b7c8d9e0f",
  "secret": "whsec_1a2b3c4d5e6f708192a3b4c5d6e7f809",
  "previousSecretExpiresAt": "2026-10-18T12:00:00.000Z"
}
```

| Status | Semântica |
| --- | --- |
| `200` | Nova secret emitida; antiga válida até `previousSecretExpiresAt` |
| `401` | Token ausente/inválido |
| `404` | Webhook inexistente (`WEBHOOK_NOT_FOUND`) |

### FDD-CONTRATO-06 — `GET /api/v1/webhooks/:id/deliveries` — histórico de entregas

Autenticado. Retorna as últimas 100 entregas do webhook, com sucesso/falha, payload, resposta e tempo de resposta ([09:34] Marcos).

**Response `200 OK`:**

```json
{
  "data": [
    {
      "id": "c1a2b3d4-e5f6-4708-9a1b-2c3d4e5f6a7b",
      "eventId": "5e4d3c2b-1a09-4876-b5c4-d3e2f1a0b9c8",
      "success": true,
      "responseStatus": 200,
      "responseBody": "{\"received\":true}",
      "durationMs": 143,
      "attempt": 1,
      "createdAt": "2026-10-17T12:00:02.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 100, "total": 1, "totalPages": 1 }
}
```

| Status | Semântica |
| --- | --- |
| `200` | Histórico retornado (máximo 100 registros) |
| `401` | Token ausente/inválido |
| `404` | Webhook inexistente (`WEBHOOK_NOT_FOUND`) |

### FDD-CONTRATO-07 — `POST /api/v1/admin/webhooks/dead-letter/:id/replay` — replay de DLQ

**Restrito à role `ADMIN`** via `requireRole('ADMIN')` ([09:36] Larissa). O autor do replay é registrado para auditoria ([09:36] Sofia).

> Na reunião o endpoint foi referido como `POST /admin/webhooks/dead-letter/:id/replay` ([09:35] Diego); sob o roteamento versionado da aplicação, o caminho completo é `/api/v1/admin/webhooks/dead-letter/:id/replay`.

**Response `202 Accepted`:**

```json
{
  "replayed": true,
  "outboxId": "a0b1c2d3-e4f5-4607-8899-aabbccddeeff",
  "eventId": "5e4d3c2b-1a09-4876-b5c4-d3e2f1a0b9c8",
  "replayedById": "9c8b7a65-4321-4f0e-9d8c-7b6a5f4e3d2c"
}
```

| Status | Semântica |
| --- | --- |
| `202` | Evento recolocado na outbox como pendente |
| `401` | Token ausente/inválido |
| `403` | Role diferente de `ADMIN` (`FORBIDDEN`) |
| `404` | Registro de DLQ inexistente (`WEBHOOK_DEAD_LETTER_NOT_FOUND`) |

### FDD-HEADER-01 a FDD-HEADER-05 — Headers das chamadas de saída

Toda chamada HTTP do worker para o cliente leva ([09:44] Diego, [09:44] Sofia):

| ID | Header | Valor | Propósito |
| --- | --- | --- | --- |
| FDD-HEADER-01 | `Content-Type` | `application/json` | Formato do corpo |
| FDD-HEADER-02 | `X-Event-Id` | UUID do evento | Deduplicação no cliente ([09:25] Diego) |
| FDD-HEADER-03 | `X-Signature` | HMAC-SHA256 do corpo, em hex | Autenticidade e integridade ([09:20] Sofia) |
| FDD-HEADER-04 | `X-Timestamp` | Timestamp do envio (ISO 8601) | Cliente pode detectar replay attack ([09:44] Diego) |
| FDD-HEADER-05 | `X-Webhook-Id` | Id do endpoint de webhook | Cliente com vários cadastros identifica qual disparou ([09:44] Sofia) |

**Corpo enviado — payload enxuto, sem `items` (FDD-PAYLOAD-01):**

```json
{
  "event_id": "5e4d3c2b-1a09-4876-b5c4-d3e2f1a0b9c8",
  "event_type": "order.status_changed",
  "timestamp": "2026-10-17T12:00:00.000Z",
  "order_id": "7d6c5b4a-3928-4f1e-8d7c-6b5a4f3e2d1c",
  "order_number": "ORD-000123",
  "from_status": "PAID",
  "to_status": "PROCESSING",
  "customer_id": "3f1c9c1e-6a1e-4f5a-9b1a-2c7d8e9f0a11",
  "total_cents": 15000
}
```

Conforme definido por Diego: JSON com `event_id`, `event_type`, `timestamp`, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e os campos básicos da order — **sem `items`, para não inflar o payload**. Se o cliente quiser detalhes, consulta o `GET /orders/:id` ([09:43] Diego, [09:43] Bruno).

- **FDD-PAYLOAD-02** — `event_type` no formato `order.status_changed` ([09:43] Diego).
- **FDD-PAYLOAD-03** — `timestamp` em ISO 8601 ([09:43] Diego).

**Cálculo da assinatura:**

```
X-Signature = HMAC-SHA256(secret_do_endpoint, corpo_bruto_do_request) em hexadecimal
```

O cliente recalcula do lado dele e compara. Durante a janela de rotação, o cliente deve validar contra a secret atual **e** contra a anterior até `previousSecretExpiresAt` ([09:21] Sofia).

## 5. Matriz de erros previstos

**FDD-ERRO-01.** Todos os códigos do módulo usam o prefixo `WEBHOOK_` ([09:29] Larissa). As classes herdam de `AppError`/subclasses existentes (`src/shared/errors/http-errors.ts`), de modo que o error middleware centralizado já os serializa no formato padrão ([09:28] Bruno, [09:29] Bruno).

| ID | Código | HTTP | Classe base | Quando ocorre | Origem |
| --- | --- | --- | --- | --- | --- |
| FDD-ERRO-02 | `WEBHOOK_NOT_FOUND` | 404 | `NotFoundError` | Webhook inexistente no CRUD/deliveries | [09:28] Bruno |
| FDD-ERRO-03 | `WEBHOOK_INVALID_URL` | 400 | `ValidationError` | URL cadastrada não é `https` | [09:23] Sofia, [09:28] Bruno |
| FDD-ERRO-04 | `WEBHOOK_SECRET_REQUIRED` | 400 | `ValidationError` | Operação exige secret ausente/inválida | [09:28] Bruno |
| FDD-ERRO-05 | `WEBHOOK_PAYLOAD_TOO_LARGE` | 422 | `UnprocessableEntityError` | Payload do evento ultrapassa 64KB | [09:24] Larissa, [09:24] Diego |
| FDD-ERRO-06 | `WEBHOOK_CUSTOMER_NOT_FOUND` | 404 | `NotFoundError` | `customerId` inexistente no cadastro | [09:31] Marcos |
| FDD-ERRO-07 | `WEBHOOK_INVALID_STATUS_FILTER` | 400 | `ValidationError` | Lista de status assinados contém valor inválido | [09:33] Marcos |
| FDD-ERRO-08 | `WEBHOOK_DELIVERY_NOT_FOUND` | 404 | `NotFoundError` | Histórico solicitado não existe | [09:34] Marcos |
| FDD-ERRO-09 | `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | `NotFoundError` | Registro de DLQ inexistente no replay | [09:18] Diego |
| FDD-ERRO-10 | `WEBHOOK_DELIVERY_FAILED` | — | (interno) | Motivo gravado na DLQ após 5 tentativas | [09:18] Diego |
| FDD-ERRO-11 | `WEBHOOK_DELIVERY_TIMEOUT` | — | (interno) | Cliente não respondeu em 10s | [09:42] Diego |

Erros transversais reutilizados sem alteração: `VALIDATION_ERROR`, `UNAUTHORIZED`, `FORBIDDEN`, `NOT_FOUND`, `CONFLICT` (`src/shared/errors/http-errors.ts`).

Formato de resposta de erro — **FDD-ERRO-12** (inalterado, produzido pelo middleware existente):

```json
{ "error": { "code": "WEBHOOK_INVALID_URL", "message": "Webhook URL must use https" } }
```

## 6. Estratégias de resiliência

| ID | Mecanismo | Valor | Origem |
| --- | --- | --- | --- |
| FDD-RESIL-01 | **Timeout da chamada HTTP** | 10 segundos; estouro é tratado como falha e entra no retry | [09:42] Diego |
| FDD-RESIL-02 | **Tentativas** | 5 no total | [09:15] Diego, [09:16] Larissa |
| FDD-RESIL-03 | **Backoff** | 1m / 5m / 30m / 2h / 12h (exponencial) | [09:17] Diego |
| FDD-RESIL-04 | **Janela total** | ~15 horas entre primeira falha e última tentativa | [09:17] Diego |
| FDD-RESIL-05 | **DLQ** | Tabela `webhook_dead_letter` após a 5ª falha | [09:18] Diego |
| FDD-RESIL-06 | **Replay** | Manual, via endpoint admin com role `ADMIN` | [09:18] Diego, [09:36] Larissa |
| FDD-RESIL-07 | **Fallback de aviso ao cliente** | **Nenhum nesta fase** — e-mail está fora de escopo | [09:37] Larissa |
| FDD-RESIL-08 | **Idempotência** | At-least-once; dedup pelo cliente via `X-Event-Id` | [ADR-005](adrs/ADR-005-garantia-at-least-once-com-x-event-id.md) |
| FDD-RESIL-09 | **Isolamento de falha do worker** | Processo separado da API | [09:11] Diego |

**Ausência de fallback explícita:** não há notificação alternativa (e-mail, SMS) quando o webhook falha. O fallback discutido — avisar o cliente por e-mail após 3 falhas seguidas — foi **deliberadamente adiado** para uma próxima fase ([09:37] Marcos, [09:37] Larissa). Nesta fase, o mecanismo de recuperação é o retry automático seguido de DLQ com replay manual.

**Isolamento entre endpoints:** uma falha de um endpoint de webhook não deve impedir o processamento dos demais eventos. O worker trata cada evento individualmente dentro do ciclo.

## 7. Observabilidade

O projeto usa **Pino** como logger (`src/shared/logger/index.ts`), com redact de campos sensíveis e o middleware `requestLogger` que injeta `req.id` e um `X-Request-Id` por requisição (`src/middlewares/request-logger.middleware.ts`). Não existe hoje biblioteca de métricas ou de tracing distribuído no projeto — os itens marcados como **propostos** abaixo exigem confirmação na implementação.

### FDD-OBS-01 — Logs (Pino, existente)

Eventos estruturados a emitir, sempre com `eventId`, `webhookId`, `orderId` e `attempt`:

| Evento | Nível | Quando |
| --- | --- | --- |
| `webhook_event_enqueued` | `info` | Inserção na outbox dentro da transação |
| `webhook_delivery_attempt` | `info` | Início de cada tentativa de envio |
| `webhook_delivery_succeeded` | `info` | Resposta 2xx com `durationMs` |
| `webhook_delivery_failed` | `warn` | Falha com motivo e próxima tentativa agendada |
| `webhook_moved_to_dlq` | `error` | Esgotamento das 5 tentativas |
| `webhook_replayed` | `info` | Replay manual, com `replayedById` (auditoria) |
| `webhook_worker_started` | `info` | Boot do worker |
| `webhook_worker_cycle` | `debug` | Fim de cada ciclo de polling, com contagem processada |

A diretriz de projeto é **não introduzir logger novo** — tudo passa pelo Pino existente ([09:29] Bruno).

### FDD-OBS-02 — Métricas (propostas)

Sem biblioteca de métricas hoje. Conjunto mínimo proposto:

| Métrica | Tipo | O que mede |
| --- | --- | --- |
| `webhook_events_enqueued_total` | contador | Eventos inseridos na outbox |
| `webhook_deliveries_total{result}` | contador | Entregas por resultado (`success`/`failure`) |
| `webhook_delivery_duration_ms` | histograma | Tempo de resposta do cliente |
| `webhook_outbox_pending` | gauge | Eventos pendentes na outbox (alerta de acúmulo) |
| `webhook_dlq_size` | gauge | Eventos na DLQ |
| `webhook_retry_attempts` | histograma | Distribuição de tentativas por evento |
| `webhook_worker_cycle_duration_ms` | histograma | Duração do ciclo de polling |

### FDD-OBS-03 — Tracing (proposto)

Não há tracing distribuído no projeto. Proposta: propagar o `eventId` como identificador de correlação entre o log da transação de origem (`changeStatus`), o log de cada tentativa do worker e o registro em `webhook_deliveries`, permitindo reconstruir a jornada de um evento ponta a ponta. O `X-Request-Id` já existente cobre a correlação das requisições HTTP de entrada.

## 8. Dependências e compatibilidade

| ID | Dependência | Situação |
| --- | --- | --- |
| FDD-DEP-01 | **MySQL existente** | Reusado; nenhuma instância nova ([09:07] Diego) |
| FDD-DEP-02 | **Prisma / `PrismaClient`** | Reusado; worker instancia o seu próprio client por ser outro processo ([09:30] Bruno) |
| FDD-DEP-03 | **Pino** | Reusado sem alteração ([09:29] Bruno) |
| FDD-DEP-04 | **Zod** | Reusado para os schemas de validação do módulo ([09:30] Larissa) |
| FDD-DEP-05 | **`uuid`** | Já é dependência do projeto (`package.json`), usada em `src/middlewares/request-logger.middleware.ts` — reaproveitada para `eventId` |
| FDD-DEP-06 | **Cliente HTTP** | `fetch` global do Node 20 (o projeto exige `node >= 20` em `package.json`), com `AbortSignal.timeout` — **sem nova dependência** |
| FDD-DEP-07 | **HMAC** | `crypto.createHmac` do Node (built-in) — **sem nova dependência** |
| FDD-DEP-08 | **Biblioteca de métricas** | **Não existe hoje** — a definir na implementação (FDD-OBS-02) |

**Compatibilidade (FDD-DEP-09):** nenhuma alteração é necessária no error middleware, no logger, nos módulos existentes ou nas configurações. A única mudança em código existente é a extensão do `changeStatus` ([09:29] Bruno).

**Migração (FDD-DEP-10):** duas migrations Prisma novas (tabelas de webhooks e tabelas de entrega), aditivas. Nenhuma tabela existente é alterada.

## 9. Critérios de aceite técnicos

| ID | Critério |
| --- | --- |
| FDD-CA-01 | Uma mudança de status que falha (rollback) não deixa nenhuma linha na `webhook_outbox` |
| FDD-CA-02 | Uma mudança de status bem-sucedida deixa exatamente uma linha por webhook ativo que assina o novo status |
| FDD-CA-03 | Nenhuma linha é inserida se nenhum webhook do customer assina o status de destino |
| FDD-CA-04 | O payload persistido é o snapshot do estado no momento da mudança, não o estado atual do pedido |
| FDD-CA-05 | O worker entrega eventos pendentes em até 2 segundos do ciclo de polling |
| FDD-CA-06 | Eventos de um mesmo `order_id` são entregues na ordem de `createdAt` da outbox |
| FDD-CA-07 | Uma falha agenda a próxima tentativa conforme a progressão 1m/5m/30m/2h/12h |
| FDD-CA-08 | Após 5 tentativas sem sucesso, o evento é movido para a DLQ com payload e motivo |
| FDD-CA-09 | O header `X-Signature` corresponde ao HMAC-SHA256 do corpo com a secret do endpoint |
| FDD-CA-10 | O mesmo `eventId` é enviado em todas as tentativas do mesmo evento |
| FDD-CA-11 | O replay exige role `ADMIN` e registra `replayedById` |
| FDD-CA-12 | Uma URL `http` é recusada no cadastro com `WEBHOOK_INVALID_URL` |
| FDD-CA-13 | Um payload acima de 64KB resulta em erro, não em truncamento |
| FDD-CA-14 | Durante o grace period, a secret anterior também valida a assinatura |
| FDD-CA-15 | O worker roda como processo independente e sobrevive a restart da API |
| FDD-CA-16 | Todos os erros do módulo usam o prefixo `WEBHOOK_` e o formato de erro padrão |

## 10. Riscos e mitigação

| ID | Risco | Prob. | Impacto | Mitigação |
| --- | --- | --- | --- | --- |
| FDD-RISCO-01 | Transação do `changeStatus` mais longa, aumentando contenção e latência do endpoint | Média | Médio | Inserção única e indexada; renderização do payload reaproveita dados já carregados na transação; monitorar `durationMs` do `PATCH /orders/:id/status` |
| FDD-RISCO-02 | Worker é ponto único de falha (single-worker, sem ordering global sob escala) | Média | Alto | Processo separado + alerta no gauge `webhook_outbox_pending`; escalabilidade mapeada como questão em aberto no RFC (Q3) |
| FDD-RISCO-03 | Crescimento indefinido da outbox sem arquivamento | Alta | Médio | Índices em `status` e `createdAt`; arquivamento após 30 dias explicitamente fora de escopo ([09:08] Diego) — reavaliar |
| FDD-RISCO-04 | Cliente não implementa deduplicação e processa duplicatas | Média | Médio | Documentação destacada no portal do desenvolvedor ([09:26] Marcos); `eventId` estável entre tentativas |
| FDD-RISCO-05 | Vazamento da secret do lado do cliente | Média | Alto | Secret por endpoint (raio limitado) + rotação com grace period de 24h ([ADR-004](adrs/ADR-004-assinatura-hmac-sha256-secret-por-endpoint.md)); redact no Pino |
| FDD-RISCO-06 | Cliente indisponível por mais de ~15h → evento cai na DLQ e fica parado | Baixa | Médio | DLQ como evidência + replay manual; nenhum aviso automático nesta fase (e-mail fora de escopo) |
| FDD-RISCO-07 | Chamada HTTP sem timeout travaria o ciclo do worker | Baixa | Alto | Timeout de 10s em toda chamada ([09:42] Diego) |
| FDD-RISCO-08 | Payload grande pressiona a transação e o armazenamento | Baixa | Baixo | Limite de 64KB com erro ([09:24] Larissa) |

## 11. Integração com o sistema existente

Esta seção mapeia os pontos de acoplamento com o código atual. A diretriz é **reuso máximo**: o módulo de webhooks se encaixa nos padrões já existentes ([09:30] Larissa, [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md)).

> **Convenção de leitura:** os caminhos citados em FDD-INTEG-01 a FDD-INTEG-09 são **arquivos existentes** no repositório, lidos e descritos como são hoje. Os caminhos abaixo são **arquivos novos a serem criados** pela feature e, por isso, não existem ainda no repositório:
>
> | Arquivo novo | Papel |
> | --- | --- |
> | `src/worker.ts` | Entry-point do processo worker, no molde de `src/server.ts` (FDD-INTEG-08) |
> | `src/modules/webhooks/webhook.worker.ts` | Lógica de processamento do worker (FDD-INTEG-08) |
> | `src/modules/webhooks/webhook.schemas.ts` | Schemas Zod do módulo (FDD-INTEG-06) |
> | `src/modules/webhooks/` (demais arquivos: `webhook.controller.ts`, `webhook.service.ts`, `webhook.repository.ts`, `webhook.routes.ts`) | Módulo completo, seguindo a estrutura dos demais módulos (FDD-INTEG-07) |
>
> Nenhum arquivo existente do projeto é renomeado, movido ou removido.

### FDD-INTEG-01 — `src/modules/orders/order.service.ts` — ponto de extensão principal

O método `changeStatus` (linha 126) é o único ponto alterado do código existente. Hoje ele executa `tx.order.update`, `tx.orderStatusHistory.create` e o ajuste de estoque dentro de `this.prisma.$transaction`. A alteração acrescenta uma chamada a `publishWebhookEvent(tx, order, from, to)` **dentro do mesmo callback transacional**, depois da atualização de status e do histórico.

A função recebe o `Prisma.TransactionClient` (`TxClient`, já tipado no arquivo) e insere na `webhook_outbox`. Se a inserção falhar, a exceção propaga e a transação inteira sofre rollback — garantindo a atomicidade exigida ([09:40] Bruno, [09:41] Bruno). O `OrderService` **não** recebe um repository de webhooks injetado; a dependência é a função pura ([09:41] Diego).

O padrão de injeção existente — `constructor(private readonly orders: OrderRepository, private readonly prisma: PrismaClient)` — é mantido.

### FDD-INTEG-02 — `src/modules/orders/order.status.ts` — máquina de estados e filtro de eventos

O módulo reutiliza as funções exportadas por este arquivo. `canTransition(from, to)` continua sendo a fonte de verdade sobre transições válidas; o conjunto de status assináveis por um webhook é exatamente o enum `OrderStatus` (`PENDING`, `PAID`, `PROCESSING`, `SHIPPED`, `DELIVERED`, `CANCELLED`), e o filtro de eventos na inserção compara `subscribedStatuses` contra o status de destino.

`isTerminal(status)` é útil para o worker decidir sobre notificações de estados finais, e `shouldDebitStock`/`shouldReplenishStock` permanecem inalteradas — a publicação do evento é ortogonal ao efeito de estoque.

### FDD-INTEG-03 — `src/shared/errors/http-errors.ts` e `src/shared/errors/app-error.ts` — hierarquia de erros

Os erros do módulo herdam das classes existentes, seguindo o padrão de códigos já usado (`INSUFFICIENT_STOCK`, `INVALID_STATUS_TRANSITION`). Exemplos: `WebhookNotFoundError extends NotFoundError`, `WebhookInvalidUrlError extends ValidationError`, `WebhookPayloadTooLargeError extends UnprocessableEntityError`. O prefixo `WEBHOOK_` é aplicado no `errorCode` ([09:28] Bruno, [09:29] Larissa).

Os erros devem ser exportados por `src/shared/errors/index.ts`, como já ocorre com os demais.

### FDD-INTEG-04 — `src/middlewares/error.middleware.ts` — tratamento centralizado

**Nenhuma alteração necessária.** Como os erros do módulo são `AppError`, o middleware já os serializa no formato `{ error: { code, message, details } }` no primeiro bloco do handler ([09:29] Bruno). Isso significa que a matriz de erros da seção 5 funciona sem tocar neste arquivo.

### FDD-INTEG-05 — `src/middlewares/auth.middleware.ts` — autorização do endpoint admin

O endpoint de replay usa `requireRole('ADMIN')`, reaproveitando a função exportada por este arquivo ([09:36] Larissa). O precedente já existe em `src/modules/users/user.routes.ts`, que aplica `authenticate, requireRole('ADMIN'), validate(...)` na mesma rota. O middleware `authenticate` popula `req.user`, de onde o replay lê `req.user.id` para gravar `replayedById`.

### FDD-INTEG-06 — `src/middlewares/validate.middleware.ts` e `src/modules/orders/order.schemas.ts` — validação

As rotas do módulo usam o `validate({ body, params, query })` existente, com schemas Zod no arquivo `src/modules/webhooks/webhook.schemas.ts`, seguindo o padrão de `order.schemas.ts`. É aqui que vivem as validações acordadas: URL `https` ([09:23] Sofia) e lista de status assinados válida ([09:33] Marcos). Erros de Zod caem automaticamente em `VALIDATION_ERROR`.

### FDD-INTEG-07 — `src/app.ts` e `src/routes/index.ts` — composição e roteamento

O `WebhookController` é instanciado em `buildControllers` (`src/app.ts`) e registrado no `Controllers` e no `buildApiRouter` (`src/routes/index.ts`), exatamente como os controllers existentes. O roteador do módulo é montado sob `/api/v1/webhooks`, e o roteador administrativo sob `/api/v1/admin/webhooks`, mantendo o padrão de versionamento atual.

### FDD-INTEG-08 — `src/config/database.ts` e `src/server.ts` — worker como nova entry-point

O `src/worker.ts` segue o molde do `src/server.ts`: bootstrap, logging de inicialização e tratamento de `SIGINT`/`SIGTERM` para shutdown gracioso, chamando `prisma.$disconnect()`. Como `PrismaClient` é por processo, o worker usa `createPrismaClient()` de `src/config/database.ts` para obter sua própria instância, com a mesma `DATABASE_URL` ([09:30] Bruno). Um script `"worker"` é adicionado ao `package.json`, no molde de `"dev"` ([09:11] Larissa).

A lógica de processamento fica em `src/modules/webhooks/webhook.worker.ts`, e não no entry-point ([09:28] Bruno).

### FDD-INTEG-09 — `prisma/schema.prisma` — novos modelos

Os modelos das seções 2.1 a 2.4 são adicionados ao schema, seguindo as convenções do arquivo: chaves `String @id @default(uuid()) @db.Char(36)`, `@@map` para nomes de tabela em snake_case, `@@index` nos campos de filtro. O enum de status da outbox é declarado junto aos enums existentes `UserRole` e `OrderStatus`.
