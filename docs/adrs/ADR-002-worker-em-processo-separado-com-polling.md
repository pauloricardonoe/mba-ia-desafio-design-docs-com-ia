# ADR-002 — Worker de entrega em processo separado com polling de 2 segundos

## Status

Aceito

## Data

2026-10-17 (decisão tomada na reunião técnica de quinta-feira, 09:00)

## Decisores

- Larissa (Tech Lead)
- Diego (Engenheiro Sênior, Plataforma)
- Bruno (Engenheiro Pleno, Pedidos)

## Contexto

Com o padrão Outbox definido em [ADR-001](ADR-001-outbox-no-mysql.md), é preciso decidir **como e onde** os eventos da tabela `webhook_outbox` são consumidos e transformados em chamadas HTTP.

Duas perguntas estavam em aberto:

1. **Como o worker descobre que há trabalho?** Reativamente (push) ou por varredura periódica (pull)?
2. **Onde o worker executa?** Dentro do processo da API ou como processo independente?

A primeira pergunta esbarra numa limitação do MySQL: não existe listener nativo equivalente ao `NOTIFY`/`LISTEN` do Postgres. Uma trigger no banco executa SQL, mas não notifica processos externos ([09:09] Diego). Já o requisito de negócio tolera latência de até 10 segundos ([09:02] Marcos).

A segunda pergunta é uma questão de resiliência: se o worker rodar dentro da instância da API, um restart da API derruba o worker junto ([09:11] Diego).

## Decisão

**Worker como processo Node separado, consumindo a outbox por polling em loop a cada 2 segundos.**

Propriedades acordadas:

- O worker roda em **processo próprio**, nunca embutido na instância da API. É uma nova entry-point no projeto, nos mesmos moldes de `src/server.ts` — um `src/worker.ts` e um script `npm run worker` ([09:11] Larissa).
- O worker conecta no **mesmo banco e usa a mesma stack**, mas instancia seu **próprio `PrismaClient`**, porque o `PrismaClient` é por processo. Mesma `DATABASE_URL`, instância nova ([09:11] Bruno, [09:30] Bruno).
- A cada ciclo, o worker busca os eventos **pendentes mais antigos** em batch pequeno, processa e marca como entregue ([09:08] Diego, [09:09] Diego).
- A lógica de processamento mora dentro do módulo de webhooks — `src/modules/webhooks/webhook.worker.ts` (ou `webhook.processor.ts`) — e não no `src/worker.ts`, que serve apenas como entry-point ([09:28] Bruno).
- **Latência aceita:** o pior caso é de 2 segundos entre a inserção do evento e o início do processamento. Registrado explicitamente como decisão por Larissa ([09:10] Larissa) e validado por Marcos ([09:10] Marcos).

## Alternativas consideradas

### 1. Worker embutido no processo da API

Iniciar o loop de polling dentro do mesmo processo do servidor Express, reaproveitando o `PrismaClient` já existente.

**Trade-off que motivou o descarte:** o ciclo de vida do worker ficaria amarrado ao da API. Um restart, deploy ou crash da API mataria o worker junto, interrompendo a entrega de notificações sem que houvesse um sinal claro disso ([09:11] Diego). O ganho de simplicidade (um processo a menos) não compensa a perda de isolamento de falha.

### 2. Consumo reativo via trigger do banco

Reagir à inserção na outbox por meio de uma trigger que notificasse o worker.

**Trade-off que motivou o descarte:** o MySQL não tem mecanismo nativo de notificação para processos externos. A trigger só executa SQL; para avisar o worker seria necessário improvisar um canal (arquivo, endpoint interno), o que foi considerado frágil e "esquisito" ([09:09] Diego). O polling de 2 segundos atende o requisito sem esse acoplamento artificial.

### 3. Múltiplos workers em paralelo

Escalar horizontalmente o consumo da outbox desde o início.

**Trade-off que motivou o descarte:** com múltiplos workers em paralelo, a ordenação de eventos por pedido se perde. A decisão foi manter **single-worker** nesta fase, com ordenação implícita por `order_id` via `created_at` da outbox. Escalar exigiria particionar por `order_id` ou usar lock pessimista — explicitamente adiado como "problema do futuro" ([09:12] Diego, [09:13] Diego).

## Consequências

### Positivas

- **Isolamento de ciclo de vida:** deploy ou restart da API não interrompe a entrega de webhooks, e vice-versa.
- **Ordenação garantida nesta fase:** com um único worker lendo em ordem de `created_at`, o cliente recebe os eventos de um pedido na ordem correta ([09:12] Diego).
- **Simplicidade de implementação e operação:** um loop de polling é trivial de entender, testar e depurar, sem broker ou infraestrutura adicional.
- **Escala de leitura controlada:** o batch pequeno somado aos índices da outbox evita varreduras caras ([09:08] Diego).

### Negativas

- **Latência mínima estrutural:** nenhum evento é entregue antes de ~2 segundos, mesmo quando o worker está ocioso. É uma latência imposta pelo desenho, não pelo cliente.
- **Custo de polling constante:** mesmo sem eventos pendentes, o worker executa uma query a cada 2 segundos indefinidamente.
- **Sem ordenação global e sem escalabilidade horizontal pronta:** a garantia de ordering é por `order_id` e **enquanto houver um único worker**; registrar como limitação conhecida ([09:13] Larissa).
- **Operação adicional:** passa a existir um segundo processo a ser implantado, monitorado e mantido vivo.

### Trade-off explícito

Aceita-se **latência mínima de 2 segundos** e **ausência de ordering global** em troca de **isolamento de falha entre API e worker**, **ordenação por pedido garantida nesta fase** e **zero infraestrutura adicional**. Como os clientes nunca pediram garantia de ordering global — "eles só querem saber se cada pedido deles mudou" ([09:14] Marcos) —, a limitação é aceitável.

## Referências

- Transcrição: [09:08] Diego, [09:09] Diego, [09:10] Larissa, [09:10] Marcos, [09:11] Larissa, [09:11] Bruno, [09:11] Diego, [09:12] Diego, [09:13] Larissa, [09:13] Diego, [09:14] Marcos, [09:28] Bruno, [09:30] Bruno
- Código: `src/server.ts` (modelo de entry-point), `src/config/database.ts` (`createPrismaClient`), `package.json` (scripts), `src/modules/orders/order.status.ts` (máquina de estados referenciada pelo fluxo)
