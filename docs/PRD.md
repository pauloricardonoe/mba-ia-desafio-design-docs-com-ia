# PRD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| **Produto** | Order Management System (OMS) |
| **Feature** | Notificação de mudança de status de pedidos via webhooks (outbound) |
| **Status** | Aprovado para desenvolvimento |
| **Data** | 2026-10-17 |
| **Origem** | Reunião técnica de quinta-feira, 09:00 — ver `TRANSCRICAO.md` |
| **Documentos relacionados** | `docs/RFC.md`, `docs/FDD.md`, `docs/adrs/` |

---

## 1. Resumo e contexto da feature

O OMS passa a notificar seus clientes B2B, por HTTP, sempre que o status de um pedido muda. Em vez de o cliente consultar periodicamente o `GET /orders` para descobrir se algo mudou, a plataforma envia o evento ativamente para um endpoint HTTPS cadastrado pelo próprio cliente.

A feature é **estritamente outbound**: a plataforma envia, os clientes recebem ([09:02] Marcos, [09:02] Sofia).

O contexto de negócio é uma demanda formal de três clientes B2B — **Atlas Comercial, MaxDistribuição e Nova Cargo** —, que hoje fazem polling e consideram a integração lenta e cara do lado deles. A Atlas sinalizou que, sem a entrega até o fim do trimestre, pode migrar para o concorrente ([09:00] Marcos).

Do ponto de vista de produto, a feature é o **primeiro mecanismo de notificação externa** do OMS: a aplicação não possui hoje eventos, filas ou webhooks.

## 2. Problema e motivação

### 2.1 Problema

Clientes B2B precisam saber quando o status dos pedidos deles muda, mas hoje só conseguem descobrir isso **consultando ativamente** o `GET /orders` em intervalos regulares. Isso tem três consequências:

1. **Integração lenta** — a informação só chega ao cliente no próximo ciclo de polling dele.
2. **Integração cara** — o cliente paga o custo de consultar repetidamente uma API que, na maioria das vezes, não tem novidade.
3. **Acoplamento manual** — a experiência descrita pelo PM é a de uma integração que "fica pendurada" e exige atualização manual ([09:02] Marcos).

### 2.2 Motivação

- **Risco de churn:** a Atlas condicionou a permanência à entrega da feature até o fim do trimestre ([09:00] Marcos).
- **Demanda explícita e conjunta** de três clientes, não de um caso isolado ([09:00] Marcos).
- **A lacuna é conhecida:** a aplicação não tem nenhum mecanismo de notificação, e essa ausência é exatamente o que a feature preenche.

### 2.3 Definição de "tempo real" para o cliente

Ponto crítico de alinhamento, validado diretamente com o cliente: **qualquer latência abaixo de 10 segundos já é considerada "tempo real"**. O que importa é que a notificação não fique pendurada e que o cliente não precise atualizar manualmente ([09:02] Marcos). Essa definição é a âncora de todos os objetivos de desempenho deste PRD.

## 3. Público-alvo e cenários de uso

### 3.1 Público-alvo

| Perfil | Descrição | Necessidade |
| --- | --- | --- |
| **Cliente B2B integrado** | Empresas como Atlas Comercial, MaxDistribuição e Nova Cargo | Receber notificações automáticas de mudança de status dos seus pedidos |
| **Usuário operador do cliente** | Usuário autenticado no nosso sistema que representa o cliente | Cadastrar e gerenciar os webhooks do cliente via API |
| **Administrador da plataforma** | Usuário com role `ADMIN` | Reprocessar manualmente eventos que falharam definitivamente |
| **Time de suporte / engenharia da plataforma** | Interno | Diagnosticar falhas de entrega e reprocessar eventos |

> O cadastro é feito **pela nossa API**, autenticado com o JWT do nosso sistema — a plataforma tem usuários que representam o cliente ([09:32] Marcos).

### 3.2 Cenários de uso

