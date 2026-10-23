# Da Reunião ao Documento: Design Docs Gerados por IA

Pacote de design docs do **Sistema de Webhooks de Notificação de Pedidos** de um Order Management System (OMS), produzido a partir da transcrição de uma reunião técnica ([`TRANSCRICAO.md`](TRANSCRICAO.md)) e do código existente. O enunciado original do desafio está no [repositório base](https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia).

> Entrega **puramente documental**: nada em `src/`, `prisma/`, `tests/` ou nas configurações foi alterado.

## Sobre o desafio

Uma empresa que opera um OMS em produção decidiu, em uma reunião de cerca de 55 minutos entre tech lead, PM, engenheiros e segurança, construir webhooks de saída para avisar clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) quando o status de um pedido muda. A decisão técnica existia apenas na transcrição da call. A tarefa foi transformá-la, junto com o código da aplicação, em documentação acionável: PRD, RFC, FDD, ADRs e um Tracker de rastreabilidade, cada um em sua altura (produto, arquitetura, decisão pontual, implementação e origem).

A regra central é que **tudo precisa ser rastreável** à transcrição ou ao código. Por isso o trabalho teve dois focos: separar o que a reunião decidiu do que descartou ou adiou (e-mail, rate limiting, dashboard, arquivamento, exactly-once, ordering global), e marcar explicitamente qualquer detalhe que o autor teve de completar por conta própria.

## Ferramentas de IA utilizadas

| Ferramenta | Papel |
| --- | --- |
| **Claude Code** (Claude Sonnet 5.5, app desktop) | Leitura do código e da transcrição, extração e classificação das decisões, redação dos documentos, geração do Tracker e execução das verificações. |
| **Skill `superpowers:brainstorming`** | Disciplina de escopo antes de produzir: o enunciado já definia o escopo por completo, então não houve rodada de perguntas; a skill apenas confirmou que a próxima etapa era produzir, não perguntar. |
| **Scripts Python (escritos pela IA)** | `build_tracker.py` gera `docs/TRACKER.md` a partir de uma tabela de itens; `validate.py` audita o pacote (ver *Iterações*). Ficaram fora do repositório por serem ferramentas de apoio, não entrega. |

## Workflow adotado

A ordem seguiu a sugerida no enunciado, de baixo para cima na "altura" dos documentos:

1. **Contextualização.** Leitura integral da transcrição e dos pontos do código citados na reunião: `order.service.ts` (`changeStatus`), `order.status.ts`, classes de erro, `auth.middleware.ts` (`requireRole`), `error.middleware.ts`, logger Pino, `server.ts`, `env.ts`, `schema.prisma`, rotas e testes.
2. **Filtragem da transcrição**, com prompt dirigido, em sete grupos: decisões fechadas, requisitos funcionais, restrições, ganchos com o código, descartado, adiado e detalhes secundários. Cada item recebeu `[hh:mm] Falante`.
3. **ADRs primeiro (7).** Outbox, worker em polling, retry+DLQ, HMAC, at-least-once, reuso de padrões e contrato de entrega. As seis decisões principais do enunciado estão cobertas; o sétimo ADR reúne decisões secundárias (payload, headers, timeout, limite de 64 KB).
4. **RFC** (~1.300 palavras): consolida a proposta, as 6 alternativas descartadas, 8 questões em aberto e linka os 7 ADRs.
5. **FDD**: fluxos, modelo de dados, 8 contratos, matriz de erros `WEBHOOK_*`, resiliência, observabilidade e a seção obrigatória de integração (17 arquivos reais do código).
6. **PRD por último**, já como consolidação de RFC, FDD e ADRs.
7. **Tracker** gerado por script a partir de uma tabela de itens, depois validado contra a transcrição e o repositório.
8. **README** (este arquivo), por último.

Regra de trabalho: o conteúdo que a reunião não fechou é marcado no próprio documento como *decisão de design* (FDD) ou *análise do autor* (ADR/RFC), nunca apresentado como decisão da reunião.

## Prompts customizados

Os dois prompts abaixo conduziram as etapas de maior risco de alucinação. Estão adaptados e reescritos aqui de forma legível.

