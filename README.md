# Da Reunião ao Documento: Design Docs Gerados por IA

Documentação do processo de produção do pacote de design docs da feature **Sistema de Webhooks de Notificação de Pedidos**, a partir da transcrição de uma reunião técnica e do código de um Order Management System existente.

> O enunciado original do desafio foi substituído por este documento, conforme previsto no requisito 6. O repositório base é [devfullcycle/mba-ia-desafio-design-docs-com-ia](https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia).

---

## Sobre o desafio

O ponto de partida era uma gravação de ~55 minutos de uma reunião técnica entre cinco pessoas — tech lead, PM, dois engenheiros e uma engenheira de segurança — na qual a decisão de construir um sistema de webhooks de notificação de pedidos foi tomada. Nada foi registrado além da transcrição. A tarefa era transformar essa conversa, cruzada com o código de uma aplicação Node.js + TypeScript funcional, em um pacote de documentação técnica acionável: PRD, RFC, FDD, ADRs, um tracker de rastreabilidade e este README.

O que torna o exercício não trivial é a restrição de **rastreabilidade absoluta**: nenhum requisito, decisão ou restrição podia existir sem origem identificável na transcrição ou no código. A reunião descarta coisas explicitamente (e-mail de aviso, dashboard, rate limiting), adia outras e decide algumas em detalhe — e distinguir "o que entra" de "o que foi mencionado e descartado" foi metade do trabalho. A outra metade foi fazer cinco documentos falarem em alturas diferentes sem se repetirem.

## Ferramentas de IA utilizadas

| Ferramenta | Papel no processo |
| --- | --- |
| **Claude Code (Anthropic)** | Ferramenta principal e única de produção. Foi usada com acesso direto ao repositório — leitura do código-fonte, da transcrição e dos arquivos de documentação —, o que permitiu citar caminhos de arquivo, funções e padrões reais em vez de descrições genéricas. Responsável por ler e mapear a codebase, extrair e classificar os itens da transcrição, redigir os seis documentos e manter a consistência cruzada entre eles. |
| **Grep / leitura direta de arquivos** | Não é uma ferramenta de IA, mas foi o mecanismo de verificação: toda estatística, contagem de linhas do tracker e existência de caminho de arquivo foi conferida por comando no repositório, não estimada pelo modelo. |

Não foram usados geradores de documento automáticos nem templates prontos. A estrutura de cada documento (MADR para os ADRs, o formato do curso para PRD/RFC/FDD) foi dada ao modelo como restrição de formato, não como conteúdo.

## Workflow adotado

A ordem de produção seguiu a sequência sugerida pelo enunciado, que se mostrou correta na prática: **as decisões são o esqueleto de tudo o mais**.

```
1. Leitura e contextualização
   ├── leitura integral do código (src/, prisma/, tests/, configs)
   └── leitura integral da transcrição, com extração cronológica dos itens

2. ADRs (7 arquivos)          ← esqueleto das decisões
3. RFC                        ← consolida as decisões + alternativas + questões em aberto
4. FDD                        ← detalha o "como construir" sobre as decisões
5. PRD                        ← consolidação em nível de produto
6. Tracker                    ← varredura dos documentos prontos
7. README                     ← documentação do processo
8. Verificação final          ← contagens, existência de caminhos, checklist de aceite
```

### Por que essa ordem

- **ADRs primeiro.** Cada decisão fechada na reunião virou um arquivo com status, contexto, decisão, alternativas e consequências. Produzi-los antes de tudo deu uma base estável: o RFC pôde *referenciar* decisões em vez de *reafirmá-las*, e o FDD pôde assumir as decisões como resolvidas.
- **RFC em seguida, curto de propósito.** O RFC é o documento onde as alternativas descartadas e as questões em aberto têm lugar natural. Mantê-lo em nível de arquitetura — sem payloads, sem matriz de erros — foi uma restrição deliberada para não invadir o FDD.
- **FDD depois.** Com as decisões formalizadas, o FDD virou detalhamento puro: fluxos, contratos, erros, observabilidade e a seção obrigatória de integração com o código.
- **PRD por último entre os grandes.** Em nível de produto, com RFC + FDD + ADRs prontos, o PRD foi praticamente uma consolidação — o esforço ficou em escolher a altura certa (problema, público, métricas) e não repetir decisão técnica.
- **Tracker no fim.** Ele só faz sentido varrendo documentos prontos.

### Como a interação foi organizada

A interação foi dirigida, não conversacional. Em vez de pedir "gere um PRD a partir dessa transcrição" — o exemplo de prompt vago que o próprio enunciado alerta contra —, cada documento foi produzido com uma instrução que fixava: (a) o **papel e a altura** do documento, (b) o **que ele não deve conter**, (c) a **obrigação de citar origem** para cada item e (d) a **estrutura de seções** exigida.

