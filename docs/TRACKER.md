# Tracker de Rastreabilidade

Rastreabilidade de cada item registrado no pacote de design docs à sua origem — na transcrição da reunião técnica (`TRANSCRICAO.md`) ou no código-fonte da aplicação.

## Como ler esta tabela

| Coluna | Significado |
| --- | --- |
| **ID** | Identificador único do item. **O ID aparece literalmente no documento de origem**, o que permite localizar o item por busca direta. |
| **Documento** | Arquivo onde o item aparece |
| **Tipo** | Requisito Funcional, Requisito Não Funcional, Decisão, Restrição, Trade-off, Alternativa, Questão em Aberto, Risco, Contrato, Fluxo, Erro, Observabilidade, Integração, Critério de Aceite, Modelo de Dados, Objetivo |
| **Conteúdo (resumo)** | Descrição de uma linha do item |
| **Fonte** | `TRANSCRICAO` ou `CODIGO` |
| **Localização** | Para `TRANSCRICAO`: `[hh:mm] Falante`. Para `CODIGO`: caminho do arquivo |

### Notas de integridade

- **Data da reunião:** a transcrição identifica a reunião apenas como "quinta-feira, 09:00", sem data de calendário. Os documentos usam **2026-10-17** como data de produção; nenhuma afirmação da transcrição foi datada por inferência.
- **Itens derivados:** alguns itens são **derivação de engenharia** a partir de uma regra explícita da reunião (ex.: o nome exato de um código de erro dentro do prefixo `WEBHOOK_` acordado, ou o conjunto de métricas propostas). Esses itens estão marcados com *(derivado)* no resumo e apontam para a regra da qual derivam.
- **Convenção de IDs:** `DOC-TIPO-NN`, onde `DOC` é o documento (PRD, RFC, FDD, ADR) e `TIPO` é a categoria do item. Os IDs são estáveis e aparecem nos dois lados — documento e tracker.
- **Cobertura:** todo item enumerado com ID nos documentos tem linha correspondente nesta tabela.

---

## 1. PRD — `docs/PRD.md`

### 1.1 Requisitos funcionais

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastrar webhook com URL e lista de status; secret gerada pela plataforma e devolvida na criação | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Editar webhook existente (URL, status assinados, ativo) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Remover webhook | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Listar webhooks de um cliente | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Filtro de eventos por status, aplicado na inserção do evento | TRANSCRICAO | [09:34] Bruno |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Notificação HTTP automática ao cliente quando o status do pedido muda | TRANSCRICAO | [09:00] Marcos |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Histórico das últimas 100 entregas com sucesso/falha, payload, resposta e tempo de resposta | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Reprocessamento manual de evento da fila de falhas, restrito à role ADMIN | TRANSCRICAO | [09:36] Larissa |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Rotação de secret com grace period de 24h para a secret anterior | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Assinatura HMAC-SHA256 do corpo, enviada em header para validação do cliente | TRANSCRICAO | [09:20] Sofia |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Identificador único de evento na notificação, para deduplicação pelo cliente | TRANSCRICAO | [09:25] Diego |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | Reenvio automático com intervalos crescentes até o limite de tentativas | TRANSCRICAO | [09:15] Diego |
| PRD-FR-13 | docs/PRD.md | Requisito Funcional | Isolamento de eventos esgotados em fila de falhas definitivas, com payload e motivo | TRANSCRICAO | [09:18] Diego |
| PRD-FR-14 | docs/PRD.md | Requisito Funcional | Registro de quem executou o replay, para auditoria | TRANSCRICAO | [09:36] Sofia |
| PRD-FR-15 | docs/PRD.md | Requisito Funcional | Cliente identifica qual cadastro de webhook originou a notificação | TRANSCRICAO | [09:44] Sofia |
| PRD-FR-16 | docs/PRD.md | Requisito Funcional | Cliente valida que a notificação não foi adulterada | TRANSCRICAO | [09:19] Sofia |

### 1.2 Requisitos não funcionais

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Latência de entrega abaixo de 10 segundos no caso feliz | TRANSCRICAO | [09:02] Marcos |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Verificação de eventos pendentes a cada 2 segundos | TRANSCRICAO | [09:09] Diego |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Limite de 5 tentativas de entrega | TRANSCRICAO | [09:16] Larissa |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | Backoff 1m / 5m / 30m / 2h / 12h | TRANSCRICAO | [09:17] Diego |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | Janela total de retry de ~15 horas | TRANSCRICAO | [09:17] Diego |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | Timeout de 10 segundos na chamada ao cliente; estouro é falha | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-07 | docs/PRD.md | Requisito Não Funcional | Payload limitado a 64 KB; acima disso, erro (não truncar) | TRANSCRICAO | [09:24] Larissa |
| PRD-NFR-08 | docs/PRD.md | Requisito Não Funcional | HTTPS obrigatório; URLs http recusadas no cadastro | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-09 | docs/PRD.md | Requisito Não Funcional | Secret única por endpoint de webhook, nunca global | TRANSCRICAO | [09:21] Sofia |
| PRD-NFR-10 | docs/PRD.md | Requisito Não Funcional | Garantia at-least-once, com dedup do lado do cliente | TRANSCRICAO | [09:24] Diego |
| PRD-NFR-11 | docs/PRD.md | Requisito Não Funcional | Ordering por order_id apenas com worker único; não é ordering global | TRANSCRICAO | [09:13] Larissa |
| PRD-NFR-12 | docs/PRD.md | Requisito Não Funcional | Worker roda em processo separado da API | TRANSCRICAO | [09:11] Diego |
| PRD-NFR-13 | docs/PRD.md | Requisito Não Funcional | Reuso máximo dos padrões existentes do projeto | TRANSCRICAO | [09:30] Larissa |
| PRD-NFR-14 | docs/PRD.md | Requisito Não Funcional | Pelo menos 2 dias úteis de revisão de segurança antes do deploy | TRANSCRICAO | [09:46] Sofia |
| PRD-NFR-15 | docs/PRD.md | Requisito Não Funcional | Sem infraestrutura nova (broker, cache ou serviço adicional) | TRANSCRICAO | [09:07] Diego |