**1. Filtragem dirigida da transcrição (o que entra e o que não entra)**

```text
Leia TRANSCRICAO.md inteira. Classifique cada ponto relevante em exatamente um grupo:
DECISAO_FECHADA, REQUISITO_FUNCIONAL, REQUISITO_NAO_FUNCIONAL, RESTRICAO,
GANCHO_COM_CODIGO, DESCARTADO, ADIADO_OU_EM_ABERTO, DETALHE_SECUNDARIO.
Para cada ponto devolva: grupo, resumo em uma linha, timestamp [hh:mm] e nome de quem falou.
Regras:
- Só use o que está escrito. Se algo parece óbvio mas ninguém disse, NÃO inclua.
- Para DESCARTADO e ADIADO_OU_EM_ABERTO, cite a fala que descartou/adiou e o motivo dado.
- Quando duas falas se contradizem ou ficam ambíguas (ex.: "5 tentativas" vs. lista de intervalos),
  liste em AMBIGUIDADES com as duas falas, sem escolher um lado.
- Itens como e-mail, rate limiting, dashboard e arquivamento devem aparecer como fora do escopo.
```

**2. Auditoria de rastreabilidade (anti-alucinação)**

```text
Para cada item com ID (PRD-*, RFC-*, FDD-*, ADR-*) nos documentos de docs/, confirme:
1) existe uma linha em docs/TRACKER.md com esse ID;
2) se Fonte = TRANSCRICAO, o timestamp existe em TRANSCRICAO.md e o falante citado de fato falou naquele horário;
3) se Fonte = CODIGO, o caminho existe no repositório;
4) nenhum item registrado como requisito foi descartado ou adiado na reunião.
Todo caminho de arquivo citado no texto dos documentos precisa existir, exceto arquivos
explicitamente planejados (src/worker.ts, src/modules/webhooks/*).
Se não encontrar a origem de um item, não complete: marque-o como "sem origem" para eu remover ou reclassificar
como decisão de design.
```

O segundo prompt foi implementado como o script `validate.py` (determinístico, em vez de depender do "olhômetro" do modelo). Antes de confiar nele, um *controle negativo* injetou dois erros propositais (falante errado e arquivo inexistente) e o script os detectou.

## Iterações e ajustes

O pacote passou por **três ciclos principais**: geração em cascata (ADRs → RFC → FDD → PRD → Tracker), revisão com ajustes e uma revisão crítica final. Os principais momentos em que o primeiro rascunho precisou ser corrigido ou questionado:

1. **Leitura do PDF do desafio falhou** (a ferramenta de renderização de páginas não estava instalada). O contorno foi extrair o texto com Python e conferi-lo com o enunciado presente no `README.md` original. Nenhum requisito foi inferido sem o texto.
2. **"5 tentativas" é ambíguo.** A lista tem 5 intervalos (1m/5m/30m/2h/12h) e Diego fala em "quase 15 horas entre primeira falha e última tentativa", o que só fecha com 1 envio inicial + 5 retentativas. Em vez de escolher em silêncio, a interpretação ficou declarada no ADR-003 e no FDD e foi aberta como questão **RFC-Q-05** para a revisão.
3. **Conflito entre `WEBHOOK_INVALID_URL` e a validação Zod.** Sofia disse que a regra de `https` é "só uma validação no schema Zod", mas Bruno nomeou o código `WEBHOOK_INVALID_URL`. Lendo `src/middlewares/validate.middleware.ts`, constatou-se que todo `ZodError` vira o genérico `VALIDATION_ERROR`. A regra `https` foi para o service para poder emitir o código do módulo; a justificativa está no FDD (FDD-INT-06).
4. **Bloqueio de cabeça de fila no worker.** O formato mais simples de worker (lote processado em sequência) falha com timeout de 10 s: um cliente lento violaria o requisito de < 10 s para todos os outros. O FDD paraleliza entre webhooks distintos e manter a sequência dentro do mesmo webhook (marcado como decisão de design e risco FDD-RISK-01).
5. **Ordem de eventos × retry.** A reunião garantiu ordem "por `order_id`", mas um evento em retry pode ser ultrapassado por um mais novo do mesmo pedido. Isso não foi discutido; entrou como análise do autor no ADR-002 e como **RFC-Q-06**.
6. **Lacunas não discutidas na reunião, expostas em vez de preenchidas:** proteção em repouso da secret (precisa ser recuperável para assinar), formato de `X-Signature` durante a rotação, posse do `customerId` no cadastro e risco de SSRF na URL. Foram levados à revisão de segurança de Sofia (RFC-Q-07, FDD-RISK-06/07).
7. **Meta quantitativa do PRD corrigida.** O rascunho dizia "3 de 3 clientes até o fim de novembro", mas só a Atlas citou novembro ([09:45] Marcos). O texto agora distingue as duas coisas.
8. **Observabilidade sem origem.** A reunião não tratou de métricas nem tracing. Em vez de inventar stack, o FDD propõe métricas por log/SQL e *tracing por correlação* via `event_id`, marcado como decisão de design.
9. **Validação do pacote:** a auditoria automática encerrou com 0 inconsistências (timestamps, falantes e caminhos conferidos).

