# ADR-003 — Retry com backoff exponencial (5 tentativas) e DLQ em tabela separada

## Status

Aceito

## Data

2026-10-17 (decisão tomada na reunião técnica de quinta-feira, 09:00)

## Decisores

- Larissa (Tech Lead)
- Diego (Engenheiro Sênior, Plataforma)
- Bruno (Engenheiro Pleno, Pedidos)
- Marcos (Product Manager) — validou a janela de tolerância

## Contexto

O worker descrito em [ADR-002](ADR-002-worker-em-processo-separado-com-polling.md) executa chamadas HTTP para endpoints fora da nossa infraestrutura. Esses endpoints vão falhar: cliente em manutenção, indisponibilidade de rede, resposta de erro, timeout. É preciso decidir o que fazer quando a entrega falha.

Dois extremos foram colocados na mesa:

- **Retry indefinido com backoff:** o evento nunca é descartado, mas fica "pendurado" para sempre se o cliente desapareceu ([09:15] Diego).
- **Poucas tentativas:** agressivo demais — uma indisponibilidade matinal do cliente mataria o evento em poucos minutos ([09:16] Bruno, [09:16] Diego).

Havia evidência histórica relevante: a empresa já teve cliente com indisponibilidade de duas horas em manutenção planejada ([09:16] Diego). A janela de retry precisa cobrir esse tipo de cenário.

Também é preciso decidir o que acontece com o evento que esgota as tentativas: descartar, marcar como `failed` na própria outbox, ou isolá-lo em outro lugar.

## Decisão

**Retry com backoff exponencial, teto de 5 tentativas, e eventos esgotados movidos para uma tabela `webhook_dead_letter` separada.**

Propriedades acordadas:

- **5 tentativas** — descartado o retry indefinido e descartado o valor 3. Cinco cobre uma janela de até 12–24 horas ([09:15] Diego, [09:16] Diego, [09:16] Larissa).
- **Progressão do backoff: 1 minuto → 5 minutos → 30 minutos → 2 horas → 12 horas**, totalizando quase 15 horas entre a primeira falha e a última tentativa ([09:17] Diego).
- **Janela validada pelo negócio:** "se um cliente meu cair por 15 horas, ele já tá com problema sério dele. Acho aceitável" ([09:17] Marcos).
- **DLQ em tabela separada** `webhook_dead_letter`, contendo a payload, o motivo da falha e o timestamp. Motivo: mantém a leitura da outbox principal limpa e serve como evidência para debug e reprocessamento ([09:18] Diego).
- **Reprocessamento manual via endpoint administrativo:** `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente ([09:18] Diego, [09:35] Diego).
- O endpoint de replay **exige role `ADMIN`** e deve **registrar quem executou o replay**, para auditoria ([09:36] Sofia, [09:36] Larissa).

## Alternativas consideradas

### 1. Retry indefinido com backoff

Continuar tentando indefinidamente, aumentando o intervalo.

**Trade-off que motivou o descarte:** sem um teto, um evento de um cliente que simplesmente desapareceu ficaria pendurado para sempre, consumindo ciclos do worker e poluindo a outbox indefinidamente. Não há como distinguir "vai voltar" de "nunca vai voltar" ([09:15] Diego).

### 2. Teto de 3 tentativas

Ser mais agressivo, com menos tentativas e ciclo mais curto.

**Trade-off que motivou o descarte:** três tentativas com backoff curto seriam esgotadas em cerca de 30 minutos. Isso mataria eventos de clientes com indisponibilidades reais e não raras — a empresa já viu um cliente ficar duas horas fora em manutenção planejada. Três tentativas não cobrem nem esse caso ([09:16] Bruno, [09:16] Diego).

### 3. Marcar como `failed` na própria tabela de outbox

Em vez de uma tabela separada, apenas mudar o status do evento na `webhook_outbox` para `failed`.

**Trade-off que motivou o descarte:** o evento morto continuaria fisicamente na tabela quente que o worker varre a cada 2 segundos, misturando-se ao caminho crítico e degradando a leitura. Uma tabela separada isola o "lixo" e dá um lugar natural para armazenar motivo da falha e payload de evidência ([09:18] Diego).

## Consequências

### Positivas

- **Cobertura de janelas reais de indisponibilidade:** quase 15 horas de janela absorvem manutenções planejadas e incidentes de horas, que são o caso real observado ([09:16] Diego).
- **Nenhum evento fica pendurado para sempre:** o teto de tentativas garante que todo evento chega a um estado terminal (entregue ou na DLQ).
- **Debug e reprocessamento viáveis:** a DLQ guarda payload, motivo e timestamp, servindo de evidência ([09:18] Diego).
- **Outbox principal limpa:** a tabela varrida pelo worker não acumula eventos mortos.
- **Auditoria do replay:** o endpoint administrativo exige `ADMIN` e registra o autor ([09:36] Sofia).

### Negativas

- **Entrega tardia em caso de indisponibilidade longa:** um evento de um cliente que voltou após 6 horas só será entregue muitas horas depois da mudança de status, e o cliente terá ficado desatualizado todo esse tempo.
- **Reprocessamento é manual:** não há replay automático da DLQ. Um evento na DLQ fica parado até que alguém com role `ADMIN` o reprocesse explicitamente.
- **Mais uma tabela e mais um endpoint para manter:** cresce a superfície do módulo (schema, migration, rota administrativa, autorização).
- **Reprocessamento pode gerar duplicata:** recolocar o evento na outbox significa que o cliente pode recebê-lo mais de uma vez — comportamento já coberto pela garantia at-least-once (ver [ADR-005](ADR-005-garantia-at-least-once-com-x-event-id.md)).

### Trade-off explícito

Aceita-se **entrega tardia e reprocessamento manual** em troca de **não perder eventos de clientes com indisponibilidade real** e de **não deixar eventos pendurados indefinidamente**. O teto de 5 tentativas foi escolhido como o ponto que equilibra essas duas forças, com o trade-off explicitamente validado pelo PM ([09:17] Marcos).

## Referências

- Transcrição: [09:15] Diego, [09:15] Bruno, [09:16] Bruno, [09:16] Diego, [09:16] Larissa, [09:17] Diego, [09:17] Marcos, [09:17] Larissa, [09:18] Diego, [09:18] Bruno, [09:35] Diego, [09:36] Sofia, [09:36] Larissa
- Código: `src/middlewares/auth.middleware.ts` (`requireRole`), `src/modules/users/user.routes.ts` (precedente de rota com `requireRole('ADMIN')`), `src/middlewares/error.middleware.ts`, `prisma/schema.prisma`