### 1.3 Objetivos e métricas

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-OBJ-01 | docs/PRD.md | Objetivo | Notificar mudanças em tempo real na percepção do cliente — meta: abaixo de 10s | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-02 | docs/PRD.md | Objetivo | Integrar os 3 clientes B2B demandantes (Atlas, MaxDistribuição, Nova Cargo) | TRANSCRICAO | [09:00] Marcos |
| PRD-OBJ-03 | docs/PRD.md | Objetivo | Não perder eventos de clientes indisponíveis — meta: janela de ~15h | TRANSCRICAO | [09:17] Diego |
| PRD-OBJ-04 | docs/PRD.md | Objetivo | Entregar em 3 sprints incluindo revisão de segurança | TRANSCRICAO | [09:46] Larissa |
| PRD-OBJ-05 | docs/PRD.md | Objetivo | Reduzir a dependência de polling manual no GET /orders | TRANSCRICAO | [09:00] Marcos |
| PRD-OBJ-06 | docs/PRD.md | Objetivo | Dar visibilidade da integração — meta: últimas 100 entregas por webhook | TRANSCRICAO | [09:34] Marcos |

### 1.4 Escopo — itens fora de escopo

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-ESCOPO-01 | docs/PRD.md | Restrição | Aviso por e-mail ao cliente em caso de falha — adiado para próxima fase | TRANSCRICAO | [09:37] Larissa |
| PRD-ESCOPO-02 | docs/PRD.md | Restrição | Dashboard visual / painel do cliente — descartado (time de frontend) | TRANSCRICAO | [09:40] Larissa |
| PRD-ESCOPO-03 | docs/PRD.md | Restrição | Rate limiting de envio por cliente — adiado, observar e decidir depois | TRANSCRICAO | [09:39] Larissa |
| PRD-ESCOPO-04 | docs/PRD.md | Restrição | Webhooks inbound (cliente → plataforma) — descartado; escopo é só outbound | TRANSCRICAO | [09:02] Marcos |
| PRD-ESCOPO-05 | docs/PRD.md | Restrição | Arquivamento das linhas entregues da outbox (~30 dias) — fora do escopo | TRANSCRICAO | [09:08] Diego |
| PRD-ESCOPO-06 | docs/PRD.md | Restrição | Escala horizontal do worker / ordering com múltiplos workers — adiado | TRANSCRICAO | [09:13] Diego |

### 1.5 Riscos

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-RISCO-01 | docs/PRD.md | Risco | Atlas migrar para o concorrente se a entrega não ocorrer até o fim do trimestre | TRANSCRICAO | [09:00] Marcos |
| PRD-RISCO-02 | docs/PRD.md | Risco | Cliente não implementa deduplicação e processa o mesmo evento duas vezes | TRANSCRICAO | [09:25] Sofia |
| PRD-RISCO-03 | docs/PRD.md | Risco | Cliente indisponível por mais de ~15h — evento vai para a fila de falhas sem aviso automático | TRANSCRICAO | [09:37] Larissa |
| PRD-RISCO-04 | docs/PRD.md | Risco | Vazamento de secret do lado do cliente (já ocorreu no histórico) | TRANSCRICAO | [09:22] Diego |
| PRD-RISCO-05 | docs/PRD.md | Risco | Acúmulo de eventos na outbox sem rotina de arquivamento | TRANSCRICAO | [09:08] Diego |
| PRD-RISCO-06 | docs/PRD.md | Risco | Atraso no cronograma por conta da revisão de segurança | TRANSCRICAO | [09:46] Sofia |

### 1.6 Dependências

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-DEP-01 | docs/PRD.md | Dependência | Três clientes B2B demandantes definem a urgência da feature | TRANSCRICAO | [09:00] Marcos |
| PRD-DEP-02 | docs/PRD.md | Dependência | Endpoints HTTPS operados pelos clientes | TRANSCRICAO | [09:23] Sofia |
| PRD-DEP-03 | docs/PRD.md | Dependência | Cliente precisa implementar verificação de assinatura e deduplicação | TRANSCRICAO | [09:25] Sofia |
| PRD-DEP-04 | docs/PRD.md | Dependência | Documentação da integração no portal do desenvolvedor (Marcos) | TRANSCRICAO | [09:26] Marcos |
| PRD-DEP-05 | docs/PRD.md | Dependência | Revisão de segurança de Sofia bloqueia o deploy | TRANSCRICAO | [09:46] Sofia |
| PRD-DEP-06 | docs/PRD.md | Dependência | Reuso do MySQL e da stack existentes, sem infra nova | TRANSCRICAO | [09:07] Diego |