**Cenário 1 — Cadastro e primeira notificação.**
Um usuário do cliente Atlas cadastra um webhook apontando para `https://integracao.atlas.com.br/...`, informando que quer ser notificado quando os pedidos virarem `SHIPPED` e `DELIVERED`. A plataforma gera e devolve a secret. Quando um pedido da Atlas passa para `SHIPPED`, a plataforma envia a notificação em poucos segundos, e o sistema da Atlas valida a assinatura e processa o evento.

**Cenário 2 — Cliente com indisponibilidade temporária.**
O endpoint do cliente está em manutenção planejada. A primeira tentativa de entrega falha; a plataforma reagenda e tenta novamente com intervalos crescentes. Quando o cliente volta, ele recebe o evento — possivelmente horas depois, mas sem perda.

**Cenário 3 — Cliente indisponível por tempo demais.**
O cliente fica fora por mais de ~15 horas. Após a quinta tentativa, o evento é movido para a fila de falhas definitivas. Um administrador reprocessa manualmente, e o evento é reenviado.

**Cenário 4 — Cliente com múltiplos webhooks.**
O cliente tem cadastros diferentes para times diferentes. Ao receber uma notificação, ele identifica qual cadastro disparou pelo header `X-Webhook-Id` ([09:44] Sofia).

**Cenário 5 — Cliente audita o que recebeu.**
O cliente consulta o histórico das últimas 100 entregas do webhook dele, com sucesso/falha, payload, resposta e tempo de resposta, para investigar uma integração ([09:34] Marcos).

**Cenário 6 — Rotação de secret.**
O cliente suspeita que sua secret vazou. Ele solicita uma nova secret; a antiga continua válida por 24 horas, dando tempo para atualizar os sistemas dele antes que ela expire ([09:21] Sofia, [09:22] Diego).

## 4. Objetivos e métricas de sucesso

| ID | Objetivo | Métrica | Meta | Origem |
| --- | --- | --- | --- | --- |
| PRD-OBJ-01 | Notificar mudanças de status em tempo real na percepção do cliente | Tempo entre a mudança de status e a entrega bem-sucedida | **Abaixo de 10 segundos** no caso feliz | [09:02] Marcos |
| PRD-OBJ-02 | Integrar os três clientes B2B demandantes | Nº de clientes B2B integrados e recebendo notificações | **3** (Atlas, MaxDistribuição, Nova Cargo) | [09:00] Marcos |
| PRD-OBJ-03 | Não perder eventos de clientes com indisponibilidade real | Janela de retry automático antes da fila de falhas definitivas | **~15 horas** (5 tentativas) | [09:17] Diego |
| PRD-OBJ-04 | Entregar a feature dentro da janela de risco comercial | Prazo de desenvolvimento, incluindo revisão de segurança | **3 sprints** + 2 dias úteis de revisão de segurança | [09:46] Larissa, [09:46] Sofia |
| PRD-OBJ-05 | Reduzir a dependência de polling manual pelos clientes | Clientes migrados do polling no `GET /orders` para webhooks | Adesão dos 3 clientes demandantes | [09:00] Marcos, [09:02] Marcos |
| PRD-OBJ-06 | Dar visibilidade ao cliente sobre a própria integração | Histórico de entregas consultável | **Últimas 100 entregas** por webhook | [09:34] Marcos |

O objetivo **PRD-OBJ-01** é o primário: é a definição de sucesso que o próprio cliente forneceu, e é o que sustenta a decisão de permanência da Atlas.

## 5. Escopo

### 5.1 Incluso

- Cadastro, edição, remoção e listagem de webhooks por cliente.
- Seleção, por webhook, de quais status de pedido o cliente quer receber.
- Envio automático de notificação HTTP quando o status de um pedido muda.
- Assinatura das notificações para o cliente validar autenticidade e integridade.
- Rotação de secret com janela de convivência de 24 horas.
- Reenvio automático em caso de falha, com intervalos crescentes e limite de tentativas.
- Isolamento de eventos que esgotaram as tentativas em uma fila de falhas definitivas.
- Reprocessamento manual de eventos da fila de falhas, restrito a administradores.
- Histórico de entregas consultável pelo cliente.
- Identificação de deduplicação para o cliente (entrega at-least-once).
- Exigência de HTTPS nos endpoints cadastrados.