O trabalho de maior valor não foi gerar texto, foi **classificar a transcrição**: separar requisito de decisão, decisão de alternativa descartada, alternativa descartada de item adiado, e item adiado de ruído. Essa classificação foi feita item a item, com timestamp e falante, antes de escrever qualquer documento — e é ela que sustenta o tracker.

## Prompts customizados

Dois prompts usados no processo, reproduzidos como foram estruturados.

### Prompt 1 — Extração e classificação dos itens da transcrição

Usado **antes** de escrever qualquer documento, para construir a base de rastreabilidade.

```
Você vai ler TRANSCRICAO.md, a gravação de uma reunião técnica de ~55 minutos
sobre a construção de um sistema de webhooks de notificação de pedidos.

Não resuma a reunião. Em vez disso, percorra a transcrição na ordem cronológica
e extraia CADA item mencionado, classificando-o em uma destas categorias:

  - DECISÃO FECHADA      → algo que o grupo explicitamente decidiu
  - REQUISITO FUNCIONAL  → comportamento que o produto deve ter
  - REQUISITO NÃO FUNC.  → restrição de performance, segurança, tamanho, latência
  - RESTRIÇÃO            → limite imposto ao desenho
  - ALTERNATIVA DESCARTADA → ideia levantada e explicitamente rejeitada,
                             com o MOTIVO do descarte
  - ADIADO / FORA DE ESCOPO → mencionado e deliberadamente deixado para depois
  - RUÍDO                → conversa que não vira nada (ex.: confirmações, despedidas)

Para cada item, registre obrigatoriamente:
  - o timestamp e o falante no formato [hh:mm] Nome
  - a fala literal que sustenta a classificação (citação curta)

Regras:
  - Se você não consegue apontar timestamp + falante, o item não existe: descarte.
  - Não infira requisitos. Se algo é dedutível mas não foi dito, marque como
    "INFERIDO" e liste separadamente — vou revisar um por um.
  - Preste atenção especial ao que NÃO entra: e-mail de aviso, dashboard,
    rate limiting, webhook inbound e arquivamento foram descartados ou adiados.
    Se qualquer um deles aparecer depois como requisito, é erro.

Saída: uma tabela por categoria, ordenada por timestamp.
```

O valor desse prompt está nas duas últimas regras. A exigência de citação literal impede que o modelo "melhore" a fala de alguém, e a lista explícita do que não entra funciona como teste de contaminação: se um desses itens reaparecer depois como requisito, é sinal de que a filtragem falhou.

### Prompt 2 — FDD com obrigação de ancoragem no código

Usado para o documento mais técnico, onde o risco de invenção é maior.

```
Produza docs/FDD.md para a feature de webhooks, com base nas decisões já
formalizadas em docs/adrs/ e no código real do repositório.

Altitude: este documento é o "como construir em detalhe". Ele pode e deve
descer ao nível de tabela, coluna, endpoint, header e código de erro.

Estrutura obrigatória:
  1. Contexto e motivação técnica
  2. Objetivos técnicos
  3. Escopo e exclusões
  4. Fluxos detalhados (inserção na outbox, processamento pelo worker, retry, DLQ)
  5. Contratos públicos — mínimo 4 endpoints, cada um com payload de request,
     payload de response e tabela de status codes
  6. Matriz de erros com códigos no prefixo WEBHOOK_
  7. Estratégias de resiliência (timeout, retry, backoff, fallback)
  8. Observabilidade (métricas, logs, tracing)
  9. Dependências e compatibilidade
  10. Critérios de aceite técnicos
  11. Riscos e mitigação
  12. Integração com o sistema existente

Regras de ancoragem — inegociáveis:
  - Na seção 12, nomeie no MÍNIMO 4 caminhos de arquivo REAIS do repositório e
    descreva como o módulo de webhooks se integra a cada um. Leia o arquivo
    antes de descrevê-lo. Não cite nenhum arquivo que você não tenha lido.
  - Os nomes de código de erro devem seguir o prefixo WEBHOOK_ acordado.
    A reunião nomeou explicitamente apenas 3 códigos. Para os demais, derive
    do padrão de src/shared/errors/http-errors.ts e marque cada um como
    "derivado", indicando a regra da qual deriva.
  - Onde um valor for numérico (timeout, limite, tentativas, backoff), use
    exatamente o valor acordado na reunião. Não arredonde nem "melhore".
  - Se a infraestrutura proposta não existe no projeto (ex.: biblioteca de
    métricas), diga isso explicitamente em vez de assumir que existe.

Não duplique o RFC: nada de reexplicar alternativas descartadas.
```