### 1.7 Critérios de aceitação

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-CA-01 | docs/PRD.md | Critério de Aceite | Cliente consegue cadastrar, editar, listar e remover webhooks | TRANSCRICAO | [09:33] Bruno |
| PRD-CA-02 | docs/PRD.md | Critério de Aceite | Secret é gerada pela plataforma e devolvida no cadastro | TRANSCRICAO | [09:31] Marcos |
| PRD-CA-03 | docs/PRD.md | Critério de Aceite | Cliente escolhe quais status deseja receber, por webhook | TRANSCRICAO | [09:33] Marcos |
| PRD-CA-04 | docs/PRD.md | Critério de Aceite | Nenhum evento é registrado quando nenhum webhook assina o status | TRANSCRICAO | [09:34] Bruno |
| PRD-CA-05 | docs/PRD.md | Critério de Aceite | Notificação entregue em menos de 10 segundos no caso feliz | TRANSCRICAO | [09:02] Marcos |
| PRD-CA-06 | docs/PRD.md | Critério de Aceite | Notificação chega assinada e o cliente valida a assinatura | TRANSCRICAO | [09:20] Sofia |
| PRD-CA-07 | docs/PRD.md | Critério de Aceite | Notificação carrega identificador que permite deduplicação | TRANSCRICAO | [09:25] Diego |
| PRD-CA-08 | docs/PRD.md | Critério de Aceite | Cliente identifica qual cadastro de webhook originou a notificação | TRANSCRICAO | [09:44] Sofia |
| PRD-CA-09 | docs/PRD.md | Critério de Aceite | Falha de entrega é reenviada conforme a progressão acordada | TRANSCRICAO | [09:17] Diego |
| PRD-CA-10 | docs/PRD.md | Critério de Aceite | Após 5 tentativas, o evento vai para a fila de falhas com payload e motivo | TRANSCRICAO | [09:18] Diego |
| PRD-CA-11 | docs/PRD.md | Critério de Aceite | Administrador reprocessa evento da fila de falhas, e a ação é auditada | TRANSCRICAO | [09:36] Sofia |
| PRD-CA-12 | docs/PRD.md | Critério de Aceite | Usuário sem role ADMIN não consegue reprocessar eventos | TRANSCRICAO | [09:36] Larissa |
| PRD-CA-13 | docs/PRD.md | Critério de Aceite | Rotação de secret com a anterior válida por 24 horas | TRANSCRICAO | [09:21] Sofia |
| PRD-CA-14 | docs/PRD.md | Critério de Aceite | URLs http são recusadas no cadastro | TRANSCRICAO | [09:23] Sofia |
| PRD-CA-15 | docs/PRD.md | Critério de Aceite | Payload acima de 64 KB é rejeitado, não truncado | TRANSCRICAO | [09:24] Larissa |
| PRD-CA-16 | docs/PRD.md | Critério de Aceite | Cliente consulta as últimas 100 entregas de um webhook | TRANSCRICAO | [09:34] Marcos |
| PRD-CA-17 | docs/PRD.md | Critério de Aceite | Eventos de um mesmo pedido são entregues em ordem | TRANSCRICAO | [09:12] Diego |
| PRD-CA-18 | docs/PRD.md | Critério de Aceite | Serviço de entrega continua funcionando após restart da API | TRANSCRICAO | [09:11] Diego |

---

## 2. RFC — `docs/RFC.md`

### 2.1 Alternativas consideradas

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| RFC-ALT-01 | docs/RFC.md | Alternativa | Envio síncrono dentro do changeStatus — descartado por travar outros pedidos e exigir rollback | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Alternativa | Redis Streams como broker — descartado por overengineering e infra extra | TRANSCRICAO | [09:07] Diego |
| RFC-ALT-03 | docs/RFC.md | Alternativa | Trigger de banco para consumo reativo — descartado; MySQL não notifica processo externo | TRANSCRICAO | [09:09] Diego |
| RFC-ALT-04 | docs/RFC.md | Alternativa | Retry indefinido com backoff — descartado; evento ficaria pendurado para sempre | TRANSCRICAO | [09:15] Diego |
| RFC-ALT-05 | docs/RFC.md | Alternativa | Garantia exactly-once — descartada; exigiria coordenação bilateral | TRANSCRICAO | [09:25] Diego |
| RFC-ALT-06 | docs/RFC.md | Alternativa | Secret global da plataforma — descartada; vazamento comprometeria todos os clientes | TRANSCRICAO | [09:21] Sofia |

### 2.2 Questões em aberto

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| RFC-QA-01 | docs/RFC.md | Questão em Aberto | Rate limiting de envio por cliente — observar e decidir depois | TRANSCRICAO | [09:39] Larissa |
| RFC-QA-02 | docs/RFC.md | Questão em Aberto | E-mail de aviso ao cliente em caso de falha — próxima fase | TRANSCRICAO | [09:37] Larissa |
| RFC-QA-03 | docs/RFC.md | Questão em Aberto | Escalabilidade do worker — particionar por order_id ou lock pessimista no futuro | TRANSCRICAO | [09:13] Diego |
| RFC-QA-04 | docs/RFC.md | Questão em Aberto | Arquivamento das linhas entregues da outbox — sem rotina definida | TRANSCRICAO | [09:08] Diego |
| RFC-QA-05 | docs/RFC.md | Questão em Aberto | Endurecimento de autorização no CRUD de configuração — revisar mais pra frente | TRANSCRICAO | [09:37] Sofia |

### 2.3 Impacto

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| RFC-IMP-01 | docs/RFC.md | Integração | Único ponto de alteração no código existente é o changeStatus; middleware e logger intactos | TRANSCRICAO | [09:40] Bruno |
| RFC-IMP-02 | docs/RFC.md | Impacto | Passa a existir um segundo processo a operar (o worker) | TRANSCRICAO | [09:11] Diego |
| RFC-IMP-03 | docs/RFC.md | Impacto | Clientes passam a expor endpoint HTTPS com verificação de assinatura e dedup | TRANSCRICAO | [09:25] Sofia |
| RFC-IMP-04 | docs/RFC.md | Impacto | Prazo de 3 sprints incluindo a revisão de segurança | TRANSCRICAO | [09:46] Larissa |

### 2.4 Riscos

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| RFC-RISCO-01 | docs/RFC.md | Trade-off | Transação do changeStatus fica mais longa com a inserção na outbox | TRANSCRICAO | [09:04] Bruno |
| RFC-RISCO-02 | docs/RFC.md | Risco | Worker é ponto único de falha enquanto for single-worker | TRANSCRICAO | [09:12] Diego |
| RFC-RISCO-03 | docs/RFC.md | Risco | Crescimento indefinido da outbox sem rotina de arquivamento | TRANSCRICAO | [09:08] Diego |
| RFC-RISCO-04 | docs/RFC.md | Risco | Cliente não implementa deduplicação e processa duplicatas | TRANSCRICAO | [09:25] Sofia |
| RFC-RISCO-05 | docs/RFC.md | Risco | Vazamento de secret do lado do cliente | TRANSCRICAO | [09:22] Diego |
| RFC-RISCO-06 | docs/RFC.md | Risco | Entrega tardia em indisponibilidade longa do cliente | TRANSCRICAO | [09:17] Diego |

---

## 3. FDD — `docs/FDD.md`