### 5.2 Fora de escopo

Itens **explicitamente descartados ou adiados** na reunião:

| ID | Item | Decisão | Origem |
| --- | --- | --- | --- |
| **PRD-ESCOPO-01** | **Aviso por e-mail ao cliente quando o webhook falha repetidamente** (ex.: 3 falhas seguidas) | **Adiado para uma próxima fase.** "Email tá fora de escopo dessa fase. Talvez próxima fase, depois que a gente medir o impacto" | [09:37] Marcos, [09:37] Larissa |
| **PRD-ESCOPO-02** | **Dashboard visual / painel do cliente** | **Descartado desta entrega.** "Só endpoints. Painel é projeto separado do time de frontend" | [09:39] Marcos, [09:40] Larissa |
| **PRD-ESCOPO-03** | **Rate limiting de envio por cliente** | **Adiado.** "A gente observa e implementa se virar problema" | [09:38] Diego, [09:39] Larissa |
| **PRD-ESCOPO-04** | **Webhooks inbound** (cliente → plataforma) | **Descartado.** O escopo é apenas outbound | [09:02] Sofia, [09:02] Marcos |
| **PRD-ESCOPO-05** | **Arquivamento das linhas já entregues da fila de eventos** (após ~30 dias) | **Fora do escopo desta feature** | [09:08] Diego |
| **PRD-ESCOPO-06** | **Escala horizontal do worker / ordering com múltiplos workers** | **Adiado.** "Problema do futuro" | [09:13] Bruno, [09:13] Diego |

## 6. Requisitos funcionais

| ID | Requisito | Origem |
| --- | --- | --- |
| **PRD-FR-01** | O cliente pode **cadastrar** um webhook informando a URL de destino e a lista de status que deseja receber. A secret é **gerada pela plataforma** e devolvida na criação. | [09:31] Marcos |
| **PRD-FR-02** | O cliente pode **editar** um webhook existente (URL, lista de status assinados, estado ativo). | [09:33] Bruno |
| **PRD-FR-03** | O cliente pode **remover** um webhook. | [09:33] Bruno |
| **PRD-FR-04** | O cliente pode **listar** os webhooks de um determinado cliente. | [09:33] Bruno |
| **PRD-FR-05** | Cada webhook pode escolher **quais status deseja receber**. O filtro é aplicado **na inserção do evento**, de modo que nenhum evento é registrado se nenhum webhook do cliente assina aquele status. | [09:33] Marcos, [09:34] Bruno, [09:34] Diego |
| **PRD-FR-06** | Quando o status de um pedido muda, a plataforma **notifica automaticamente** os webhooks ativos do cliente que assinam aquele status, via requisição HTTP `POST`. | [09:00] Marcos, [09:06] Diego |
| **PRD-FR-07** | O cliente pode consultar o **histórico de entregas** de um webhook, com os **últimos 100** envios, incluindo sucesso/falha, payload, resposta e tempo de resposta. | [09:34] Marcos |
| **PRD-FR-08** | Um administrador pode **reprocessar manualmente** um evento da fila de falhas definitivas, recolocando-o na fila de envio. O acesso é **restrito à role `ADMIN`**. | [09:18] Diego, [09:35] Diego, [09:36] Larissa |
| **PRD-FR-09** | O cliente pode **rotacionar a secret** de um webhook. A secret anterior permanece válida por **24 horas** em paralelo, e depois é invalidada. | [09:21] Sofia |
| **PRD-FR-10** | Toda notificação é **assinada** com HMAC-SHA256 sobre o corpo da requisição, usando a secret do endpoint, e a assinatura é enviada em um header para validação do cliente. | [09:20] Sofia, [09:22] Sofia |
| **PRD-FR-11** | Toda notificação carrega um **identificador único de evento**, permitindo ao cliente **deduplicar** entregas repetidas. | [09:25] Diego |
| **PRD-FR-12** | Em caso de falha de entrega, a plataforma **reenvia automaticamente** o evento com intervalos crescentes, até o limite de tentativas. | [09:15] Diego |
| **PRD-FR-13** | Esgotado o limite de tentativas, o evento é **isolado em uma fila de falhas definitivas**, preservando payload e motivo da falha para diagnóstico e reprocessamento. | [09:18] Diego |
| **PRD-FR-14** | O reprocessamento manual de um evento **registra quem o executou**, para fins de auditoria. | [09:36] Sofia |
| **PRD-FR-15** | O cliente identifica qual dos seus cadastros de webhook originou a notificação recebida. | [09:44] Sofia |
| **PRD-FR-16** | O cliente consegue validar que a notificação **não foi adulterada** no trajeto. | [09:19] Sofia |

