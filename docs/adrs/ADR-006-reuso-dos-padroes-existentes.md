# ADR-006 — Reuso máximo dos padrões existentes do projeto

## Status

Aceito

## Data

2026-10-17 (decisão tomada na reunião técnica de quinta-feira, 09:00)

## Decisores

- Larissa (Tech Lead) — fechou a decisão
- Bruno (Engenheiro Pleno, Pedidos) — mapeou os padrões da codebase
- Diego (Engenheiro Sênior, Plataforma)

## Contexto

A aplicação existente já tem convenções consolidadas. Bruno abriu o bloco de estrutura de código com uma constatação direta: "a gente tem um padrão claro na codebase. Cada domínio é um módulo em `src/modules` com controller, service, repository, routes e schemas" ([09:27] Bruno).

O risco de não seguir esses padrões é criar um módulo que "parece estrangeiro" no projeto: tratamento de erro diferente, logger diferente, estrutura de pastas diferente. Isso aumenta o custo de manutenção e o tempo de onboarding, e é especialmente ruim num time pequeno.

Além disso, havia uma decisão prática a tomar sobre infraestrutura compartilhada: o worker (processo separado, ver [ADR-002](ADR-002-worker-em-processo-separado-com-polling.md)) usa o mesmo `PrismaClient` da API ou uma instância própria?

## Decisão

**Reuso máximo dos padrões e componentes já existentes no projeto.** O módulo de webhooks é construído como mais um módulo igual aos outros, sem introduzir bibliotecas ou abstrações novas.

Componentes reaproveitados, conforme acordado ([09:30] Larissa):

| Padrão existente | Arquivo de referência | Como o módulo de webhooks usa |
| --- | --- | --- |
| Estrutura modular (`controller` / `service` / `repository` / `routes` / `schemas`) | `src/modules/orders/` | `src/modules/webhooks/` segue a mesma estrutura ([09:27] Bruno, [09:28] Bruno) |
| Hierarquia de erros (`AppError` e subclasses) | `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts` | Erros de webhook herdam das classes existentes, com códigos no prefixo `WEBHOOK_` ([09:28] Bruno) |
| Padrão de códigos de erro (`INSUFFICIENT_STOCK`, `INVALID_STATUS_TRANSITION`) | `src/shared/errors/http-errors.ts` | Códigos como `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED` ([09:28] Bruno, [09:29] Larissa) |
| Error middleware centralizado | `src/middlewares/error.middleware.ts` | Trata os erros do módulo sem nenhuma alteração — "vai pegar nossos erros sem precisar mudar nada" ([09:29] Bruno) |
| Logger Pino | `src/shared/logger/index.ts` | Nenhum logger novo é introduzido ([09:29] Bruno) |
| Schemas de validação Zod | `src/modules/orders/order.schemas.ts`, `src/middlewares/validate.middleware.ts` | Validações de payload, URL https e filtros de evento ([09:23] Sofia, [09:30] Larissa) |
| Middleware de autorização | `src/middlewares/auth.middleware.ts` (`requireRole`) | Endpoint de replay exige role `ADMIN` ([09:36] Larissa) |

Sobre infraestrutura de banco: o worker abre o **seu próprio `PrismaClient`**, porque o cliente Prisma é por processo, mas usa **mesmo banco e mesma `DATABASE_URL`** ([09:30] Bruno).

O `src/worker.ts` é uma **nova entry-point** no molde do `src/server.ts` existente ([09:11] Larissa).

## Alternativas consideradas

### 1. Módulo de webhooks como serviço/projeto separado

Extrair a feature para um serviço independente, com stack e convenções próprias.

**Trade-off que motivou o descarte:** implicaria subir e operar mais uma aplicação, com banco, deploy e monitoramento próprios, para um time pequeno — o mesmo argumento de *overengineering* usado para descartar o Redis Cluster em [ADR-001](ADR-001-outbox-no-mysql.md) ([09:07] Diego). Além disso, a integração crítica acontece **dentro da transação do `changeStatus`** ([09:40] Bruno), o que exige acesso ao mesmo banco e ao mesmo client de transação; um serviço separado tornaria essa atomicidade impossível.

### 2. Injetar o repository de webhooks inteiro no `OrderService`

Para permitir a inserção na outbox, passar o `WebhookRepository` como dependência do `OrderService`.

**Trade-off que motivou o descarte:** aumentaria o acoplamento entre os dois módulos e a superfície da API do `OrderService`. Preferiu-se uma **função pura** que recebe o client de transação — `publishWebhookEvent(tx, order, fromStatus, toStatus)` — mantendo o desacoplamento ([09:41] Bruno, [09:41] Diego).

### 3. Logger ou biblioteca de erros dedicada para o módulo

Introduzir uma biblioteca própria para o módulo de webhooks.

**Trade-off que motivou o descarte:** fragmentaria a observabilidade e o tratamento de erro do projeto. O Pino e o error middleware já cobrem o caso, e Larissa fixou a diretriz: "reuso máximo do que já existe" ([09:29] Bruno, [09:30] Larissa).

## Consequências

### Positivas

- **Consistência arquitetural:** o módulo de webhooks é indistinguível dos demais, reduzindo o custo cognitivo de quem navega o projeto.
- **Menos código novo:** tratamento de erro, logging, validação e autorização vêm de graça dos componentes existentes.
- **Onboarding e manutenção mais baratos:** qualquer pessoa familiarizada com `src/modules/orders` entende `src/modules/webhooks` ([09:27] Bruno).
- **Risco de regressão controlado:** o error middleware não precisa ser alterado, então os fluxos existentes não são tocados ([09:29] Bruno).

### Negativas

- **Acoplamento às convenções do projeto:** o módulo herda também as limitações desses padrões (por exemplo, o formato de erro e o formato de resposta paginada), mesmo onde um desenho específico de webhooks pudesse divergir.
- **Prefixação `WEBHOOK_` obrigatória em todos os códigos:** exige disciplina para que nenhum código escape do padrão ([09:29] Larissa).
- **O worker precisa duplicar a inicialização de conexão:** mesmo banco, mas uma instância de `PrismaClient` separada, com pool e ciclo de vida próprios ([09:30] Bruno).

### Trade-off explícito

Aceita-se **herdar as convenções e limitações do projeto** em troca de **consistência arquitetural**, **menos superfície de código novo** e **custo de manutenção menor num time pequeno**. O trade-off de abrir mão de um desenho dedicado é aceitável porque nenhum requisito da feature exige um padrão que os componentes existentes não atendam.

## Referências

- Transcrição: [09:07] Diego, [09:11] Larissa, [09:23] Sofia, [09:27] Bruno, [09:28] Bruno, [09:29] Bruno, [09:29] Larissa, [09:30] Bruno, [09:30] Larissa, [09:36] Larissa, [09:40] Bruno, [09:41] Bruno, [09:41] Diego
- Código: `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts`, `src/middlewares/error.middleware.ts`, `src/middlewares/auth.middleware.ts`, `src/middlewares/validate.middleware.ts`, `src/shared/logger/index.ts`, `src/modules/orders/order.schemas.ts`, `src/modules/orders/order.service.ts`, `src/config/database.ts`, `src/server.ts`, `src/modules/users/user.routes.ts`