A regra "leia o arquivo antes de descrevê-lo" e a obrigação de marcar itens derivados foram o que separou o FDD de um documento plausível — o tipo de texto que *parece* certo e cita arquivos que não existem.

## Iterações e ajustes

O processo não saiu pronto de primeira. Os ajustes concretos:

### Iteração 1 — As estatísticas do tracker estavam estimadas, não medidas

A primeira versão do tracker trazia, na seção de cobertura, números estimados de cabeça: "156 linhas, 135 com fonte TRANSCRICAO (86,5%), 21 com fonte CODIGO". Ao conferir com um comando sobre o arquivo real, os números já eram outros — e mudaram mais duas vezes, primeiro por causa da reescrita descrita na Iteração 2 e depois ao fechar o último gap de cobertura. Os números finais, **238 linhas, 219 TRANSCRICAO (92,0%) e 19 CODIGO (8,0%)**, foram todos obtidos por comando.

O detalhe que importa: a porcentagem estimada estava "certa por coincidência", o que é pior do que estar errada, porque passaria despercebida numa revisão superficial. **Correção:** nenhuma afirmação quantitativa do pacote ficou baseada em estimativa — todas passaram a ser obtidas por contagem sobre o arquivo final.

### Iteração 2 — Os IDs do tracker não existiam nos documentos

Este foi o ajuste mais estrutural. O tracker usava IDs no formato sugerido pelo enunciado (`PRD-FR-01`, `FDD-CONTRATO-03`), mas os documentos usavam IDs locais (`FR-01`, `### 4.1`). O resultado era um tracker que *parecia* correto e era **inútil para busca**: procurar `PRD-FR-01` no PRD não retornava nada.

É exatamente o tipo de defeito que passa numa leitura linear e falha no uso real — a função declarada do tracker é permitir que alguém encontre de onde veio cada item.

**Correção:** alinhei os dois lados. Cada item dos documentos passou a carregar o ID canônico (`| **PRD-FR-01** |`, `### FDD-CONTRATO-01 —`, `### RFC-ALT-01 —`), e o tracker foi reescrito sobre esses IDs. Uma verificação automática passou a garantir que **todo ID do tracker aparece literalmente no documento que ele referencia**. Para itens que não têm título próprio (a decisão de snapshot, a ordenação, o prefixo versionado), o ID foi inserido inline no texto onde o item é discutido.

### Iteração 3 — A reunião não tem data, e inventar uma seria uma alucinação

Os documentos exigem metadados com data (RFC, ADRs). A transcrição diz apenas "quinta-feira, 09:00" — **não há data de calendário em lugar nenhum**. A tentação é preencher com uma data plausível e seguir.

**Correção:** em vez de inventar, adotei **2026-10-17** como data de *produção* dos documentos (a data em que foram escritos), e registrei a limitação explicitamente — no cabeçalho dos ADRs ("decisão tomada na reunião técnica de quinta-feira, 09:00") e numa nota de integridade no topo do tracker, que diz que nenhuma afirmação da transcrição foi datada por inferência. A ausência de data virou informação documentada, não um dado fabricado.

### Iteração 4 — Três códigos de erro reais vs. onze na matriz