## 7. Requisitos não funcionais

| ID | Requisito | Valor | Origem |
| --- | --- | --- | --- |
| **PRD-NFR-01** | Latência de entrega no caso feliz | **Abaixo de 10 segundos** | [09:02] Marcos |
| **PRD-NFR-02** | Cadência de verificação de eventos pendentes | A cada **2 segundos** | [09:09] Diego, [09:10] Larissa |
| **PRD-NFR-03** | Número de tentativas de entrega | **5** | [09:15] Diego, [09:16] Larissa |
| **PRD-NFR-04** | Progressão do intervalo entre tentativas | **1m / 5m / 30m / 2h / 12h** | [09:17] Diego |
| **PRD-NFR-05** | Janela total de retry | **~15 horas** | [09:17] Diego |
| **PRD-NFR-06** | Timeout da chamada ao cliente | **10 segundos**; estouro é tratado como falha | [09:42] Diego |
| **PRD-NFR-07** | Tamanho máximo do payload | **64 KB**; acima disso, erro (não truncar) | [09:23] Sofia, [09:24] Diego |
| **PRD-NFR-08** | Transporte obrigatório | **HTTPS**; URLs `http` são recusadas no cadastro | [09:23] Sofia |
| **PRD-NFR-09** | Escopo da secret | **Uma secret por endpoint de webhook**, nunca global | [09:21] Sofia |
| **PRD-NFR-10** | Garantia de entrega | **At-least-once** (duplicatas possíveis; dedup pelo cliente) | [09:24] Diego |
| **PRD-NFR-11** | Ordenação de eventos | **Por `order_id`**, garantida apenas com um único worker — não é ordering global | [09:12] Diego, [09:13] Larissa |
| **PRD-NFR-12** | Isolamento do processo de entrega | Worker roda em **processo separado** da API | [09:11] Diego |
| **PRD-NFR-13** | Consistência de padrões | Reuso máximo do que já existe no projeto (erros, logger, middleware, módulos, validações) | [09:30] Larissa |
| **PRD-NFR-14** | Revisão de segurança antes do deploy | **Pelo menos 2 dias úteis** dedicados à revisão de HMAC e geração de secret | [09:46] Sofia |
| **PRD-NFR-15** | Ausência de infraestrutura nova | Sem broker, cache ou serviço adicional | [09:07] Diego |

## 8. Decisões e trade-offs principais

As decisões completas estão nos ADRs; esta é a leitura de produto.

| Decisão | Trade-off aceito |
| --- | --- |
| **Padrão Outbox no MySQL** ([ADR-001](adrs/ADR-001-outbox-no-mysql.md)) | Aceita-se latência mínima de ~2s e crescimento da tabela em troca de consistência forte entre status e evento, sem infra nova |
| **Worker separado com polling de 2s** ([ADR-002](adrs/ADR-002-worker-em-processo-separado-com-polling.md)) | Aceita-se latência mínima estrutural e ausência de ordering global em troca de isolamento de falha e ordering por pedido |
| **5 tentativas com backoff e fila de falhas** ([ADR-003](adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md)) | Aceita-se entrega tardia e reprocessamento manual em troca de não perder eventos de clientes com indisponibilidade real |
| **HMAC-SHA256 com secret por endpoint** ([ADR-004](adrs/ADR-004-assinatura-hmac-sha256-secret-por-endpoint.md)) | Aceita-se depender de o cliente guardar bem a secret em troca de raio de explosão limitado e rotação sem downtime |
| **At-least-once com identificador de evento** ([ADR-005](adrs/ADR-005-garantia-at-least-once-com-x-event-id.md)) | Aceita-se transferir a responsabilidade de deduplicação ao cliente em troca de simplicidade e alinhamento com o padrão de mercado |
| **Reuso dos padrões existentes** ([ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md)) | Aceita-se herdar as convenções do projeto em troca de consistência e menor custo de manutenção |
| **Payload como snapshot na criação do evento** ([ADR-007](adrs/ADR-007-snapshot-do-payload-na-insercao.md)) | Aceita-se duplicar o payload no banco em troca de um evento que reflete o pedido no momento da mudança de status |

