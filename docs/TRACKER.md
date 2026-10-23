# Tracker de Rastreabilidade

Mapeia cada item registrado em `docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md` e nos ADRs à sua origem na `TRANSCRICAO.md` (timestamp + falante) ou no código-fonte (caminho do arquivo).

**Como ler**

- **ID**: o identificador aparece no próprio documento (PRD/RFC/FDD) ou é o número do ADR; IDs com sufixo (`ADR-001-SNAPSHOT`) detalham uma decisão específica dentro do ADR.
- **Fonte / Localização**: a origem *principal* do item. Quando o item combina mais de uma fala, a tabela registra a que o fundamenta de forma mais direta.
- Itens que o documento marca como *decisão de design* ou *análise do autor* (detalhes que a reunião não fechou) apontam para a fala ou para o arquivo de código que os motivou; o texto do documento explicita que a escolha é do autor.
- Tudo que a reunião **descartou ou adiou** está identificado como `Fora de escopo`, `Alternativa descartada` ou `Questão em aberto`.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-CEN-01 | `docs/PRD.md` | Cenário de uso | Atlas cadastra webhook para ouvir apenas SHIPPED e DELIVERED | TRANSCRICAO | [09:33] Marcos |
| PRD-CEN-02 | `docs/PRD.md` | Cenário de uso | Cliente fora do ar por manutenção de 2h; plataforma retenta por ~15h | TRANSCRICAO | [09:16] Diego |
| PRD-CEN-03 | `docs/PRD.md` | Cenário de uso | Cliente recebe evento duplicado e deduplica por event_id | TRANSCRICAO | [09:25] Diego |
| PRD-CEN-04 | `docs/PRD.md` | Cenário de uso | Secret vazada: cliente rotaciona e mantém a antiga válida por 24h | TRANSCRICAO | [09:21] Sofia |
| PRD-CEN-05 | `docs/PRD.md` | Cenário de uso | Cliente consulta histórico dos últimos envios para diagnóstico | TRANSCRICAO | [09:34] Marcos |
| PRD-CEN-06 | `docs/PRD.md` | Cenário de uso | Evento esgota tentativas, vai à DLQ e ADMIN o reenvia | TRANSCRICAO | [09:18] Diego |
| PRD-OBJ-01 | `docs/PRD.md` | Objetivo | 3 de 3 clientes B2B com webhook em produção até o fim de novembro | TRANSCRICAO | [09:45] Marcos |
| PRD-OBJ-02 | `docs/PRD.md` | Objetivo | Entrega em menos de 10s após o commit do novo status | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-03 | `docs/PRD.md` | Objetivo | Zero mudanças de status sem evento correspondente | TRANSCRICAO | [09:40] Bruno |
| PRD-OBJ-04 | `docs/PRD.md` | Objetivo | Entregar a feature em 3 sprints incluindo revisão de segurança | TRANSCRICAO | [09:46] Larissa |
| PRD-OBJ-05 | `docs/PRD.md` | Objetivo | Reduzir o polling de GET /orders pelos clientes (meta a definir) | TRANSCRICAO | [09:00] Marcos |
| PRD-ESC-01 | `docs/PRD.md` | Escopo incluso | Notificação de saída a cada mudança de status de pedido | TRANSCRICAO | [09:02] Marcos |
| PRD-ESC-02 | `docs/PRD.md` | Escopo incluso | API de cadastro, listagem, edição e remoção de webhooks | TRANSCRICAO | [09:33] Bruno |
| PRD-ESC-03 | `docs/PRD.md` | Escopo incluso | Consulta ao histórico de entregas do webhook | TRANSCRICAO | [09:34] Marcos |
| PRD-ESC-04 | `docs/PRD.md` | Escopo incluso | Rotação de secret pela API | TRANSCRICAO | [09:21] Sofia |
| PRD-ESC-05 | `docs/PRD.md` | Escopo incluso | Reenvio manual da DLQ restrito a ADMIN | TRANSCRICAO | [09:36] Sofia |
| PRD-FORA-01 | `docs/PRD.md` | Fora de escopo | E-mail ao cliente quando o webhook falha: adiado para próxima fase | TRANSCRICAO | [09:37] Larissa |
| PRD-FORA-02 | `docs/PRD.md` | Fora de escopo | Rate limiting de saída: observar e decidir depois | TRANSCRICAO | [09:39] Larissa |
| PRD-FORA-03 | `docs/PRD.md` | Fora de escopo | Dashboard visual descartado; só endpoints | TRANSCRICAO | [09:40] Larissa |
| PRD-FORA-04 | `docs/PRD.md` | Fora de escopo | Arquivamento de eventos entregues após ~30 dias fora da feature | TRANSCRICAO | [09:08] Diego |
| PRD-FORA-05 | `docs/PRD.md` | Fora de escopo | Ordering global e múltiplos workers: limitação conhecida | TRANSCRICAO | [09:13] Larissa |
| PRD-FORA-06 | `docs/PRD.md` | Fora de escopo | Exactly-once descartado em favor de at-least-once | TRANSCRICAO | [09:25] Diego |
| PRD-FORA-07 | `docs/PRD.md` | Fora de escopo | Webhooks de entrada (cliente → plataforma) fora de escopo | TRANSCRICAO | [09:02] Marcos |
| PRD-FR-01 | `docs/PRD.md` | Requisito Funcional | Cadastrar webhook com URL e lista de status; secret gerada e devolvida na criação | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-02 | `docs/PRD.md` | Requisito Funcional | customer_id vem no body/path, não do JWT | TRANSCRICAO | [09:32] Larissa |
| PRD-FR-03 | `docs/PRD.md` | Requisito Funcional | Listar webhooks de um customer | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | `docs/PRD.md` | Requisito Funcional | Editar webhook (PATCH) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | `docs/PRD.md` | Requisito Funcional | Remover webhook (DELETE) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-06 | `docs/PRD.md` | Requisito Funcional | Filtro de eventos por status, aplicado na inserção da outbox | TRANSCRICAO | [09:34] Bruno |
| PRD-FR-07 | `docs/PRD.md` | Requisito Funcional | Evento enviado a cada mudança elegível com payload enxuto, sem items | TRANSCRICAO | [09:43] Diego |
| PRD-FR-08 | `docs/PRD.md` | Requisito Funcional | Histórico de entregas (últimos 100) com sucesso/falha, payload, response e tempo | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-09 | `docs/PRD.md` | Requisito Funcional | Rotação de secret com grace period de 24h | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-10 | `docs/PRD.md` | Requisito Funcional | Envio assinado com HMAC-SHA256 e headers de identificação | TRANSCRICAO | [09:44] Diego |
| PRD-FR-11 | `docs/PRD.md` | Requisito Funcional | Identificador único de evento estável entre reenvios | TRANSCRICAO | [09:25] Diego |
| PRD-FR-12 | `docs/PRD.md` | Requisito Funcional | Retry automático com backoff 1m/5m/30m/2h/12h e DLQ | TRANSCRICAO | [09:17] Larissa |
| PRD-FR-13 | `docs/PRD.md` | Requisito Funcional | Replay manual por ADMIN com registro do autor | TRANSCRICAO | [09:36] Sofia |
| PRD-FR-14 | `docs/PRD.md` | Requisito Funcional | Somente URLs https aceitas no cadastro | TRANSCRICAO | [09:23] Sofia |
| PRD-FR-15 | `docs/PRD.md` | Requisito Funcional | CRUD para qualquer role autenticada; replay exige ADMIN | TRANSCRICAO | [09:37] Sofia |
| PRD-NFR-01 | `docs/PRD.md` | Requisito Não Funcional | Latência de entrega < 10s com polling de 2s | TRANSCRICAO | [09:10] Larissa |
| PRD-NFR-02 | `docs/PRD.md` | Requisito Não Funcional | Mudança de status e registro do evento são atômicos | TRANSCRICAO | [09:40] Bruno |
| PRD-NFR-03 | `docs/PRD.md` | Requisito Não Funcional | Cliente lento/offline não trava nem reverte mudanças de status | TRANSCRICAO | [09:04] Bruno |
| PRD-NFR-04 | `docs/PRD.md` | Requisito Não Funcional | Semântica de entrega at-least-once | TRANSCRICAO | [09:24] Diego |
| PRD-NFR-05 | `docs/PRD.md` | Requisito Não Funcional | Segurança: HMAC-SHA256, secret por endpoint, TLS obrigatório | TRANSCRICAO | [09:21] Sofia |
| PRD-NFR-06 | `docs/PRD.md` | Requisito Não Funcional | Payload máximo de 64 KB; acima disso, erro | TRANSCRICAO | [09:24] Larissa |
| PRD-NFR-07 | `docs/PRD.md` | Requisito Não Funcional | Timeout de 10s por chamada ao cliente | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-08 | `docs/PRD.md` | Requisito Não Funcional | Auditoria: replay registra quem executou | TRANSCRICAO | [09:36] Sofia |
| PRD-NFR-09 | `docs/PRD.md` | Requisito Não Funcional | Sem infraestrutura nova; reuso de MySQL, Prisma, Pino e padrões | TRANSCRICAO | [09:30] Larissa |
| PRD-NFR-10 | `docs/PRD.md` | Restrição | Ordem garantida só por pedido e com worker único | TRANSCRICAO | [09:13] Larissa |
| PRD-DEC-01 | `docs/PRD.md` | Decisão | Outbox no MySQL na mesma transação do pedido | TRANSCRICAO | [09:08] Larissa |
| PRD-DEC-02 | `docs/PRD.md` | Decisão | Worker separado com polling de 2s | TRANSCRICAO | [09:10] Larissa |
| PRD-DEC-03 | `docs/PRD.md` | Decisão | Retry com backoff (5 tentativas) e DLQ com replay manual | TRANSCRICAO | [09:17] Larissa |
| PRD-DEC-04 | `docs/PRD.md` | Decisão | HMAC-SHA256 com secret por endpoint e rotação de 24h | TRANSCRICAO | [09:22] Sofia |
| PRD-DEC-05 | `docs/PRD.md` | Decisão | At-least-once com X-Event-Id | TRANSCRICAO | [09:26] Larissa |
| PRD-DEC-06 | `docs/PRD.md` | Decisão | Reuso máximo dos padrões do projeto | TRANSCRICAO | [09:30] Larissa |
| PRD-DEC-07 | `docs/PRD.md` | Decisão | Payload enxuto sem items | TRANSCRICAO | [09:43] Diego |
| PRD-DEP-01 | `docs/PRD.md` | Dependência | MySQL/Prisma existentes e módulo de pedidos (changeStatus) | TRANSCRICAO | [09:07] Diego |
| PRD-DEP-02 | `docs/PRD.md` | Dependência | Revisão de segurança de Sofia (≥ 2 dias úteis) antes do deploy | TRANSCRICAO | [09:46] Sofia |
| PRD-DEP-03 | `docs/PRD.md` | Dependência | Documentação no portal de desenvolvedor feita por Marcos | TRANSCRICAO | [09:26] Marcos |
| PRD-DEP-04 | `docs/PRD.md` | Dependência | Operar um segundo processo (worker) em produção | TRANSCRICAO | [09:11] Diego |
| PRD-DEP-05 | `docs/PRD.md` | Dependência | Clientes implementam HTTPS, validação de HMAC e deduplicação | TRANSCRICAO | [09:25] Diego |
| PRD-DEP-06 | `docs/PRD.md` | Dependência | Marcos confirma o prazo com a Atlas | TRANSCRICAO | [09:47] Marcos |
| PRD-RISK-01 | `docs/PRD.md` | Risco | Perder a data da Atlas e o cliente migrar para o concorrente | TRANSCRICAO | [09:00] Marcos |
| PRD-RISK-02 | `docs/PRD.md` | Risco | Cliente não deduplica e processa duas vezes | TRANSCRICAO | [09:25] Sofia |
| PRD-RISK-03 | `docs/PRD.md` | Risco | Vazamento de secret de cliente | TRANSCRICAO | [09:22] Diego |
| PRD-RISK-04 | `docs/PRD.md` | Risco | Retries em janelas longas para clientes indisponíveis | TRANSCRICAO | [09:16] Diego |
| PRD-RISK-05 | `docs/PRD.md` | Risco | Eventos fora de ordem para o mesmo pedido | TRANSCRICAO | [09:12] Diego |
| PRD-ACE-01 | `docs/PRD.md` | Critério de aceitação | Cadastro https com status de interesse; URL http recusada | TRANSCRICAO | [09:23] Sofia |
| PRD-ACE-02 | `docs/PRD.md` | Critério de aceitação | Evento recebido em < 10s com assinatura válida e headers | TRANSCRICAO | [09:02] Marcos |
| PRD-ACE-03 | `docs/PRD.md` | Critério de aceitação | Status não ouvido não gera evento | TRANSCRICAO | [09:34] Bruno |
| PRD-ACE-04 | `docs/PRD.md` | Critério de aceitação | Atomicidade entre status e evento nos dois sentidos | TRANSCRICAO | [09:40] Bruno |
| PRD-ACE-05 | `docs/PRD.md` | Critério de aceitação | Retentativas e movimento para DLQ ao esgotar | TRANSCRICAO | [09:17] Larissa |
| PRD-ACE-06 | `docs/PRD.md` | Critério de aceitação | ADMIN reenvia; OPERATOR recusado; autor registrado | TRANSCRICAO | [09:36] Sofia |
| PRD-ACE-07 | `docs/PRD.md` | Critério de aceitação | Histórico com resultado, payload, resposta e tempo | TRANSCRICAO | [09:34] Marcos |
| PRD-ACE-08 | `docs/PRD.md` | Critério de aceitação | Rotação: duas secrets válidas por 24h, depois só a nova | TRANSCRICAO | [09:21] Sofia |
| PRD-ACE-09 | `docs/PRD.md` | Critério de aceitação | Reuso dos padrões do projeto e revisão de segurança concluída | TRANSCRICAO | [09:30] Larissa |
| PRD-TEST-01 | `docs/PRD.md` | Estratégia de teste | Unitários: backoff, HMAC, payload, filtro, URL https, limite 64 KB | CODIGO | `tests/helpers/factories.ts` |
| PRD-TEST-02 | `docs/PRD.md` | Estratégia de teste | Integração de API com Supertest e envelope de erros | CODIGO | `tests/orders.test.ts` |
| PRD-TEST-03 | `docs/PRD.md` | Estratégia de teste | Integração transacional de changeStatus e limpeza de tabelas de teste | CODIGO | `tests/setup.ts` |
| PRD-TEST-04 | `docs/PRD.md` | Estratégia de teste | Worker contra servidor HTTP local: sucesso, timeout, 5xx, retry, DLQ | TRANSCRICAO | [09:15] Diego |
| PRD-TEST-05 | `docs/PRD.md` | Estratégia de teste | Revisão de código de Sofia e confirmação de prazo por Marcos | TRANSCRICAO | [09:46] Sofia |
| RFC-PROP-01 | `docs/RFC.md` | Decisão | Outbox no MySQL na mesma transação, snapshot e filtro na inserção | TRANSCRICAO | [09:06] Diego |
| RFC-PROP-02 | `docs/RFC.md` | Decisão | Worker separado em polling de 2s com src/worker.ts | TRANSCRICAO | [09:11] Larissa |
| RFC-PROP-03 | `docs/RFC.md` | Decisão | Retry exponencial, timeout de 10s e DLQ com replay ADMIN | TRANSCRICAO | [09:17] Larissa |
| RFC-PROP-04 | `docs/RFC.md` | Decisão | HMAC-SHA256 com secret por endpoint, rotação 24h e HTTPS | TRANSCRICAO | [09:22] Sofia |
| RFC-PROP-05 | `docs/RFC.md` | Decisão | At-least-once com X-Event-Id e dedup no cliente | TRANSCRICAO | [09:26] Larissa |
| RFC-PROP-06 | `docs/RFC.md` | Decisão | Reuso dos padrões: módulo, AppError, WEBHOOK_, Pino, requireRole | TRANSCRICAO | [09:30] Larissa |
| RFC-PROP-07 | `docs/RFC.md` | Decisão | Contrato de entrega: payload sem items, headers e limite de 64 KB | TRANSCRICAO | [09:44] Diego |
| RFC-ALT-01 | `docs/RFC.md` | Alternativa descartada | Disparo HTTP síncrono no OrderService | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | `docs/RFC.md` | Alternativa descartada | Fila externa (Redis Streams) por ser overengineering | TRANSCRICAO | [09:07] Diego |
| RFC-ALT-03 | `docs/RFC.md` | Alternativa descartada | Trigger de banco para acionar o worker | TRANSCRICAO | [09:09] Diego |
| RFC-ALT-04 | `docs/RFC.md` | Alternativa descartada | Retry indefinido ou apenas 3 tentativas | TRANSCRICAO | [09:16] Diego |
| RFC-ALT-05 | `docs/RFC.md` | Alternativa descartada | Exactly-once por exigir coordenação dos dois lados | TRANSCRICAO | [09:25] Diego |
| RFC-ALT-06 | `docs/RFC.md` | Alternativa descartada | Secret global da plataforma | TRANSCRICAO | [09:21] Sofia |
| RFC-Q-01 | `docs/RFC.md` | Questão em aberto | Rate limiting de saída por cliente | TRANSCRICAO | [09:39] Larissa |
| RFC-Q-02 | `docs/RFC.md` | Questão em aberto | Escala do worker: particionar por order_id ou lock pessimista | TRANSCRICAO | [09:13] Diego |
| RFC-Q-03 | `docs/RFC.md` | Questão em aberto | Aviso por e-mail quando o webhook falha repetidamente | TRANSCRICAO | [09:37] Larissa |
| RFC-Q-04 | `docs/RFC.md` | Questão em aberto | Arquivamento de linhas entregues da outbox (~30 dias) | TRANSCRICAO | [09:08] Diego |
| RFC-Q-05 | `docs/RFC.md` | Questão em aberto | Interpretação de '5 tentativas' (1 envio + 5 retentativas) | TRANSCRICAO | [09:17] Diego |
| RFC-Q-06 | `docs/RFC.md` | Questão em aberto | Ordem entre eventos do mesmo pedido durante retry (análise do autor) | TRANSCRICAO | [09:12] Diego |
| RFC-Q-07 | `docs/RFC.md` | Questão em aberto | Proteção em repouso da secret e formato de X-Signature na rotação | TRANSCRICAO | [09:46] Sofia |
| RFC-Q-08 | `docs/RFC.md` | Questão em aberto | Endurecer permissões do CRUD de webhook no futuro | TRANSCRICAO | [09:37] Sofia |
| RFC-RISK-01 | `docs/RFC.md` | Risco | INSERT extra aumenta a duração da transação de changeStatus | TRANSCRICAO | [09:04] Bruno |
| RFC-RISK-02 | `docs/RFC.md` | Risco | Cliente não deduplica e processa o evento duas vezes | TRANSCRICAO | [09:26] Marcos |
| RFC-RISK-03 | `docs/RFC.md` | Risco | Vazamento de secret de cliente | TRANSCRICAO | [09:22] Diego |
| RFC-RISK-04 | `docs/RFC.md` | Risco | Prazo de 3 sprints versus data prometida à Atlas | TRANSCRICAO | [09:45] Marcos |
| FDD-OBJ-01 | `docs/FDD.md` | Objetivo técnico | Evento gravado na mesma transação de changeStatus | TRANSCRICAO | [09:40] Bruno |
| FDD-OBJ-02 | `docs/FDD.md` | Objetivo técnico | Entrega em < 10s após o commit | TRANSCRICAO | [09:02] Marcos |
| FDD-OBJ-03 | `docs/FDD.md` | Objetivo técnico | At-least-once com X-Event-Id estável | TRANSCRICAO | [09:24] Diego |
| FDD-OBJ-04 | `docs/FDD.md` | Objetivo técnico | HMAC-SHA256 com secret por endpoint e rotação | TRANSCRICAO | [09:22] Sofia |
| FDD-OBJ-05 | `docs/FDD.md` | Objetivo técnico | Sem infraestrutura ou dependência nova | TRANSCRICAO | [09:07] Diego |
| FDD-ESC-01 | `docs/FDD.md` | Exclusão | Webhooks de entrada fora do escopo | TRANSCRICAO | [09:02] Marcos |
| FDD-ESC-02 | `docs/FDD.md` | Exclusão | Aviso por e-mail fora do escopo | TRANSCRICAO | [09:37] Larissa |
| FDD-ESC-03 | `docs/FDD.md` | Exclusão | Rate limiting de saída fora do escopo | TRANSCRICAO | [09:39] Larissa |
| FDD-ESC-04 | `docs/FDD.md` | Exclusão | Dashboard visual fora do escopo | TRANSCRICAO | [09:40] Larissa |
| FDD-ESC-05 | `docs/FDD.md` | Exclusão | Arquivamento da outbox fora do escopo | TRANSCRICAO | [09:08] Diego |
| FDD-ESC-06 | `docs/FDD.md` | Exclusão | Múltiplos workers e ordering global fora do escopo | TRANSCRICAO | [09:13] Larissa |
| FDD-ESC-07 | `docs/FDD.md` | Exclusão | create() de pedido não gera evento; só changeStatus | TRANSCRICAO | [09:40] Bruno |
| FDD-DATA-01 | `docs/FDD.md` | Modelo de dados | Tabela webhooks: url, secret, customer_id, ativo, eventos | TRANSCRICAO | [09:21] Bruno |
| FDD-DATA-02 | `docs/FDD.md` | Modelo de dados | Tabela webhook_outbox com status e índices em status/created_at | TRANSCRICAO | [09:08] Diego |
| FDD-DATA-03 | `docs/FDD.md` | Modelo de dados | Tabela de tentativas para histórico de entregas | TRANSCRICAO | [09:34] Marcos |
| FDD-DATA-04 | `docs/FDD.md` | Modelo de dados | Tabela webhook_dead_letter com payload, motivo e timestamp | TRANSCRICAO | [09:18] Diego |
| FDD-FLUXO-01 | `docs/FDD.md` | Fluxo | Criação do evento via publishWebhookEvent(tx, order, from, to) | TRANSCRICAO | [09:41] Bruno |
| FDD-FLUXO-02 | `docs/FDD.md` | Fluxo | Worker em polling de 2s, lote pequeno, mais antigos primeiro | TRANSCRICAO | [09:09] Diego |
| FDD-FLUXO-03 | `docs/FDD.md` | Fluxo | Retry com backoff 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Diego |
| FDD-FLUXO-04 | `docs/FDD.md` | Fluxo | Falha permanente movida para webhook_dead_letter | TRANSCRICAO | [09:18] Diego |
| FDD-FLUXO-05 | `docs/FDD.md` | Fluxo | Replay ADMIN recoloca o evento na outbox como pendente | TRANSCRICAO | [09:18] Diego |
| FDD-FLUXO-06 | `docs/FDD.md` | Fluxo | Rotação de secret com antiga válida por 24h | TRANSCRICAO | [09:21] Sofia |
| FDD-FLUXO-07 | `docs/FDD.md` | Fluxo | Recuperação de eventos PROCESSING na partida (decisão de design) | TRANSCRICAO | [09:24] Diego |
| FDD-CONTRATO-01 | `docs/FDD.md` | Contrato | POST /webhooks cria webhook e devolve a secret | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | `docs/FDD.md` | Contrato | GET /webhooks lista webhooks de um customer | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | `docs/FDD.md` | Contrato | PATCH /webhooks/:id edita o webhook | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | `docs/FDD.md` | Contrato | DELETE /webhooks/:id remove o webhook | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | `docs/FDD.md` | Contrato | GET /webhooks/:id/deliveries retorna histórico de entregas | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-06 | `docs/FDD.md` | Contrato | Rotação de secret pela API (caminho é decisão de design) | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-07 | `docs/FDD.md` | Contrato | POST /admin/webhooks/dead-letter/:id/replay (ADMIN) | TRANSCRICAO | [09:18] Diego |
| FDD-CONTRATO-08 | `docs/FDD.md` | Contrato | Requisição de entrega com headers X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id | TRANSCRICAO | [09:44] Diego |
| FDD-ERRO-01 | `docs/FDD.md` | Erro | WEBHOOK_NOT_FOUND (404) | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-02 | `docs/FDD.md` | Erro | WEBHOOK_INVALID_URL (400) para URL não https | TRANSCRICAO | [09:23] Sofia |
| FDD-ERRO-03 | `docs/FDD.md` | Erro | WEBHOOK_INVALID_EVENT (400): PENDING nunca é destino de transição | CODIGO | `src/modules/orders/order.status.ts` |
| FDD-ERRO-04 | `docs/FDD.md` | Erro | WEBHOOK_CUSTOMER_NOT_FOUND (404), análogo ao Customer not found do create | CODIGO | `src/modules/orders/order.service.ts` |
| FDD-ERRO-05 | `docs/FDD.md` | Erro | WEBHOOK_DEAD_LETTER_NOT_FOUND (404) | TRANSCRICAO | [09:18] Diego |
| FDD-ERRO-06 | `docs/FDD.md` | Erro | WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED (409) | TRANSCRICAO | [09:18] Diego |
| FDD-ERRO-07 | `docs/FDD.md` | Erro | WEBHOOK_SECRET_REQUIRED (worker, DLQ direto) | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-08 | `docs/FDD.md` | Erro | WEBHOOK_PAYLOAD_TOO_LARGE (> 64 KB) | TRANSCRICAO | [09:24] Larissa |
| FDD-ERRO-09 | `docs/FDD.md` | Erro | WEBHOOK_ENDPOINT_INACTIVE (webhook desativado após enfileirar) | TRANSCRICAO | [09:21] Bruno |
| FDD-ERRO-10 | `docs/FDD.md` | Erro | WEBHOOK_DELIVERY_TIMEOUT após 10s | TRANSCRICAO | [09:42] Diego |
| FDD-ERRO-11 | `docs/FDD.md` | Erro | WEBHOOK_DELIVERY_FAILED para resposta fora de 2xx | TRANSCRICAO | [09:15] Diego |
| FDD-ERRO-12 | `docs/FDD.md` | Erro | WEBHOOK_DELIVERY_NETWORK_ERROR para falha de rede | TRANSCRICAO | [09:04] Bruno |
| FDD-RES-01 | `docs/FDD.md` | Resiliência | Timeout de 10s por chamada | TRANSCRICAO | [09:42] Diego |
| FDD-RES-02 | `docs/FDD.md` | Resiliência | Retry e backoff exponencial até a DLQ | TRANSCRICAO | [09:17] Larissa |
| FDD-RES-03 | `docs/FDD.md` | Resiliência | Fallback: DLQ persistida e replay manual; sem e-mail | TRANSCRICAO | [09:18] Diego |
| FDD-RES-04 | `docs/FDD.md` | Resiliência | Falha de entrega não afeta changeStatus (desacoplamento por outbox) | TRANSCRICAO | [09:04] Bruno |
| FDD-RES-05 | `docs/FDD.md` | Resiliência | Paralelismo entre webhooks distintos no ciclo (decisão de design) | TRANSCRICAO | [09:42] Diego |
| FDD-RES-06 | `docs/FDD.md` | Resiliência | Recuperação de crash coberta pelo at-least-once | TRANSCRICAO | [09:24] Diego |
| FDD-RES-07 | `docs/FDD.md` | Resiliência | Shutdown graceful nos moldes de server.ts | CODIGO | `src/server.ts` |
| FDD-OBS-01 | `docs/FDD.md` | Observabilidade | Log webhook_event_enqueued via Pino | TRANSCRICAO | [09:29] Bruno |
| FDD-OBS-02 | `docs/FDD.md` | Observabilidade | Log de cada tentativa com status HTTP, duração e erro | TRANSCRICAO | [09:34] Marcos |
| FDD-OBS-03 | `docs/FDD.md` | Observabilidade | Logs de DLQ e replay registram o autor | TRANSCRICAO | [09:36] Sofia |
| FDD-OBS-04 | `docs/FDD.md` | Observabilidade | Log de rotação sem expor o valor das secrets | TRANSCRICAO | [09:22] Diego |
| FDD-OBS-05 | `docs/FDD.md` | Observabilidade | Métrica de backlog e idade do evento mais antigo | TRANSCRICAO | [09:02] Marcos |
| FDD-OBS-06 | `docs/FDD.md` | Observabilidade | Métricas de latência por tentativa e fim a fim | TRANSCRICAO | [09:02] Marcos |
| FDD-OBS-07 | `docs/FDD.md` | Observabilidade | Taxa de sucesso e tentativas por evento | TRANSCRICAO | [09:15] Diego |
| FDD-OBS-08 | `docs/FDD.md` | Observabilidade | Métricas de DLQ para decidir e-mail e rate limit | TRANSCRICAO | [09:37] Larissa |
| FDD-OBS-09 | `docs/FDD.md` | Observabilidade | Tracing por correlação via event_id | TRANSCRICAO | [09:25] Diego |
| FDD-INT-01 | `docs/FDD.md` | Integração | changeStatus chama publishWebhookEvent dentro da transação | CODIGO | `src/modules/orders/order.service.ts` |
| FDD-INT-02 | `docs/FDD.md` | Integração | Máquina de estados define quais status são eventos válidos | CODIGO | `src/modules/orders/order.status.ts` |
| FDD-INT-03 | `docs/FDD.md` | Integração | Erros WEBHOOK_* como subclasses em http-errors.ts | CODIGO | `src/shared/errors/http-errors.ts` |
| FDD-INT-04 | `docs/FDD.md` | Integração | AppError como base dos erros do módulo | CODIGO | `src/shared/errors/app-error.ts` |
| FDD-INT-05 | `docs/FDD.md` | Integração | authenticate e requireRole('ADMIN') nas rotas | CODIGO | `src/middlewares/auth.middleware.ts` |
| FDD-INT-06 | `docs/FDD.md` | Integração | validate() com schemas Zod do módulo | CODIGO | `src/middlewares/validate.middleware.ts` |
| FDD-INT-07 | `docs/FDD.md` | Integração | error.middleware.ts permanece sem alteração | CODIGO | `src/middlewares/error.middleware.ts` |
| FDD-INT-08 | `docs/FDD.md` | Integração | buildApiRouter monta /webhooks e /admin/webhooks | CODIGO | `src/routes/index.ts` |
| FDD-INT-09 | `docs/FDD.md` | Integração | buildControllers instancia o módulo de webhooks | CODIGO | `src/app.ts` |
| FDD-INT-10 | `docs/FDD.md` | Integração | server.ts como modelo para src/worker.ts e script npm run worker | CODIGO | `src/server.ts` |
| FDD-INT-11 | `docs/FDD.md` | Integração | Worker cria PrismaClient próprio via createPrismaClient() | CODIGO | `src/config/database.ts` |
| FDD-INT-12 | `docs/FDD.md` | Integração | Novas variáveis WEBHOOK_* validadas por Zod | CODIGO | `src/config/env.ts` |
| FDD-INT-13 | `docs/FDD.md` | Integração | Logger Pino reutilizado e redactPaths ampliado | CODIGO | `src/shared/logger/index.ts` |
| FDD-INT-14 | `docs/FDD.md` | Integração | paginated() nas listagens | CODIGO | `src/shared/http/response.ts` |
| FDD-INT-15 | `docs/FDD.md` | Integração | Novos modelos e migration no Prisma | CODIGO | `prisma/schema.prisma` |
| FDD-INT-16 | `docs/FDD.md` | Integração | Setup e factories de teste ganham tabelas webhook_* | CODIGO | `tests/setup.ts` |
| FDD-ACE-01 | `docs/FDD.md` | Critério de aceite | 1 linha de outbox por webhook elegível; erro no insert reverte o status | TRANSCRICAO | [09:40] Bruno |
| FDD-ACE-02 | `docs/FDD.md` | Critério de aceite | Sem webhook interessado, nada é inserido | TRANSCRICAO | [09:34] Bruno |
| FDD-ACE-03 | `docs/FDD.md` | Critério de aceite | Envio em ≤ 2s + chamada; total < 10s | TRANSCRICAO | [09:10] Larissa |
| FDD-ACE-04 | `docs/FDD.md` | Critério de aceite | Headers presentes e HMAC conferindo com o corpo | TRANSCRICAO | [09:44] Diego |
| FDD-ACE-05 | `docs/FDD.md` | Critério de aceite | Retentativas nos intervalos definidos e DLQ ao esgotar | TRANSCRICAO | [09:17] Larissa |
| FDD-ACE-06 | `docs/FDD.md` | Critério de aceite | Replay ADMIN com mesmo event_id; OPERATOR recebe 403; autor logado | TRANSCRICAO | [09:36] Sofia |
| FDD-ACE-07 | `docs/FDD.md` | Critério de aceite | Após rotação, duas secrets por 24h, depois só a nova | TRANSCRICAO | [09:21] Sofia |
| FDD-ACE-08 | `docs/FDD.md` | Critério de aceite | URL http recusada; payload > 64 KB não enviado | TRANSCRICAO | [09:24] Larissa |
| FDD-ACE-09 | `docs/FDD.md` | Critério de aceite | Reinício da API não interrompe entregas | TRANSCRICAO | [09:11] Diego |
| FDD-ACE-10 | `docs/FDD.md` | Critério de aceite | error.middleware.ts inalterado e erros no envelope padrão | CODIGO | `src/middlewares/error.middleware.ts` |
| FDD-RISK-01 | `docs/FDD.md` | Risco | Cliente lento atrasa a fila e viola < 10s | TRANSCRICAO | [09:42] Diego |
| FDD-RISK-02 | `docs/FDD.md` | Risco | Retry reordena eventos do mesmo pedido | TRANSCRICAO | [09:12] Diego |
| FDD-RISK-03 | `docs/FDD.md` | Risco | Secret recuperável pode vazar via banco/backup | TRANSCRICAO | [09:46] Sofia |
| FDD-RISK-04 | `docs/FDD.md` | Risco | Crescimento da outbox degrada o polling | TRANSCRICAO | [09:08] Diego |
| FDD-RISK-05 | `docs/FDD.md` | Risco | Remoção de webhook apaga histórico e DLQ via cascade | TRANSCRICAO | [09:18] Diego |
| FDD-RISK-06 | `docs/FDD.md` | Risco | Sem checagem de posse do customerId no cadastro | TRANSCRICAO | [09:32] Larissa |
| FDD-RISK-07 | `docs/FDD.md` | Risco | URL de destino apontando para rede interna (SSRF) | TRANSCRICAO | [09:23] Sofia |
| FDD-RISK-08 | `docs/FDD.md` | Risco | Cliente não deduplica e processa duas vezes | TRANSCRICAO | [09:25] Sofia |
| ADR-001 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Decisão | Outbox no MySQL com publicação na mesma transação | TRANSCRICAO | [09:08] Larissa |
| ADR-001-SNAPSHOT | `docs/adrs/ADR-001-outbox-no-mysql.md` | Decisão | Payload renderizado como snapshot na inserção | TRANSCRICAO | [09:52] Larissa |
| ADR-001-FILTRO | `docs/adrs/ADR-001-outbox-no-mysql.md` | Decisão | Filtro de status aplicado na inserção da outbox | TRANSCRICAO | [09:34] Bruno |
| ADR-001-UUID | `docs/adrs/ADR-001-outbox-no-mysql.md` | Decisão | Chave primária UUID, como no resto do projeto | TRANSCRICAO | [09:51] Larissa |
| ADR-001-REF | `docs/adrs/ADR-001-outbox-no-mysql.md` | Restrição | changeStatus já roda em prisma.$transaction (update, history, estoque) | CODIGO | `src/modules/orders/order.service.ts` |
| ADR-002 | `docs/adrs/ADR-002-worker-separado-em-polling.md` | Decisão | Worker em polling de 2s | TRANSCRICAO | [09:10] Larissa |
| ADR-002-PROCESSO | `docs/adrs/ADR-002-worker-separado-em-polling.md` | Decisão | Worker em processo separado da API com src/worker.ts | TRANSCRICAO | [09:11] Diego |
| ADR-002-ORDERING | `docs/adrs/ADR-002-worker-separado-em-polling.md` | Restrição | Ordering por order_id apenas com worker único | TRANSCRICAO | [09:13] Larissa |
| ADR-002-REF | `docs/adrs/ADR-002-worker-separado-em-polling.md` | Restrição | server.ts é o único entry point atual e trata SIGINT/SIGTERM | CODIGO | `src/server.ts` |
| ADR-003 | `docs/adrs/ADR-003-retry-backoff-e-dlq.md` | Decisão | Retry 1m/5m/30m/2h/12h e DLQ em tabela separada | TRANSCRICAO | [09:17] Larissa |
| ADR-003-DLQ | `docs/adrs/ADR-003-retry-backoff-e-dlq.md` | Decisão | DLQ em tabela webhook_dead_letter, não na própria outbox | TRANSCRICAO | [09:18] Diego |
| ADR-003-REF | `docs/adrs/ADR-003-retry-backoff-e-dlq.md` | Restrição | requireRole('ADMIN') disponível para o replay | CODIGO | `src/middlewares/auth.middleware.ts` |
| ADR-004 | `docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md` | Decisão | HMAC-SHA256 com secret por endpoint e rotação com grace de 24h | TRANSCRICAO | [09:22] Sofia |
| ADR-004-ROTACAO | `docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md` | Decisão | Rotação pela API com antiga válida por 24h | TRANSCRICAO | [09:21] Sofia |
| ADR-004-TLS | `docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md` | Restrição | URL do webhook deve ser https | TRANSCRICAO | [09:23] Sofia |
| ADR-005 | `docs/adrs/ADR-005-at-least-once-com-x-event-id.md` | Decisão | At-least-once com X-Event-Id para dedup no cliente | TRANSCRICAO | [09:26] Larissa |
| ADR-005-REF | `docs/adrs/ADR-005-at-least-once-com-x-event-id.md` | Restrição | Biblioteca uuid já usada para gerar ids de requisição | CODIGO | `src/middlewares/request-logger.middleware.ts` |
| ADR-006 | `docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md` | Decisão | Reuso máximo dos padrões existentes do projeto | TRANSCRICAO | [09:30] Larissa |
| ADR-006-PREFIXO | `docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md` | Decisão | Prefixo WEBHOOK_ em todos os códigos de erro do módulo | TRANSCRICAO | [09:29] Larissa |
| ADR-006-REF | `docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md` | Restrição | Padrão de erros: subclasses de AppError com errorCode | CODIGO | `src/shared/errors/http-errors.ts` |
| ADR-006-MODULO | `docs/adrs/ADR-006-reuso-dos-padroes-do-projeto.md` | Restrição | Padrão de módulo com controller, service, repository, routes, schemas | CODIGO | `src/modules/orders/order.routes.ts` |
| ADR-007 | `docs/adrs/ADR-007-contrato-de-entrega-payload-e-headers.md` | Decisão | Payload enxuto, headers padronizados, timeout de 10s e limite de 64 KB | TRANSCRICAO | [09:44] Diego |
| ADR-007-LIMITE | `docs/adrs/ADR-007-contrato-de-entrega-payload-e-headers.md` | Restrição | Limite de 64 KB com erro, sem truncar | TRANSCRICAO | [09:24] Larissa |
| ADR-007-HEADERS | `docs/adrs/ADR-007-contrato-de-entrega-payload-e-headers.md` | Decisão | X-Webhook-Id adicionado a pedido de Sofia | TRANSCRICAO | [09:44] Sofia |
| ADR-007-TIMEOUT | `docs/adrs/ADR-007-contrato-de-entrega-payload-e-headers.md` | Decisão | Timeout de 10s por chamada ao cliente | TRANSCRICAO | [09:42] Diego |
| ADR-007-REF | `docs/adrs/ADR-007-contrato-de-entrega-payload-e-headers.md` | Restrição | Campos de payload derivados de Order (orderNumber, totalCents) | CODIGO | `prisma/schema.prisma` |