O FDD exige uma matriz de erros com prefixo `WEBHOOK_`. A reunião nomeia explicitamente **apenas três** códigos: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL` e `WEBHOOK_SECRET_REQUIRED` ([09:28] Bruno). A matriz final tem onze.

Apresentar onze códigos como se todos tivessem sido ditos na reunião seria uma fabricação sutil — o tipo que passa porque cada linha individualmente *parece* razoável. **Correção:** marquei cada código derivado com "*(derivado)*" no tracker, apontando para a regra da qual ele deriva (o limite de 64KB gera `WEBHOOK_PAYLOAD_TOO_LARGE`; a validação de HTTPS gera `WEBHOOK_INVALID_URL`, que *é* citado; o cadastro por `customerId` gera `WEBHOOK_CUSTOMER_NOT_FOUND`). A coluna de origem continua sendo TRANSCRICAO porque a *regra* vem da transcrição — mas o nome do código está sinalizado como derivação, não como citação.

### Iteração 5 — O caminho do endpoint não é o que foi dito na reunião

Diego refere-se ao endpoint de replay como `POST /admin/webhooks/dead-letter/:id/replay` ([09:35] Diego). Mas a aplicação monta tudo sob `/api/v1` (`src/routes/index.ts`), então o caminho real é `/api/v1/admin/webhooks/dead-letter/:id/replay`.

**Correção:** usei o caminho completo no FDD, com uma nota explicando a diferença em relação à fala da reunião, e registrei a origem nos dois formatos. Sem isso, um desenvolvedor implementando a partir do FDD teria um caminho que não bate com o roteamento real da aplicação.

### Iteração 6 — RFC e FDD estavam começando a se repetir

A primeira passagem do RFC trazia payloads de exemplo e a progressão completa do backoff — conteúdo que pertence ao FDD. O sinal foi o próprio critério do enunciado: "conteúdo duplicado entre documentos é sinal de que algo está no lugar errado".

**Correção:** o RFC foi reescrito para operar em nível de arquitetura. As decisões viraram uma tabela de uma linha por tema, cada uma apontando para o ADR correspondente; as alternativas descartadas ficaram só com o trade-off que motivou o descarte; e o detalhamento foi delegado explicitamente ao FDD ("o detalhamento de implementação está no `docs/FDD.md` e não é objeto deste RFC").

**Total: 6 iterações principais** entre geração, verificação e correção. A Iteração 2 é a que mais mudou o resultado — e é a que um leitor desatento não notaria, porque o documento *parecia* pronto antes dela.

## Como navegar a entrega

### Arquivos entregues

```
.
├── README.md                                  ← este documento (processo de produção)
├── TRANSCRICAO.md                             ← fonte primária (não alterado)
└── docs/
    ├── PRD.md                                 ← produto: por que e o quê
    ├── RFC.md                                 ← arquitetura: proposta e questões em aberto
    ├── FDD.md                                 ← implementação: como construir em detalhe
    ├── TRACKER.md                             ← rastreabilidade de cada item
    └── adrs/
        ├── README.md                          ← índice dos ADRs
        ├── ADR-001-outbox-no-mysql.md
        ├── ADR-002-worker-em-processo-separado-com-polling.md
        ├── ADR-003-retry-com-backoff-exponencial-e-dlq.md
        ├── ADR-004-assinatura-hmac-sha256-secret-por-endpoint.md
        ├── ADR-005-garantia-at-least-once-com-x-event-id.md
        ├── ADR-006-reuso-dos-padroes-existentes.md
        └── ADR-007-snapshot-do-payload-na-insercao.md
```

O código da aplicação (`src/`, `prisma/`, `tests/` e configurações) **não foi alterado** — serviu apenas de contexto e referência.

### Ordem sugerida de leitura

1. **`docs/PRD.md`** — comece pelo problema e pelo escopo. É o documento que responde "por que isso existe" e, principalmente, "o que *não* está incluído". Se você ler só um documento, leia este.
2. **`docs/RFC.md`** — a proposta técnica em nível de arquitetura, com as alternativas que foram descartadas e o que ficou em aberto. É o resumo mais eficiente das decisões.
3. **`docs/adrs/`** — as decisões isoladas, uma por arquivo. Leia o `README.md` do diretório primeiro: ele tem o índice e o mapa de qual ADR cobre qual decisão principal. Comece pelo ADR-001 (outbox), que é a decisão da qual todas as outras dependem.
4. **`docs/FDD.md`** — o detalhamento de implementação. É o documento que um desenvolvedor deve ter aberto ao codar. A seção 11 ("Integração com o sistema existente") é a mais importante para quem já conhece a codebase.
5. **`docs/TRACKER.md`** — use como ferramenta de auditoria, não de leitura linear. Se alguma afirmação dos outros documentos parecer inventada, procure o item aqui: se não houver linha com origem identificável, desconfie.

### Leitura para perfis específicos

- **Quem vai implementar:** FDD → ADRs 001, 002, 003 → FDD §11.
- **Quem vai revisar tecnicamente:** RFC → ADRs → FDD §5 e §6.
- **Quem vai revisar segurança:** ADR-004 → FDD §4.8 e §5 → PRD §7.
- **Quem é de produto/negócio:** PRD §1 a §5 → PRD §10.

### Verificação

Antes do fechamento, a checklist de critérios de aceite do enunciado foi percorrida item a item, e toda afirmação quantitativa e todo caminho de arquivo citado foram conferidos por comando contra o repositório — não estimados.

Os **14 caminhos distintos com origem `CODIGO`** no tracker e os **26 caminhos da tabela consolidada** (seção 5 do tracker) existem e foram verificados um a um. Também foi verificado que **todo ID do tracker aparece literalmente no documento que referencia** (238 de 238).

A única exceção são os **três arquivos que a feature cria** e que, por definição, ainda não existem no repositório: `src/worker.ts`, `src/modules/webhooks/webhook.worker.ts` e `src/modules/webhooks/webhook.schemas.ts`. Eles aparecem no FDD §11 com uma marcação explícita de "arquivo novo a ser criado", justamente para que nenhum leitor — humano ou IA — os confunda com arquivos existentes da aplicação. Todo o restante do pacote referencia apenas código que existe.