## 9. Dependências

| ID | Dependência | Tipo | Observação |
| --- | --- | --- | --- |
| PRD-DEP-01 | Três clientes B2B demandantes | Externa | Atlas Comercial, MaxDistribuição e Nova Cargo — definem a urgência ([09:00] Marcos) |
| PRD-DEP-02 | Endpoints HTTPS operados pelos clientes | Externa | Sem um endpoint válido do lado do cliente, a entrega falha e cai no retry |
| PRD-DEP-03 | Capacidade do cliente de implementar verificação de assinatura e deduplicação | Externa | A garantia at-least-once só funciona se o cliente deduplicar ([09:25] Sofia) |
| PRD-DEP-04 | Documentação no portal do desenvolvedor | Interna | Marcos se comprometeu a documentar a integração e o comportamento at-least-once ([09:26] Marcos, [09:40] Marcos) |
| PRD-DEP-05 | Revisão de segurança de Sofia | Interna | Bloqueia o deploy; 2 dias úteis reservados ([09:46] Sofia) |
| PRD-DEP-06 | Banco MySQL e stack existentes | Técnica | Reusados integralmente, sem infra nova ([09:07] Diego) |

## 10. Riscos e mitigação

| # | Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- | --- |
| **PRD-RISCO-01** | **Perda do cliente Atlas** por não entregar a feature até o fim do trimestre | Média | **Alto** | Prazo de 3 sprints com revisão de segurança incluída; Marcos confirma o prazo com o cliente ([09:46] Larissa, [09:47] Marcos) |
| **PRD-RISCO-02** | **Cliente não implementa deduplicação** e processa o mesmo evento mais de uma vez | Média | Médio | Documentar o comportamento at-least-once e o uso do identificador de evento em destaque no portal do desenvolvedor ([09:26] Marcos) |
| **PRD-RISCO-03** | **Cliente indisponível por mais de ~15 horas**: o evento cai na fila de falhas e fica parado, sem aviso automático | Baixa | Médio | Evento preservado com payload e motivo para replay manual; aviso por e-mail mapeado como próxima fase ([09:37] Larissa) |
| **PRD-RISCO-04** | **Vazamento de secret** do lado do cliente (já ocorreu no histórico da empresa) | Média | **Alto** | Secret por endpoint (limita o raio do vazamento) e rotação com 24h de convivência ([09:22] Diego, [09:21] Sofia) |
| **PRD-RISCO-05** | **Acúmulo de eventos na fila** sem rotina de arquivamento, degradando a performance da entrega | Média | Médio | Índices dedicados e leitura em lotes pequenos; arquivamento reconhecido como fora de escopo, a reavaliar ([09:08] Diego) |
| **PRD-RISCO-06** | **Atraso no cronograma** por conta da revisão de segurança | Baixa | Médio | Revisão já embutida na estimativa de 3 sprints, não tratada como atividade paralela ([09:46] Larissa) |

## 11. Critérios de aceitação

Critérios verificáveis de produto, na perspectiva de quem valida a entrega:

| # | Critério |
| --- | --- |
| PRD-CA-01 | Um cliente consegue cadastrar, editar, listar e remover webhooks pela API |
| PRD-CA-02 | A secret é gerada pela plataforma e devolvida no cadastro |
| PRD-CA-03 | O cliente escolhe quais status deseja receber, por webhook |
| PRD-CA-04 | Nenhum evento é registrado quando nenhum webhook do cliente assina aquele status |
| PRD-CA-05 | Uma mudança de status gera notificação entregue em menos de 10 segundos no caso feliz |
| PRD-CA-06 | A notificação chega assinada e o cliente consegue validar a assinatura |
| PRD-CA-07 | A notificação carrega um identificador que permite deduplicação |
| PRD-CA-08 | O cliente identifica qual cadastro de webhook originou a notificação |
| PRD-CA-09 | Uma falha de entrega é reenviada automaticamente conforme a progressão acordada |
| PRD-CA-10 | Após 5 tentativas, o evento vai para a fila de falhas definitivas com payload e motivo |
| PRD-CA-11 | Um administrador consegue reprocessar um evento da fila de falhas, e a ação é auditada |
| PRD-CA-12 | Um usuário sem role `ADMIN` **não consegue** reprocessar eventos |
| PRD-CA-13 | O cliente consegue rotacionar a secret e a anterior continua válida por 24 horas |
| PRD-CA-14 | URLs `http` são recusadas no cadastro |
| PRD-CA-15 | Um payload acima de 64 KB é rejeitado, não truncado |
| PRD-CA-16 | O cliente consegue consultar as últimas 100 entregas de um webhook |
| PRD-CA-17 | Eventos de um mesmo pedido são entregues em ordem |
| PRD-CA-18 | O serviço de entrega continua funcionando após um restart da API |

## 12. Estratégia de testes e validação

### 12.1 Abordagem

A validação combina testes automatizados (o projeto já usa **Vitest** e **Supertest**, com factories em `tests/helpers/factories.ts`) e uma revisão de segurança dedicada.

| Camada | O que valida | Como |
| --- | --- | --- |
| **Unitário** | Máquina de estados, filtro de eventos por status, cálculo do backoff, geração e verificação de assinatura | Testes isolados das funções puras |
| **Integração — atomicidade** | FR-06 / CA-04 / CA-05 do FDD: um rollback não deixa evento na fila; um commit deixa exatamente um por webhook elegível | Teste transacional sobre o `changeStatus`, seguindo o padrão de `tests/orders.test.ts` |
| **Integração — CRUD** | Cadastro, edição, listagem, remoção, rotação de secret e histórico de entregas | Testes de API com Supertest, autenticados via `bootstrapAuthenticatedUser` |
| **Integração — autorização** | FR-08 / CA-12: replay exige `ADMIN` | Teste com usuário `OPERATOR` esperando `403`, no padrão de `tests/auth.test.ts` |
| **Worker** | Retry, progressão do backoff, timeout de 10s, movimento para a fila de falhas | Teste do processador com cliente HTTP simulado (sucesso, erro, timeout) |
| **Contrato de assinatura** | FR-10 / CA-06: a assinatura enviada é validável com a secret do endpoint, inclusive durante o grace period | Teste de verificação independente do HMAC |
| **Ponta a ponta** | Mudança de status → evento na fila → entrega assinada no endpoint simulado | Fluxo completo com um receptor HTTP de teste |
| **Validação de negócio** | CA-05 (latência < 10s) e CA-17 (ordering) em ambiente integrado | Medição com carga controlada e múltiplas transições do mesmo pedido |
| **Segurança** | HMAC, geração e rotação de secret, ausência de vazamento de secret em logs | Revisão manual dedicada de Sofia, 2 dias úteis antes do deploy ([09:46] Sofia) |

### 12.2 Validação com os clientes

A feature só é considerada bem-sucedida quando os três clientes demandantes estiverem efetivamente integrados e recebendo notificações, o que valida o objetivo **O2**. Marcos fica responsável por comunicar o prazo e o comportamento de integração aos clientes ([09:47] Marcos, [09:40] Marcos).

### 12.3 Critério de saída

A entrega é considerada pronta quando: os critérios CA-01 a CA-18 forem verificados, a revisão de segurança de Sofia for concluída e aprovada, e ao menos um cliente piloto estiver recebendo notificações em produção.