### 3.1 Objetivos técnicos

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| FDD-OT-01 | docs/FDD.md | Objetivo | Atomicidade entre mudança de status e registro do evento | TRANSCRICAO | [09:40] Bruno |
| FDD-OT-02 | docs/FDD.md | Objetivo | Latência de entrega abaixo de 10s no caso feliz | TRANSCRICAO | [09:02] Marcos |
| FDD-OT-03 | docs/FDD.md | Objetivo | Entrega resiliente a indisponibilidade de até ~15h | TRANSCRICAO | [09:17] Diego |
| FDD-OT-04 | docs/FDD.md | Objetivo | Autenticidade e integridade verificáveis pelo cliente | TRANSCRICAO | [09:19] Sofia |
| FDD-OT-05 | docs/FDD.md | Objetivo | Deduplicação possível pelo cliente via X-Event-Id | TRANSCRICAO | [09:25] Diego |
| FDD-OT-06 | docs/FDD.md | Objetivo | Zero infraestrutura nova | TRANSCRICAO | [09:07] Diego |
| FDD-OT-07 | docs/FDD.md | Objetivo | Consistência de padrões com o restante do projeto | TRANSCRICAO | [09:30] Larissa |

### 3.2 Modelo de dados

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| FDD-MODELO-01 | docs/FDD.md | Modelo de Dados | Tabela webhook_endpoints: url, secret, customer_id, estado ativo e lista de status | TRANSCRICAO | [09:21] Bruno |
| FDD-MODELO-02 | docs/FDD.md | Modelo de Dados | Tabela webhook_outbox: evento, payload snapshot, status, tentativas e agendamento | TRANSCRICAO | [09:08] Diego |
| FDD-MODELO-03 | docs/FDD.md | Modelo de Dados | Tabela webhook_dead_letter: payload, motivo da falha, timestamp e auditoria de replay | TRANSCRICAO | [09:18] Diego |
| FDD-MODELO-04 | docs/FDD.md | Modelo de Dados | Tabela webhook_deliveries: histórico com sucesso/falha, resposta e tempo de resposta | TRANSCRICAO | [09:34] Marcos |

### 3.3 Fluxos

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| FDD-FLUXO-01 | docs/FDD.md | Fluxo | Inserção do evento na outbox dentro da transação do changeStatus, com rollback conjunto | TRANSCRICAO | [09:40] Bruno |
| FDD-FLUXO-02 | docs/FDD.md | Fluxo | Worker busca pendentes mais antigos em lote pequeno e marca como entregue | TRANSCRICAO | [09:08] Diego |
| FDD-FLUXO-03 | docs/FDD.md | Fluxo | Retry com progressão 1m/5m/30m/2h/12h antes da fila de falhas | TRANSCRICAO | [09:17] Diego |
| FDD-FLUXO-04 | docs/FDD.md | Fluxo | DLQ em tabela separada e replay manual via endpoint admin | TRANSCRICAO | [09:18] Diego |
| FDD-FLUXO-05 | docs/FDD.md | Fluxo | Snapshot do payload renderizado na inserção, não no envio | TRANSCRICAO | [09:52] Larissa |
| FDD-FLUXO-06 | docs/FDD.md | Fluxo | Ordering por order_id via created_at enquanto houver worker único | TRANSCRICAO | [09:12] Diego |

### 3.4 Contratos públicos

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | POST /api/v1/webhooks — cadastro; devolve a secret gerada | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | GET /api/v1/webhooks — lista os webhooks de um cliente | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | PATCH /api/v1/webhooks/:id — edição | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | DELETE /api/v1/webhooks/:id — remoção | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato | POST /api/v1/webhooks/:id/rotate-secret — rotação com grace period | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato | GET /api/v1/webhooks/:id/deliveries — histórico das últimas 100 entregas | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato | POST /api/v1/admin/webhooks/dead-letter/:id/replay — replay, role ADMIN | TRANSCRICAO | [09:35] Diego |
| FDD-CONTRATO-08 | docs/FDD.md | Contrato | Prefixo versionado /api/v1 no roteamento dos endpoints | CODIGO | src/routes/index.ts |
| FDD-CONTRATO-09 | docs/FDD.md | Contrato | Formato de resposta paginada { data, pagination } para listagens | CODIGO | src/shared/http/response.ts |

### 3.5 Headers e payload das chamadas de saída

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| FDD-HEADER-01 | docs/FDD.md | Contrato | Content-Type: application/json | TRANSCRICAO | [09:44] Diego |
| FDD-HEADER-02 | docs/FDD.md | Contrato | X-Event-Id com o UUID do evento, para deduplicação | TRANSCRICAO | [09:25] Diego |
| FDD-HEADER-03 | docs/FDD.md | Contrato | X-Signature com o HMAC-SHA256 do corpo | TRANSCRICAO | [09:20] Sofia |
| FDD-HEADER-04 | docs/FDD.md | Contrato | X-Timestamp com o timestamp do envio | TRANSCRICAO | [09:44] Diego |
| FDD-HEADER-05 | docs/FDD.md | Contrato | X-Webhook-Id com o id do endpoint que disparou o envio | TRANSCRICAO | [09:44] Sofia |
| FDD-PAYLOAD-01 | docs/FDD.md | Contrato | Payload enxuto: event_id, event_type, timestamp, order_id, order_number, from_status, to_status, customer_id, total_cents — sem items | TRANSCRICAO | [09:43] Diego |
| FDD-PAYLOAD-02 | docs/FDD.md | Contrato | event_type no formato "order.status_changed" | TRANSCRICAO | [09:43] Diego |
| FDD-PAYLOAD-03 | docs/FDD.md | Contrato | timestamp em ISO 8601 | TRANSCRICAO | [09:43] Diego |

