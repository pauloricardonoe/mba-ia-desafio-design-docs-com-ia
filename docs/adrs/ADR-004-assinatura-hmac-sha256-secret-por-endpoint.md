# ADR-004 — Assinatura HMAC-SHA256 com secret por endpoint e rotação com grace period

## Status

Aceito

## Data

2026-10-17 (decisão tomada na reunião técnica de quinta-feira, 09:00)

## Decisores

- Sofia (Engenheira de Segurança) — propôs e conduziu
- Larissa (Tech Lead)
- Bruno (Engenheiro Pleno, Pedidos)
- Diego (Engenheiro Sênior, Plataforma)

## Contexto

A feature expõe eventos contendo dados de pedidos para endpoints que estão **fora da nossa infraestrutura**, controlados pelos clientes. Isso cria duas necessidades de segurança levantadas por Sofia no início do bloco de segurança:

1. O cliente precisa conseguir **validar que a requisição veio realmente de nós** — não de um terceiro se passando pela plataforma.
2. O cliente precisa conseguir validar que **ninguém adulterou o payload** no caminho.

Além disso, a gestão do segredo compartilhado tem que ser pensada desde o início: uma secret comprometida não pode comprometer todos os clientes, e um cliente precisa conseguir trocar de secret sem parar a integração dele.

## Decisão

**Assinatura HMAC-SHA256 sobre o corpo do request, com secret única por endpoint de webhook, e rotação com grace period de 24 horas.**

Propriedades acordadas:

- **Algoritmo HMAC-SHA256.** Escolhido por ser o padrão de mercado — "todo cliente sério tem biblioteca pra isso" ([09:20] Sofia).
- **A assinatura vai no header `X-Signature`**, calculada sobre o payload, e o cliente a verifica do lado dele ([09:20] Sofia).
- **Secret por endpoint, nunca global.** Cada cadastro de webhook tem sua própria secret. O motivo é explícito: "não é uma secret global da nossa plataforma. Senão se vaza uma, vaza tudo" ([09:21] Sofia).
- **A secret é gerada pela plataforma** e devolvida ao cliente no momento da criação do webhook ([09:31] Marcos).
- **A secret é rotacionável via API.** O cliente pode pedir uma nova secret por endpoint; durante **24 horas** a secret antiga permanece válida em paralelo, para dar tempo de migrar os sistemas do cliente. Passado o grace period, a antiga morre ([09:21] Sofia).
- **TLS obrigatório.** A URL do webhook precisa ser `https`. Cadastro com `http` é recusado com erro de validação — implementado como validação no schema Zod, não como decisão arquitetural separada ([09:23] Sofia).
- **Limite de tamanho de payload de 64KB, com erro acima disso.** Não se trunca: "se chegou nesse tamanho, tem algo errado". Registrado como requisito não funcional ([09:23] Sofia, [09:24] Diego, [09:24] Larissa).

O contexto do schema de configuração de webhook — `url` + `secret` + `customer_id` + estado ativo — foi confirmado por Bruno e Sofia ([09:21] Bruno, [09:21] Sofia).

A rotação existe porque há precedente real do problema: "a gente já teve cliente que vazou secret em log de aplicação dele uma vez" ([09:22] Diego).

## Alternativas consideradas

### 1. Secret global da plataforma

Uma única secret compartilhada entre a plataforma e todos os clientes, usada para assinar todos os webhooks.

**Trade-off que motivou o descarte:** o raio de explosão de um vazamento seria total. Se a secret vazasse (e já houve caso de cliente vazando secret em log — [09:22] Diego), **todos** os clientes ficariam expostos e a rotação exigiria coordenar a troca com toda a base simultaneamente. A secret por endpoint isola o comprometimento a um único cliente ([09:21] Sofia).

### 2. Rotação imediata da secret (sem grace period)

Ao rotacionar, invalidar a secret antiga na hora.

**Trade-off que motivou o descarte:** quebraria a integração do cliente no instante da rotação. O cliente tem sistemas próprios que precisam ser atualizados com a nova secret; sem uma janela de convivência, a rotação causaria indisponibilidade de entrega do lado dele. O grace period de 24 horas resolve isso permitindo que as duas secrets sejam válidas em paralelo durante a transição ([09:21] Sofia).

## Consequências

### Positivas

- **Autenticidade e integridade verificáveis pelo cliente:** o HMAC-SHA256 permite ao cliente confirmar origem e detectar adulteração do payload ([09:19] Sofia).
- **Raio de explosão limitado:** um vazamento de secret compromete apenas o endpoint daquele cliente.
- **Rotação sem downtime para o cliente:** o grace period de 24h permite migração sem interrupção da integração ([09:21] Sofia).
- **Padrão de mercado:** HMAC-SHA256 tem suporte em praticamente qualquer stack, reduzindo o esforço de integração do cliente ([09:20] Sofia).
- **Validação de transporte desde o cadastro:** a obrigatoriedade de `https` impede configuração insegura na origem ([09:23] Sofia).

### Negativas

- **Segredo compartilhado exige disciplina dos dois lados:** a segurança depende de o cliente guardar bem a secret. Já houve vazamento do lado do cliente em log de aplicação ([09:22] Diego).
- **Complexidade de armazenamento e ciclo de vida:** é preciso armazenar secret atual, secret anterior e a janela de validade, além de implementar a lógica de rotação e expiração.
- **Janela de 24h é um compromisso:** durante o grace period, duas secrets válidas coexistem — quem obtiver a antiga ainda consegue assinar por até 24 horas.
- **Validação HMAC e geração de secret exigem revisão de segurança dedicada:** Sofia reservou explicitamente pelo menos dois dias úteis para revisar esse código antes do deploy ([09:46] Sofia).

### Trade-off explícito

Aceita-se **segredo compartilhado com o cliente** (e a dependência de que ele o guarde bem) e a **complexidade de gerenciar duas secrets durante a rotação**, em troca de **autenticidade verificável pelo cliente**, **raio de explosão limitado por endpoint** e **rotação sem downtime**. A alternativa de exactly-once de autenticação (ex.: mTLS) não foi discutida na reunião e não é considerada aqui.

## Referências

- Transcrição: [09:19] Sofia, [09:20] Sofia, [09:20] Bruno, [09:21] Sofia, [09:21] Bruno, [09:22] Sofia, [09:22] Diego, [09:23] Sofia, [09:24] Diego, [09:24] Larissa, [09:31] Marcos, [09:46] Sofia
- Código: `src/modules/orders/order.schemas.ts` (padrão de schemas Zod para validação), `src/middlewares/validate.middleware.ts` (ponto de aplicação das validações), `src/shared/logger/index.ts` (redact de campos sensíveis), `prisma/schema.prisma` (modelo `Customer`, chaves UUID)
