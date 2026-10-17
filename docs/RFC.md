# RFC — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| **Título** | Sistema de Webhooks de Notificação de Pedidos (outbound) |
| **Autor** | Larissa (Tech Lead) |
| **Revisores** | Marcos (Product Manager), Bruno (Engenheiro Pleno, Pedidos), Diego (Engenheiro Sênior, Plataforma), Sofia (Engenheira de Segurança) |
| **Status** | Em revisão |
| **Data** | 2026-10-17 |
| **Origem** | Reunião técnica de quinta-feira, 09:00 (~55 min) — ver `TRANSCRICAO.md` |
| **Documentos relacionados** | `docs/PRD.md`, `docs/FDD.md`, `docs/adrs/` |

---

## 1. Resumo executivo (TL;DR)

Propomos adicionar **webhooks outbound** ao Order Management System: sempre que o status de um pedido muda, a plataforma notifica por HTTP os endpoints cadastrados pelos clientes, em vez de deixá-los fazendo polling no `GET /orders`.

A proposta usa o **padrão Outbox no MySQL já existente**: o evento é gravado na mesma transação que hoje muda o status do pedido, e um **worker em processo separado**, com polling de 2 segundos, é o responsável pela entrega. As chamadas são assinadas com **HMAC-SHA256** usando secret por endpoint, entregues com garantia **at-least-once** (deduplicação pelo cliente via `X-Event-Id`) e protegidas por **retry com backoff exponencial** e uma **dead letter queue** para eventos que esgotam as tentativas.

A proposta não introduz nenhuma infraestrutura nova — nem broker, nem cache, nem serviço separado — e reutiliza integralmente os padrões já estabelecidos no projeto.

## 2. Contexto e problema

Três clientes B2B — Atlas Comercial, MaxDistribuição e Nova Cargo — pediram formalmente ser notificados quando o status dos pedidos deles muda. Hoje eles fazem polling periódico no `GET /orders`, o que torna a integração lenta e cara do lado deles. A Atlas sinalizou risco de migração para o concorrente caso a entrega não ocorra até o fim do trimestre.

O requisito de latência foi calibrado diretamente com o cliente: **abaixo de 10 segundos já é considerado "tempo real"** ([09:02] Marcos). O escopo é **estritamente outbound** — os clientes recebem, não enviam ([09:02] Sofia, [09:02] Marcos).

Do lado técnico, o problema se concentra no método `changeStatus` do `OrderService` (`src/modules/orders/order.service.ts:126`), que executa uma transação com três efeitos: atualiza `orders.status`, insere uma linha em `order_status_history` e ajusta o estoque dos produtos do pedido. A aplicação **não possui hoje nenhum mecanismo de eventos, filas ou webhooks** — esse vácuo é exatamente o que esta proposta preenche.

A restrição central é de consistência: **não pode existir o caso de o status mudar e o evento não sair** ([09:40] Bruno).

## 3. Proposta técnica

### 3.1 Visão geral

```
PATCH /orders/:id/status
        │
        ▼
┌─────────────────────────────────────────────┐
│ OrderService.changeStatus  (transação única) │
│  • UPDATE orders.status                      │
│  • INSERT order_status_history               │
│  • débito/reposição de estoque               │
│  • INSERT webhook_outbox  ◄── NOVO (snapshot)│
└─────────────────────────────────────────────┘
        │ commit atômico
        ▼
   webhook_outbox  ──polling 2s──▶  Worker (processo separado)
                                        │
                                        ├─ assina HMAC-SHA256
                                        ├─ POST https://cliente/...
                                        └─ falha ─▶ retry (backoff) ─▶ DLQ
```

### 3.2 Componentes

- **Módulo `src/modules/webhooks/`** — novo módulo seguindo a estrutura modular já existente no projeto (controller, service, repository, routes, schemas).
- **Tabela `webhook_outbox`** — fila persistida no MySQL, com índice em status e `created_at`. Guarda o **payload já renderizado** (snapshot) no momento da inserção.
- **Tabela `webhook_dead_letter`** — eventos que esgotaram as tentativas, com payload, motivo e timestamp.
- **`src/worker.ts`** — nova entry-point, nos moldes do `src/server.ts`, com a lógica de processamento no módulo de webhooks.
- **Tabela de configuração de webhook** — `url` + `secret` + `customer_id` + estado ativo + lista de status de interesse.

### 3.3 Decisões de arquitetura

| Tema | Decisão | ADR |
| --- | --- | --- |
| Publicação do evento | Outbox no MySQL, dentro da transação do `changeStatus` | [ADR-001](adrs/ADR-001-outbox-no-mysql.md) |
| Consumo | Worker em processo separado, polling de 2s | [ADR-002](adrs/ADR-002-worker-em-processo-separado-com-polling.md) |
| Falha de entrega | 5 tentativas, backoff 1m/5m/30m/2h/12h, DLQ em tabela separada | [ADR-003](adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md) |
| Autenticidade | HMAC-SHA256, secret por endpoint, rotação com grace period de 24h | [ADR-004](adrs/ADR-004-assinatura-hmac-sha256-secret-por-endpoint.md) |
| Garantia de entrega | At-least-once com `X-Event-Id` para dedup no cliente | [ADR-005](adrs/ADR-005-garantia-at-least-once-com-x-event-id.md) |
| Padrões de código | Reuso máximo do que já existe no projeto | [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md) |
| Formato do evento | Snapshot renderizado na inserção | [ADR-007](adrs/ADR-007-snapshot-do-payload-na-insercao.md) |