### 3.6 Matriz de erros

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| FDD-ERRO-01 | docs/FDD.md | Erro | Prefixo WEBHOOK_ obrigatório em todos os códigos do módulo | TRANSCRICAO | [09:29] Larissa |
| FDD-ERRO-02 | docs/FDD.md | Erro | WEBHOOK_NOT_FOUND — webhook inexistente | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-03 | docs/FDD.md | Erro | WEBHOOK_INVALID_URL — URL não https | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-04 | docs/FDD.md | Erro | WEBHOOK_SECRET_REQUIRED — secret ausente/inválida | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-05 | docs/FDD.md | Erro | WEBHOOK_PAYLOAD_TOO_LARGE — payload acima de 64KB *(derivado da regra de limite)* | TRANSCRICAO | [09:24] Larissa |
| FDD-ERRO-06 | docs/FDD.md | Erro | WEBHOOK_CUSTOMER_NOT_FOUND — customerId inexistente no cadastro *(derivado)* | TRANSCRICAO | [09:31] Marcos |
| FDD-ERRO-07 | docs/FDD.md | Erro | WEBHOOK_INVALID_STATUS_FILTER — lista de status assinados inválida *(derivado)* | TRANSCRICAO | [09:33] Marcos |
| FDD-ERRO-08 | docs/FDD.md | Erro | WEBHOOK_DELIVERY_NOT_FOUND — histórico de entregas inexistente *(derivado)* | TRANSCRICAO | [09:34] Marcos |
| FDD-ERRO-09 | docs/FDD.md | Erro | WEBHOOK_DEAD_LETTER_NOT_FOUND — registro de DLQ inexistente no replay *(derivado)* | TRANSCRICAO | [09:18] Diego |
| FDD-ERRO-10 | docs/FDD.md | Erro | WEBHOOK_DELIVERY_FAILED — motivo gravado na DLQ após 5 tentativas *(derivado)* | TRANSCRICAO | [09:18] Diego |
| FDD-ERRO-11 | docs/FDD.md | Erro | WEBHOOK_DELIVERY_TIMEOUT — cliente não respondeu em 10s *(derivado)* | TRANSCRICAO | [09:42] Diego |
| FDD-ERRO-12 | docs/FDD.md | Erro | Formato de erro { error: { code, message, details } } produzido pelo middleware existente | CODIGO | src/middlewares/error.middleware.ts |

### 3.7 Resiliência

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| FDD-RESIL-01 | docs/FDD.md | Decisão | Timeout de 10s por chamada HTTP; estouro entra no retry | TRANSCRICAO | [09:42] Diego |
| FDD-RESIL-02 | docs/FDD.md | Decisão | 5 tentativas no total | TRANSCRICAO | [09:16] Larissa |
| FDD-RESIL-03 | docs/FDD.md | Decisão | Backoff exponencial 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Diego |
| FDD-RESIL-04 | docs/FDD.md | Decisão | Janela total de ~15h entre primeira falha e última tentativa | TRANSCRICAO | [09:17] Diego |
| FDD-RESIL-05 | docs/FDD.md | Decisão | DLQ em tabela separada após a 5ª falha | TRANSCRICAO | [09:18] Diego |
| FDD-RESIL-06 | docs/FDD.md | Decisão | Replay manual via endpoint admin com role ADMIN | TRANSCRICAO | [09:36] Larissa |
| FDD-RESIL-07 | docs/FDD.md | Restrição | Sem fallback de aviso nesta fase — e-mail fora de escopo | TRANSCRICAO | [09:37] Larissa |
| FDD-RESIL-08 | docs/FDD.md | Decisão | Idempotência via at-least-once com dedup pelo cliente | TRANSCRICAO | [09:24] Diego |
| FDD-RESIL-09 | docs/FDD.md | Decisão | Isolamento de falha do worker por processo separado | TRANSCRICAO | [09:11] Diego |

### 3.8 Observabilidade

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| FDD-OBS-01 | docs/FDD.md | Observabilidade | Logs estruturados via Pino existente, com redact de campos sensíveis; sem logger novo | TRANSCRICAO | [09:29] Bruno |
| FDD-OBS-02 | docs/FDD.md | Observabilidade | Conjunto proposto de métricas (enfileirados, entregas, duração, pendentes, tamanho da DLQ) *(derivado, a confirmar)* | CODIGO | src/shared/logger/index.ts |
| FDD-OBS-03 | docs/FDD.md | Observabilidade | Correlação por eventId entre transação de origem e tentativas do worker *(derivado)* | CODIGO | src/middlewares/request-logger.middleware.ts |

### 3.9 Critérios de aceite técnicos

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| FDD-CA-01 | docs/FDD.md | Critério de Aceite | Rollback não deixa linha na outbox | TRANSCRICAO | [09:40] Bruno |
| FDD-CA-02 | docs/FDD.md | Critério de Aceite | Commit deixa uma linha por webhook ativo que assina o novo status | TRANSCRICAO | [09:40] Bruno |
| FDD-CA-03 | docs/FDD.md | Critério de Aceite | Nenhuma linha inserida se nenhum webhook assina o status de destino | TRANSCRICAO | [09:34] Bruno |
| FDD-CA-04 | docs/FDD.md | Critério de Aceite | Payload persistido é o snapshot do momento da mudança | TRANSCRICAO | [09:52] Larissa |
| FDD-CA-05 | docs/FDD.md | Critério de Aceite | Entrega em até 2 segundos do ciclo de polling | TRANSCRICAO | [09:09] Diego |
| FDD-CA-06 | docs/FDD.md | Critério de Aceite | Eventos do mesmo order_id entregues na ordem de createdAt | TRANSCRICAO | [09:12] Diego |
| FDD-CA-07 | docs/FDD.md | Critério de Aceite | Falha agenda próxima tentativa conforme a progressão acordada | TRANSCRICAO | [09:17] Diego |
| FDD-CA-08 | docs/FDD.md | Critério de Aceite | Após 5 tentativas, evento vai para a DLQ com payload e motivo | TRANSCRICAO | [09:18] Diego |
| FDD-CA-09 | docs/FDD.md | Critério de Aceite | X-Signature corresponde ao HMAC-SHA256 do corpo com a secret do endpoint | TRANSCRICAO | [09:20] Sofia |
| FDD-CA-10 | docs/FDD.md | Critério de Aceite | Mesmo eventId em todas as tentativas do mesmo evento | TRANSCRICAO | [09:25] Diego |
| FDD-CA-11 | docs/FDD.md | Critério de Aceite | Replay exige role ADMIN e registra replayedById | TRANSCRICAO | [09:36] Sofia |
| FDD-CA-12 | docs/FDD.md | Critério de Aceite | URL http recusada com WEBHOOK_INVALID_URL | TRANSCRICAO | [09:23] Sofia |
| FDD-CA-13 | docs/FDD.md | Critério de Aceite | Payload acima de 64KB resulta em erro, não truncamento | TRANSCRICAO | [09:24] Larissa |
| FDD-CA-14 | docs/FDD.md | Critério de Aceite | Durante o grace period, a secret anterior também valida a assinatura | TRANSCRICAO | [09:21] Sofia |
| FDD-CA-15 | docs/FDD.md | Critério de Aceite | Worker roda como processo independente e sobrevive a restart da API | TRANSCRICAO | [09:11] Diego |
| FDD-CA-16 | docs/FDD.md | Critério de Aceite | Todos os erros do módulo usam prefixo WEBHOOK_ e formato padrão | TRANSCRICAO | [09:29] Larissa |

