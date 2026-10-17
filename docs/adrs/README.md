# Architectural Decision Records

Este diretório armazena os ADRs (Architectural Decision Records) da feature **Sistema de Webhooks de Notificação de Pedidos**.

Cada arquivo registra uma decisão arquitetural fechada na reunião técnica de quinta-feira, 09:00, no formato MADR (Status, Contexto, Decisão, Alternativas Consideradas, Consequências).

## Índice

| ADR | Título | Decisão principal | Status |
| --- | --- | --- | --- |
| [ADR-001](ADR-001-outbox-no-mysql.md) | Padrão Outbox no MySQL para publicação de eventos de pedido | Padrão Outbox no MySQL | Aceito |
| [ADR-002](ADR-002-worker-em-processo-separado-com-polling.md) | Worker de entrega em processo separado com polling de 2 segundos | Worker em processo separado em polling | Aceito |
| [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md) | Retry com backoff exponencial (5 tentativas) e DLQ em tabela separada | Política de retry com backoff e DLQ | Aceito |
| [ADR-004](ADR-004-assinatura-hmac-sha256-secret-por-endpoint.md) | Assinatura HMAC-SHA256 com secret por endpoint e rotação com grace period | Autenticação HMAC-SHA256 com secret por endpoint | Aceito |
| [ADR-005](ADR-005-garantia-at-least-once-com-x-event-id.md) | Garantia at-least-once com `X-Event-Id` para deduplicação no cliente | Garantia at-least-once com `X-Event-Id` | Aceito |
| [ADR-006](ADR-006-reuso-dos-padroes-existentes.md) | Reuso máximo dos padrões existentes do projeto | Reuso dos padrões existentes do projeto | Aceito |
| [ADR-007](ADR-007-snapshot-do-payload-na-insercao.md) | Snapshot do payload renderizado no momento da inserção na outbox | Decisão secundária (formato do evento) | Aceito |

## Cobertura das decisões principais da reunião

As seis decisões principais discutidas estão cobertas:

| Decisão principal | ADR |
| --- | --- |
| Padrão Outbox no MySQL | ADR-001 |
| Política de retry com backoff e DLQ | ADR-003 |
| Autenticação HMAC-SHA256 com secret por endpoint | ADR-004 |
| Garantia at-least-once com `X-Event-Id` | ADR-005 |
| Worker em processo separado em polling | ADR-002 |
| Reuso dos padrões existentes do projeto | ADR-006 |

O ADR-007 cobre uma decisão secundária tomada no encerramento da reunião (formato de armazenamento do evento na outbox), que não estava na lista das seis decisões principais.

## Convenção de nomes

`ADR-NNN-titulo-em-kebab-case.md`, numeração sequencial e estável.