### 3.4 Superfície pública

A feature expõe um CRUD de configuração de webhooks autenticado pelo JWT já existente, um endpoint de histórico de entregas e um endpoint administrativo de replay de DLQ restrito à role `ADMIN`. O detalhamento de contratos, payloads, headers e códigos de erro está no `docs/FDD.md`.

### 3.5 Integração com o sistema existente

O único ponto de acoplamento com o código atual é o `changeStatus`. A proposta é introduzir uma função **`publishWebhookEvent(tx, order, fromStatus, toStatus)`** que recebe o client de transação (`Prisma.TransactionClient`) e insere o evento na outbox. O `OrderService` apenas a invoca dentro da transação que já executa — sem injetar um repository inteiro no serviço ([09:41] Bruno, [09:41] Diego).

## 4. Alternativas consideradas

### RFC-ALT-01 — Envio síncrono dentro de `changeStatus`

Disparar o HTTP diretamente na transação de mudança de status.

**Trade-off que motivou o descarte:** a transação já é pesada e um cliente lento travaria a mudança de status de **outros** pedidos; um cliente fora do ar forçaria rollback da mudança de status, o que é inaceitável ([09:04] Bruno). Detalhamento em [ADR-001](adrs/ADR-001-outbox-no-mysql.md).

### RFC-ALT-02 — Redis Streams como broker

Publicar o evento em um stream Redis e consumir com um worker dedicado.

**Trade-off que motivou o descarte:** exigiria subir infraestrutura nova (Redis Cluster) para um time pequeno, resolvendo um problema que o banco relacional já resolve. Classificado explicitamente como *overengineering* ([09:07] Diego, [09:07] Larissa). Detalhamento em [ADR-001](adrs/ADR-001-outbox-no-mysql.md).

### RFC-ALT-03 — Trigger de banco para consumo reativo

Usar uma trigger do MySQL para reagir à inserção na outbox.

**Trade-off que motivou o descarte:** o MySQL não tem listener nativo para processos externos (equivalente ao `NOTIFY`/`LISTEN` do Postgres); a trigger só executa SQL. Improvisar um canal de notificação foi considerado frágil ([09:09] Diego). Detalhamento em [ADR-002](adrs/ADR-002-worker-em-processo-separado-com-polling.md).

### RFC-ALT-04 — Retry indefinido com backoff

Continuar tentando indefinidamente até obter sucesso.

**Trade-off que motivou o descarte:** um evento de um cliente que desapareceu ficaria pendurado para sempre. O teto de tentativas dá a todo evento um estado terminal ([09:15] Diego). Detalhamento em [ADR-003](adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md).

### RFC-ALT-05 — Garantia exactly-once

Garantir entrega única ao cliente.

**Trade-off que motivou o descarte:** exigiria coordenação dos dois lados (protocolo de ack bilateral), muito mais complexo que o problema justifica, e ainda assim com casos de borda não eliminados. O padrão de mercado (Stripe, GitHub) é at-least-once com `event_id` ([09:25] Diego). Detalhamento em [ADR-005](adrs/ADR-005-garantia-at-least-once-com-x-event-id.md).

### RFC-ALT-06 — Secret global da plataforma

Uma única secret compartilhada com todos os clientes.

**Trade-off que motivou o descarte:** o raio de explosão de um vazamento seria total — e já houve caso de cliente vazando secret em log ([09:21] Sofia, [09:22] Diego). Detalhamento em [ADR-004](adrs/ADR-004-assinatura-hmac-sha256-secret-por-endpoint.md).

## 5. Questões em aberto

| # | Questão | Situação na reunião | Encaminhamento |
| --- | --- | --- | --- |
| RFC-QA-01 | **Rate limiting de envio por cliente** — se o cliente tiver 50 pedidos mudando de status em um minuto, ele recebe 50 chamadas? | Levantada por Diego; decidido não incluir no escopo ([09:38] Diego, [09:39] Larissa) | Observar em produção e implementar **se virar problema** ("observar e decidir depois") |
| RFC-QA-02 | **E-mail de aviso ao cliente quando o webhook dele está com problema** (ex.: falhou 3 vezes seguidas) | Levantada por Marcos; explicitamente adiada ([09:37] Marcos, [09:37] Larissa) | Fora do escopo desta fase; reavaliar como próxima fase, **depois de medir o impacto** |
| RFC-QA-03 | **Escalabilidade horizontal do worker** — como preservar ordering com múltiplos workers | Levantada por Bruno; adiada ([09:12] Bruno, [09:13] Diego) | Se necessário, particionar por `order_id` ou usar lock pessimista. "Problema do futuro" |
| RFC-QA-04 | **Arquivamento das linhas entregues da outbox** (após ~30 dias) | Mencionada por Diego como necessária; declarada fora do escopo ([09:08] Diego) | Sem rotina definida nesta fase; a outbox cresce indefinidamente até que seja endereçada |
| RFC-QA-05 | **Endurecimento de autorização no CRUD de configuração** | Levantada por Marcos e respondida como "por enquanto sim" por Sofia ([09:36] Marcos, [09:37] Sofia) | Hoje qualquer role autenticada; revisar "mais pra frente" |