### 3.10 Riscos

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| FDD-RISCO-01 | docs/FDD.md | Risco | Transação do changeStatus mais longa, aumentando contenção | TRANSCRICAO | [09:04] Bruno |
| FDD-RISCO-02 | docs/FDD.md | Risco | Worker é ponto único de falha (single-worker) | TRANSCRICAO | [09:12] Diego |
| FDD-RISCO-03 | docs/FDD.md | Risco | Crescimento indefinido da outbox sem arquivamento | TRANSCRICAO | [09:08] Diego |
| FDD-RISCO-04 | docs/FDD.md | Risco | Cliente não implementa deduplicação e processa duplicatas | TRANSCRICAO | [09:25] Sofia |
| FDD-RISCO-05 | docs/FDD.md | Risco | Vazamento da secret do lado do cliente | TRANSCRICAO | [09:22] Diego |
| FDD-RISCO-06 | docs/FDD.md | Risco | Cliente indisponível por mais de ~15h — evento cai na DLQ e fica parado | TRANSCRICAO | [09:17] Diego |
| FDD-RISCO-07 | docs/FDD.md | Risco | Chamada HTTP sem timeout travaria o ciclo do worker | TRANSCRICAO | [09:42] Diego |
| FDD-RISCO-08 | docs/FDD.md | Risco | Payload grande pressiona a transação e o armazenamento | TRANSCRICAO | [09:24] Larissa |

### 3.11 Integração com o sistema existente

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| FDD-INTEG-01 | docs/FDD.md | Integração | changeStatus estendido com publishWebhookEvent(tx, order, from, to) dentro da transação | CODIGO | src/modules/orders/order.service.ts |
| FDD-INTEG-02 | docs/FDD.md | Integração | Máquina de estados e conjunto de status assináveis (canTransition, isTerminal) | CODIGO | src/modules/orders/order.status.ts |
| FDD-INTEG-03 | docs/FDD.md | Integração | Novos erros herdam de AppError e subclasses, seguindo o padrão de códigos | CODIGO | src/shared/errors/http-errors.ts |
| FDD-INTEG-04 | docs/FDD.md | Integração | Error middleware centralizado trata os erros do módulo sem alteração | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INTEG-05 | docs/FDD.md | Integração | requireRole('ADMIN') reutilizado no endpoint de replay | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INTEG-06 | docs/FDD.md | Integração | validate({ body, params, query }) reutilizado com schemas Zod do módulo | CODIGO | src/middlewares/validate.middleware.ts |
| FDD-INTEG-07 | docs/FDD.md | Integração | Registro do controller em buildControllers e montagem do roteador sob /api/v1 | CODIGO | src/app.ts |
| FDD-INTEG-08 | docs/FDD.md | Integração | src/worker.ts segue o molde de bootstrap e shutdown do src/server.ts | CODIGO | src/server.ts |
| FDD-INTEG-09 | docs/FDD.md | Integração | Novos modelos no schema, com chaves uuid Char(36), @@map e @@index | CODIGO | prisma/schema.prisma |

### 3.12 Dependências e compatibilidade

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| FDD-DEP-01 | docs/FDD.md | Dependência | MySQL existente reusado; nenhuma instância nova | TRANSCRICAO | [09:07] Diego |
| FDD-DEP-02 | docs/FDD.md | Dependência | Prisma/PrismaClient reusado; worker com client próprio por ser outro processo | TRANSCRICAO | [09:30] Bruno |
| FDD-DEP-03 | docs/FDD.md | Dependência | Pino reusado sem alteração | TRANSCRICAO | [09:29] Bruno |
| FDD-DEP-04 | docs/FDD.md | Dependência | Zod reusado para os schemas de validação do módulo | TRANSCRICAO | [09:30] Larissa |
| FDD-DEP-05 | docs/FDD.md | Dependência | `uuid` já é dependência do projeto; reaproveitada para o eventId | CODIGO | package.json |
| FDD-DEP-06 | docs/FDD.md | Dependência | Cliente HTTP: `fetch` global do Node 20, sem nova dependência | CODIGO | package.json |
| FDD-DEP-07 | docs/FDD.md | Dependência | HMAC via `crypto.createHmac` do Node (built-in), sem nova dependência | CODIGO | package.json |
| FDD-DEP-08 | docs/FDD.md | Restrição | Biblioteca de métricas não existe hoje; a definir na implementação *(derivado)* | CODIGO | src/shared/logger/index.ts |
| FDD-DEP-09 | docs/FDD.md | Restrição | Nenhuma alteração necessária em error middleware, logger, módulos ou configs | TRANSCRICAO | [09:29] Bruno |
| FDD-DEP-10 | docs/FDD.md | Restrição | Migrations aditivas; nenhuma tabela existente é alterada | CODIGO | prisma/schema.prisma |

---