**Terceiro ciclo: revisão crítica pela própria IA, depois de tudo publicado.** A IA releu os documentos como um revisor externo, cruzando com o código, e achou três problemas que a validação automática não pega, por serem de coerência e não de existência de origem:

10. **Exclusão de customer.** O FDD criava a FK `webhooks.customerId` sem tratar que `CustomerRepository.delete` faz `prisma.customer.delete` direto; excluir um customer com webhooks cairia em erro do Prisma sem tratamento (500). Foi definido `ON DELETE CASCADE` e criado o item FDD-INT-17.
11. **Inconsistência sobre "5 tentativas".** Apesar de a contagem estar aberta (RFC-Q-05), o PRD e o resumo do RFC ainda a afirmavam como fato. Os dois passaram a remeter à questão em aberto.
12. **Assinatura × retry.** O FDD não dizia que o corpo deve ser serializado uma única vez e que os mesmos bytes são assinados e enviados, nem diferenciava `timestamp` do payload (momento do evento) de `X-Timestamp` (momento de cada envio). Ambos foram especificados.

Estado final: **3 ciclos** (geração, revisão com ajustes e revisão crítica), 225 linhas no Tracker (86% `TRANSCRICAO`, 31 `CODIGO`), 0 inconsistências na auditoria. Até aqui as iterações foram conduzidas pela IA; feedback humano sobre o conteúdo ainda não foi incorporado.

## Como navegar a entrega

Ordem sugerida de leitura (do "porquê" ao "como"):

1. [`docs/PRD.md`](docs/PRD.md): problema, público, escopo, requisitos e métricas.
2. [`docs/RFC.md`](docs/RFC.md): proposta técnica, alternativas descartadas e questões em aberto.
3. [`docs/adrs/`](docs/adrs/README.md): uma decisão por arquivo.
   - [ADR-001 Outbox no MySQL](docs/adrs/ADR-001-outbox-no-mysql.md)
   - [ADR-002 Worker separado em polling](docs/adrs/ADR-002-worker-separado-em-polling.md)
   - [ADR-003 Retry, backoff e DLQ](docs/adrs/ADR-003-retry-backoff-e-dlq.md)
   - [ADR-004 HMAC-SHA256 com secret por endpoint](docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md)
   - [ADR-005 At-least-once com X-Event-Id](docs/adrs/ADR-005-at-least-once-com-x-event-id.md)
   - [ADR-006 Reuso dos padrões do projeto](docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md)
   - [ADR-007 Contrato de entrega](docs/adrs/ADR-007-contrato-de-entrega-payload-e-headers.md)
4. [`docs/FDD.md`](docs/FDD.md): especificação de implementação (fluxos, contratos, erros, integração com o código).
5. [`docs/TRACKER.md`](docs/TRACKER.md): origem de cada item na transcrição ou no código.

```
.
├── README.md
├── TRANSCRICAO.md          (não alterado)
├── docs/
│   ├── PRD.md
│   ├── RFC.md
│   ├── FDD.md
│   ├── TRACKER.md
│   └── adrs/               (README + ADR-001 … ADR-007)
└── src/ prisma/ tests/     (não alterados)
```