As questões Q1, Q2, Q4 e Q5 são decisões **conscientes de não fazer agora**, não lacunas de análise. A Q3 é a única com gatilho técnico condicional (escala futura).

## 6. Impacto e riscos

### 6.1 Impacto

| ID | Dimensão | Impacto |
| --- | --- | --- |
| RFC-IMP-01 | Código existente | Alteração em um único ponto crítico (`changeStatus`), que passa a inserir na outbox dentro da transação. Nenhuma alteração é necessária no error middleware, no logger ou nos módulos existentes ([09:29] Bruno) |
| RFC-IMP-02 | Operação | Passa a existir um segundo processo a ser implantado e monitorado (o worker), além de tabelas novas ([09:11] Diego) |
| RFC-IMP-03 | Clientes | Passam a ter um endpoint HTTPS público para receber as chamadas, com verificação de assinatura e deduplicação por `event_id` ([09:25] Sofia) |
| RFC-IMP-04 | Prazo | Estimativa de **3 sprints** incluindo a revisão de segurança, com **pelo menos 2 dias úteis** reservados para Sofia antes do deploy ([09:46] Larissa, [09:46] Sofia) |

### 6.2 Riscos

| ID | Risco | Prob. | Impacto | Mitigação |
| --- | --- | --- | --- | --- |
| RFC-RISCO-01 | Transação do `changeStatus` fica mais longa, aumentando contenção | Média | Médio | Inserção única e indexada; monitorar tempo de transação |
| RFC-RISCO-02 | Worker é ponto único de falha (single-worker) | Média | Alto | Processo separado + monitoramento de pendentes acumulados; escalabilidade mapeada em RFC-QA-03 |
| RFC-RISCO-03 | Crescimento indefinido da outbox sem rotina de arquivamento | Alta | Médio | Índices em status e `created_at`; arquivamento mapeado em RFC-QA-04 |
| RFC-RISCO-04 | Cliente não implementa deduplicação por `event_id` e processa duplicatas | Média | Médio | Documentação destacada no portal do desenvolvedor ([09:26] Marcos) |
| RFC-RISCO-05 | Vazamento de secret do lado do cliente | Média | Alto | Secret por endpoint (raio limitado) + rotação com grace period de 24h ([ADR-004](adrs/ADR-004-assinatura-hmac-sha256-secret-por-endpoint.md)) |
| RFC-RISCO-06 | Entrega tardia em indisponibilidade longa do cliente | Baixa | Médio | Janela de retry de ~15h; DLQ com replay manual ([ADR-003](adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md)) |

## 7. Decisões relacionadas

Todas as decisões fechadas na reunião estão formalizadas nos ADRs:

- [ADR-001 — Padrão Outbox no MySQL](adrs/ADR-001-outbox-no-mysql.md)
- [ADR-002 — Worker em processo separado com polling de 2 segundos](adrs/ADR-002-worker-em-processo-separado-com-polling.md)
- [ADR-003 — Retry com backoff exponencial e DLQ](adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md)
- [ADR-004 — Assinatura HMAC-SHA256 com secret por endpoint](adrs/ADR-004-assinatura-hmac-sha256-secret-por-endpoint.md)
- [ADR-005 — Garantia at-least-once com `X-Event-Id`](adrs/ADR-005-garantia-at-least-once-com-x-event-id.md)
- [ADR-006 — Reuso dos padrões existentes](adrs/ADR-006-reuso-dos-padroes-existentes.md)
- [ADR-007 — Snapshot do payload na inserção](adrs/ADR-007-snapshot-do-payload-na-insercao.md)

## 8. Pedido aos revisores

Este RFC está aberto para revisão de Marcos, Bruno, Diego e Sofia. Os pontos onde a revisão é mais crítica:

1. **Diego** — validar o desenho do worker e da outbox (polling, batch, índices) e o comportamento de ordering sob falha.
2. **Bruno** — validar o ponto de acoplamento no `changeStatus` e a assinatura da função `publishWebhookEvent`.
3. **Sofia** — validar o desenho de HMAC, geração e rotação de secret, e o grace period de 24h. A revisão de segurança dedicada (2 dias úteis) está reservada antes do deploy ([09:46] Sofia).
4. **Marcos** — validar os requisitos funcionais cobertos e a comunicação ao cliente sobre at-least-once e deduplicação.

O detalhamento de implementação (contratos HTTP, matriz de erros, fluxos de retry, observabilidade) está no `docs/FDD.md` e não é objeto deste RFC.