## 4. ADRs — `docs/adrs/`

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Padrão Outbox no MySQL, com inserção na mesma transação do changeStatus | TRANSCRICAO | [09:06] Diego |
| ADR-001-ALT-01 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa | Envio síncrono dentro da transação — descartado | TRANSCRICAO | [09:04] Bruno |
| ADR-001-ALT-02 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa | Redis Streams — descartado por overengineering | TRANSCRICAO | [09:07] Diego |
| ADR-001-ALT-03 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa | Trigger de banco — descartada | TRANSCRICAO | [09:09] Diego |
| ADR-001-CONS-01 | docs/adrs/ADR-001-outbox-no-mysql.md | Restrição | Índice no status da outbox e em created_at; leitura em lote pequeno | TRANSCRICAO | [09:08] Diego |
| ADR-001-CONS-02 | docs/adrs/ADR-001-outbox-no-mysql.md | Restrição | Identificador da outbox é UUID, seguindo o padrão do projeto | TRANSCRICAO | [09:51] Larissa |
| ADR-002 | docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md | Decisão | Worker em processo separado com polling de 2 segundos | TRANSCRICAO | [09:11] Diego |
| ADR-002-ALT-01 | docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md | Alternativa | Worker embutido no processo da API — descartado (perde-se no restart) | TRANSCRICAO | [09:11] Diego |
| ADR-002-ALT-02 | docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md | Alternativa | Múltiplos workers em paralelo — adiado; perde ordering | TRANSCRICAO | [09:12] Diego |
| ADR-002-CONS-01 | docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md | Trade-off | Ordering por order_id apenas com worker único | TRANSCRICAO | [09:13] Larissa |
| ADR-002-CONS-02 | docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md | Restrição | src/worker.ts como entry-point nova e script npm run worker | TRANSCRICAO | [09:11] Larissa |
| ADR-002-CONS-03 | docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md | Decisão | Worker usa PrismaClient próprio, mesmo banco e mesma DATABASE_URL | TRANSCRICAO | [09:30] Bruno |
| ADR-003 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Decisão | 5 tentativas com backoff 1m/5m/30m/2h/12h e DLQ em tabela separada | TRANSCRICAO | [09:17] Diego |
| ADR-003-ALT-01 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Alternativa | Retry indefinido — descartado; evento pendurado para sempre | TRANSCRICAO | [09:15] Diego |
| ADR-003-ALT-02 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Alternativa | Teto de 3 tentativas — descartado; não cobre manutenção de 2h | TRANSCRICAO | [09:16] Diego |
| ADR-003-ALT-03 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Alternativa | Marcar como failed na própria outbox — descartado; tabela separada é mais limpa | TRANSCRICAO | [09:18] Diego |
| ADR-003-CONS-01 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Decisão | DLQ guarda payload, motivo da falha e timestamp | TRANSCRICAO | [09:18] Diego |
| ADR-003-CONS-02 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Decisão | Replay manual recoloca o evento na outbox como pendente | TRANSCRICAO | [09:18] Diego |
| ADR-003-CONS-03 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Trade-off | Janela de ~15h validada pelo negócio como aceitável | TRANSCRICAO | [09:17] Marcos |
| ADR-004 | docs/adrs/ADR-004-assinatura-hmac-sha256-secret-por-endpoint.md | Decisão | HMAC-SHA256 sobre o corpo, secret por endpoint, rotação com grace period de 24h | TRANSCRICAO | [09:22] Sofia |
| ADR-004-ALT-01 | docs/adrs/ADR-004-assinatura-hmac-sha256-secret-por-endpoint.md | Alternativa | Secret global — descartada; vazamento comprometeria todos | TRANSCRICAO | [09:21] Sofia |
| ADR-004-ALT-02 | docs/adrs/ADR-004-assinatura-hmac-sha256-secret-por-endpoint.md | Alternativa | Rotação imediata sem grace period — descartada; quebraria a integração | TRANSCRICAO | [09:21] Sofia |
| ADR-004-CONS-01 | docs/adrs/ADR-004-assinatura-hmac-sha256-secret-por-endpoint.md | Decisão | Algoritmo SHA-256 por ser padrão de mercado | TRANSCRICAO | [09:20] Sofia |
| ADR-004-CONS-02 | docs/adrs/ADR-004-assinatura-hmac-sha256-secret-por-endpoint.md | Decisão | Schema de configuração: url + secret + customer_id + estado ativo | TRANSCRICAO | [09:21] Bruno |
| ADR-004-CONS-03 | docs/adrs/ADR-004-assinatura-hmac-sha256-secret-por-endpoint.md | Risco | Precedente real de cliente vazando secret em log de aplicação | TRANSCRICAO | [09:22] Diego |
| ADR-004-CONS-04 | docs/adrs/ADR-004-assinatura-hmac-sha256-secret-por-endpoint.md | Restrição | Validação de HTTPS como validação de schema, não decisão arquitetural | TRANSCRICAO | [09:23] Sofia |
| ADR-005 | docs/adrs/ADR-005-garantia-at-least-once-com-x-event-id.md | Decisão | Garantia at-least-once com X-Event-Id para dedup no cliente | TRANSCRICAO | [09:25] Diego |
| ADR-005-ALT-01 | docs/adrs/ADR-005-garantia-at-least-once-com-x-event-id.md | Alternativa | Exactly-once — descartado; exigiria coordenação dos dois lados | TRANSCRICAO | [09:25] Diego |
| ADR-005-ALT-02 | docs/adrs/ADR-005-garantia-at-least-once-com-x-event-id.md | Alternativa | At-most-once — incompatível com o requisito de não perder eventos | TRANSCRICAO | [09:24] Diego |
| ADR-005-CONS-01 | docs/adrs/ADR-005-garantia-at-least-once-com-x-event-id.md | Trade-off | Responsabilidade de deduplicação transferida ao cliente | TRANSCRICAO | [09:25] Sofia |
| ADR-005-CONS-02 | docs/adrs/ADR-005-garantia-at-least-once-com-x-event-id.md | Decisão | event_id é UUID gerado na inserção na outbox | TRANSCRICAO | [09:25] Diego |
| ADR-005-CONS-03 | docs/adrs/ADR-005-garantia-at-least-once-com-x-event-id.md | Decisão | Comportamento at-least-once documentado no portal do desenvolvedor | TRANSCRICAO | [09:26] Marcos |
| ADR-005-CONS-04 | docs/adrs/ADR-005-garantia-at-least-once-com-x-event-id.md | Trade-off | Padrão de mercado (Stripe, GitHub) como justificativa | TRANSCRICAO | [09:25] Diego |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Decisão | Reuso máximo dos padrões existentes; webhook é módulo igual aos outros | TRANSCRICAO | [09:30] Larissa |
| ADR-006-CONS-01 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Integração | Estrutura modular controller/service/repository/routes/schemas como padrão | TRANSCRICAO | [09:27] Bruno |
| ADR-006-CONS-02 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Integração | Reuso de AppError, Pino, error middleware, schemas Zod e requireRole | TRANSCRICAO | [09:30] Larissa |
| ADR-006-CONS-03 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Integração | Error middleware existente cobre os erros do módulo sem alteração | TRANSCRICAO | [09:29] Bruno |
| ADR-006-ALT-01 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Alternativa | Serviço/projeto separado — descartado; inviabilizaria a atomicidade | TRANSCRICAO | [09:40] Bruno |
| ADR-006-ALT-02 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Alternativa | Injetar o repository de webhooks no OrderService — preterido por função pura com tx | TRANSCRICAO | [09:41] Bruno |
| ADR-007 | docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md | Decisão | Payload renderizado como snapshot na inserção da outbox | TRANSCRICAO | [09:52] Larissa |
| ADR-007-ALT-01 | docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md | Alternativa | Guardar só order_id e renderizar no envio — descartado; geraria evento inconsistente | TRANSCRICAO | [09:52] Larissa |
| ADR-007-CONS-01 | docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md | Trade-off | Payload duplicado no banco e trabalho extra dentro da transação | TRANSCRICAO | [09:08] Diego |
| ADR-007-CONS-02 | docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md | Trade-off | Replay envia exatamente o mesmo payload, tornando-o previsível | TRANSCRICAO | [09:52] Diego |

