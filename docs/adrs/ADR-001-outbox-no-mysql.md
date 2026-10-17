# ADR-001 — Padrão Outbox no MySQL para publicação de eventos de pedido

## Status

Aceito

## Data

2026-10-17 (decisão tomada na reunião técnica de quinta-feira, 09:00)

## Decisores

- Larissa (Tech Lead) — conduziu e fechou a decisão
- Diego (Engenheiro Sênior, Plataforma) — propôs o padrão
- Bruno (Engenheiro Pleno, Pedidos) — levantou as restrições da transação atual

## Contexto

A feature de webhooks precisa entregar uma notificação HTTP ao cliente sempre que o status de um pedido muda. O ponto de origem desse evento é o método `changeStatus` do `OrderService`, que hoje executa uma transação com três efeitos: atualiza `orders.status`, insere uma linha em `order_status_history` e debita (ou repõe) `stock_quantity` dos produtos do pedido (`src/modules/orders/order.service.ts:126`).

A transação é pesada e crítica. Duas alternativas de acoplamento direto foram descartadas logo no início da discussão:

- **Envio síncrono dentro da transação** — um cliente HTTP lento travaria a mudança de status de outros pedidos, e um cliente fora do ar forçaria rollback da mudança de status, o que é inaceitável do ponto de vista de negócio ([09:04] Bruno).
- **Publicação fora da transação** (ex.: disparar o HTTP depois do commit) — quebraria a garantia de que "se o status mudou, o evento existe", criando janelas de inconsistência ([09:41] Diego).

A restrição de projeto é que o evento e a mudança de status sejam **atômicos**: se a transação commitou, o evento foi registrado; se ela deu rollback, o evento desaparece junto.

## Decisão

Adotar o **padrão Outbox persistido no MySQL já existente na aplicação**.

Quando o status do pedido muda, o `OrderService.changeStatus` insere uma linha na nova tabela `webhook_outbox` **dentro da mesma transação SQL** que atualiza `orders` e `order_status_history`. Um worker separado lê essa tabela e é o responsável por executar as chamadas HTTP.

Propriedades acordadas:

- A inserção na outbox participa da transação existente; se a inserção falhar, a transação inteira sofre rollback — não pode existir o caso de "status mudou e evento não saiu" ([09:40] Bruno).
- A tabela tem índice no campo de status (`pendente`, `processando`, `falhou`, `entregue`) e em `created_at`, para que o worker leia apenas os pendentes em batch pequeno ([09:08] Diego).
- O identificador da linha é **UUID**, seguindo o padrão de todo o restante do projeto, onde todas as chaves são `uuid @db.Char(36)` ([09:51] Larissa, `prisma/schema.prisma`).
- Linhas já entregues serão arquivadas depois de ~30 dias — **fora do escopo desta feature** ([09:08] Diego).

## Alternativas consideradas

### 1. Chamada HTTP síncrona dentro de `changeStatus`

Disparar o webhook diretamente no service, dentro da transação de mudança de status.

**Trade-off que motivou o descarte:** a transação de mudança de status já é pesada (update de order, insert em history, débito de estoque). Acrescentar um I/O de rede no meio faria um único cliente lento travar a mudança de status de **outros** pedidos, e um cliente indisponível levaria a um impasse insolúvel — não é aceitável dar rollback na mudança de status porque o cliente não respondeu ([09:04] Bruno, [09:04] Larissa).

### 2. Redis Streams / Redis Cluster

Usar um broker externo como mecanismo de fila, publicando o evento em um stream e consumindo com um worker.

**Trade-off que motivou o descarte:** exigiria subir infraestrutura nova (cluster Redis) para um time pequeno resolver um problema que o banco relacional já resolve. Diego classificou explicitamente como *overengineering* — "outbox no MySQL existente resolve" ([09:07] Diego, [09:07] Larissa).

### 3. Trigger de banco de dados

Usar uma trigger no MySQL para reagir à mudança de status.

**Trade-off que motivou o descarte:** o MySQL não tem mecanismo nativo de notificação a processos externos equivalente ao `NOTIFY`/`LISTEN` do Postgres. Uma trigger só executa SQL — para avisar o worker seria preciso improvisar (escrever em arquivo, bater em endpoint), o que foi avaliado como "esquisito". O polling de 2 segundos já atende o requisito de latência abaixo de 10 segundos ([09:09] Diego).

## Consequências

### Positivas

- **Atomicidade real:** o evento e a mudança de status compartilham a mesma transação, eliminando a possibilidade de inconsistência entre o estado do pedido e o evento publicado.
- **Zero infraestrutura nova:** reutiliza o MySQL e o `PrismaClient` já existentes (`src/config/database.ts`), alinhado à decisão de reuso máximo do projeto (ver [ADR-006](ADR-006-reuso-dos-padroes-existentes.md)).
- **Desacoplamento da latência do cliente:** a resposta do `PATCH /orders/:id/status` não depende de nenhum cliente externo responder.
- **Simplicidade operacional:** a "fila" é inspecionável via SQL comum, o que facilita debug e suporte.

### Negativas

- **Latência mínima de entrega:** o evento só é enviado no próximo ciclo de polling, o que impõe um piso de ~2 segundos entre a mudança de status e o envio (ver [ADR-002](ADR-002-worker-em-processo-separado-com-polling.md)).
- **Crescimento da tabela:** a outbox acumula linhas indefinidamente até que a rotina de arquivamento (30 dias) seja implementada — adiada para uma fase futura.
- **Acoplamento do módulo de webhooks à transação de orders:** o `changeStatus` passa a depender da outbox, aumentando o tempo de vida da transação e o raio de impacto de uma falha no módulo de webhooks.
- **Novo ponto de falha operacional:** se o worker não estiver rodando, os eventos acumulam como pendentes sem que a API sinalize problema.

### Trade-off explícito

Aceita-se **latência mínima de ~2 segundos** e o **crescimento da tabela até que o arquivamento seja implementado**, em troca de **consistência forte entre status e evento** e **nenhuma infraestrutura adicional**. O requisito de negócio tolera até 10 segundos ("qualquer coisa abaixo de 10 segundos já é tempo real" — [09:02] Marcos), o que dá margem confortável para o polling.

## Referências

- Transcrição: [09:04] Bruno, [09:06] Diego, [09:07] Larissa, [09:07] Diego, [09:08] Diego, [09:08] Larissa, [09:40] Bruno, [09:41] Diego, [09:51] Larissa
- Código: `src/modules/orders/order.service.ts` (`changeStatus`), `src/config/database.ts`, `prisma/schema.prisma`