---

## 5. Referências de código consolidadas

Caminhos de arquivo do repositório citados nos documentos. Todos os caminhos abaixo **existem** na aplicação e foram conferidos:

> Arquivos **novos** propostos pela feature (`src/worker.ts`, `src/modules/webhooks/webhook.worker.ts`, `src/modules/webhooks/webhook.schemas.ts` e os demais arquivos de `src/modules/webhooks/`) **não** aparecem nesta tabela justamente por não existirem ainda — eles estão listados no FDD §11 com a marcação de "arquivo novo a ser criado".

| Caminho | Usado em |
| --- | --- |
| `src/modules/orders/order.service.ts` | FDD-INTEG-01, ADR-001, ADR-006, ADR-007, RFC §3.5 |
| `src/modules/orders/order.status.ts` | FDD-INTEG-02, ADR-002 |
| `src/modules/orders/order.repository.ts` | FDD §11, ADR-007 |
| `src/modules/orders/order.schemas.ts` | FDD-INTEG-06, ADR-004, ADR-006 |
| `src/modules/orders/order.routes.ts` | FDD §4 (padrão de rotas) |
| `src/modules/orders/order.controller.ts` | FDD §4 (padrão de controllers) |
| `src/shared/errors/app-error.ts` | FDD-INTEG-03, ADR-006 |
| `src/shared/errors/http-errors.ts` | FDD-INTEG-03, FDD §5, ADR-006 |
| `src/shared/errors/index.ts` | FDD §11 |
| `src/shared/logger/index.ts` | FDD-OBS-01, FDD-OBS-02, ADR-004, ADR-006 |
| `src/shared/http/response.ts` | FDD-CONTRATO-09, ADR-005 |
| `src/middlewares/error.middleware.ts` | FDD-INTEG-04, FDD-ERRO-12, ADR-003, ADR-006 |
| `src/middlewares/auth.middleware.ts` | FDD-INTEG-05, ADR-003, ADR-006 |
| `src/middlewares/validate.middleware.ts` | FDD-INTEG-06, ADR-004, ADR-006 |
| `src/middlewares/request-logger.middleware.ts` | FDD-OBS-03 |
| `src/modules/users/user.routes.ts` | FDD-INTEG-05, ADR-003, ADR-006 |
| `src/app.ts` | FDD-INTEG-07 |
| `src/routes/index.ts` | FDD-INTEG-07, FDD-CONTRATO-08 |
| `src/server.ts` | FDD-INTEG-08, ADR-002, ADR-006 |
| `src/config/database.ts` | FDD §11, ADR-001, ADR-002, ADR-006 |
| `src/config/env.ts` | FDD §8 (variáveis de ambiente) |
| `prisma/schema.prisma` | FDD-INTEG-09, FDD §2, ADR-001, ADR-004, ADR-005 |
| `package.json` | FDD §8 (Node >= 20, scripts, dependências) |
| `tests/orders.test.ts` | PRD §12.1 |
| `tests/auth.test.ts` | PRD §12.1 |
| `tests/helpers/factories.ts` | PRD §12.1 |

---

## 6. Estatísticas de cobertura

| Métrica | Valor |
| --- | --- |
| Total de linhas de rastreabilidade | **238** |
| Linhas com Fonte = `TRANSCRICAO` | **219 (92,0%)** |
| Linhas com Fonte = `CODIGO` | **19 (8,0%)** |
| Cobertura de itens identificáveis nos documentos | 100% dos itens enumerados com ID |
| Documentos cobertos | PRD, RFC, FDD, 7 ADRs |
| ADRs cobertos | 7 de 7 |
| Caminhos de arquivo distintos com Fonte = `CODIGO` | 14 |
| Caminhos de arquivo distintos na tabela consolidada (§5) | 26 |

Todos os caminhos de arquivo referenciados foram verificados como existentes no repositório. Os três arquivos novos da feature estão marcados como tal no FDD §11.

**Verificação de integridade dos IDs:** todo ID do tracker aparece literalmente no documento que ele referencia, o que permite localizar o item por busca direta no arquivo.
