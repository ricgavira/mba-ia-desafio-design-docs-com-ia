# PRD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| **Produto** | Order Management System (OMS) |
| **Feature** | Webhooks de notificação de mudança de status de pedido |
| **Responsável de produto** | Marcos (PM) |
| **Status** | Pronto para revisão |
| **Data** | 2026-10-23 |
| **Documentos relacionados** | [RFC](RFC.md) · [FDD](FDD.md) · [ADRs](adrs/) · [Tracker](TRACKER.md) |

Os identificadores `PRD-*` ligam cada item ao [Tracker](TRACKER.md). Este documento responde **por que e o quê**; a arquitetura está no RFC e a construção no FDD.

## 1. Resumo e contexto da feature

O OMS passará a **avisar automaticamente, por webhook HTTP de saída**, cada cliente B2B cadastrado quando o status de um pedido dele mudar. O cliente registra uma URL, escolhe quais status quer ouvir e passa a receber um evento assinado a cada transição relevante, sem precisar consultar a API.

A aplicação atual é uma API Node.js + TypeScript com módulos de autenticação, usuários, clientes, produtos e pedidos, ciclo de vida de pedido controlado (`PENDING → PAID → PROCESSING → SHIPPED → DELIVERED`, com `CANCELLED`) e auditoria em `order_status_history`. Ela não possui nenhum mecanismo de notificação externa.

## 2. Problema e motivação

- **Problema.** Atlas Comercial, MaxDistribuição e Nova Cargo pediram formalmente para serem notificados em tempo real. Hoje consultam `GET /orders` repetidamente, o que deixa a integração "lenta e cara" para eles ([09:00] Marcos).
- **Motivação de negócio.** A Atlas indicou que pode migrar para o concorrente se a entrega não ocorrer até o fim do trimestre ([09:00] Marcos); a Atlas quer a entrega para o fim de novembro ([09:45] Marcos).
- **Definição de "tempo real".** Para os clientes, qualquer coisa abaixo de 10 segundos ([09:02] Marcos).

## 3. Público-alvo e cenários de uso

**Públicos.**

| Público | Papel nesta feature |
| --- | --- |
| Clientes B2B (Atlas, MaxDistribuição, Nova Cargo) | Recebem os eventos nos sistemas deles e precisam validar autenticidade e deduplicar. |
| Usuários da plataforma que representam o cliente | Cadastram e gerenciam webhooks pela API, autenticados com JWT do OMS ([09:32] Marcos). |
| Administradores (role `ADMIN`) | Reprocessam eventos que falharam em definitivo ([09:36] Sofia). |
| Segurança (Sofia) e Produto (Marcos) | Revisam a segurança antes do deploy e documentam a integração no portal de desenvolvedor ([09:46] Sofia; [09:40] Marcos). |

**Cenários.**

| ID | Cenário |
| --- | --- |
| PRD-CEN-01 | A Atlas cadastra um webhook para ouvir só `SHIPPED` e `DELIVERED` ([09:33] Marcos) e passa a receber um aviso a cada pedido que entra nesses status. |
| PRD-CEN-02 | O endpoint de um cliente fica fora do ar numa manutenção de 2 horas ([09:16] Diego); a plataforma tenta de novo ao longo de cerca de 15 horas e entrega quando ele volta. |
| PRD-CEN-03 | O cliente recebe o mesmo evento duas vezes e o descarta pelo identificador do evento ([09:25] Diego). |
| PRD-CEN-04 | A secret de um cliente vaza em log da aplicação dele; ele gera uma nova pela API e mantém a antiga válida por 24 h enquanto migra ([09:21]–[09:22] Sofia, Diego). |
| PRD-CEN-05 | O cliente consulta os últimos envios (sucesso ou falha, payload, resposta e tempo) para diagnosticar um problema ([09:34] Marcos). |
| PRD-CEN-06 | Um evento esgota as tentativas, vai para a fila de falhas e um `ADMIN` o reenvia manualmente ([09:18] Diego). |

## 4. Objetivos e métricas de sucesso

| ID | Objetivo | Métrica | Meta | Origem |
| --- | --- | --- | --- | --- |
| PRD-OBJ-01 | Atender o pedido dos clientes B2B | Clientes com webhook em produção | **3 de 3** (partindo de 0), com a Atlas em produção até o fim de novembro; a reunião não definiu prazo para MaxDistribuição e Nova Cargo | [09:00] Marcos; [09:45] Marcos |
| PRD-OBJ-02 | Notificar em "tempo real" | Tempo entre o commit da mudança de status e a entrega ao endpoint saudável | **< 10 s** (polling de 2 s no pior caso + chamada HTTP) | [09:02] Marcos; [09:10] Larissa |
| PRD-OBJ-03 | Nunca perder um evento por inconsistência | Mudanças de status sem evento correspondente (para webhooks elegíveis) | **0** | [09:40] Bruno |
| PRD-OBJ-04 | Entregar no prazo comprometido | Duração do projeto | **3 sprints**, incluindo a revisão de segurança | [09:46]–[09:47] Larissa |
| PRD-OBJ-05 | Reduzir o consumo de `GET /orders` pelos clientes | Chamadas de polling dos 3 clientes | Redução acompanhada após o go-live (linha de base ainda não medida; **meta a definir**) | [09:00] Marcos |

## 5. Escopo

### Incluso

| ID | Item |
| --- | --- |
| PRD-ESC-01 | Notificação **de saída** a cada mudança de status de pedido ([09:02] Marcos; [09:03] Sofia). |
| PRD-ESC-02 | API para cadastrar, listar, editar e remover webhooks, com escolha dos status de interesse. |
| PRD-ESC-03 | Consulta ao histórico de entregas de um webhook. |
| PRD-ESC-04 | Rotação de secret pela API. |
| PRD-ESC-05 | Reenvio manual de eventos com falha permanente, restrito a `ADMIN`. |

### Fora de escopo (descartado ou adiado na reunião)

| ID | Item | Situação | Origem |
| --- | --- | --- | --- |
| PRD-FORA-01 | **Aviso por e-mail** ao cliente quando o webhook falha | Adiado para a próxima fase, após medir o impacto | [09:37] Larissa; [09:38] Marcos |
| PRD-FORA-02 | **Rate limiting de saída** por cliente | Não entra; "observar e decidir depois" | [09:39] Diego, Larissa |
| PRD-FORA-03 | **Dashboard visual** para o cliente | Descartado; só endpoints, painel é projeto separado do time de frontend | [09:40] Larissa |
| PRD-FORA-04 | **Arquivamento** de eventos entregues após ~30 dias | Fora desta feature | [09:08] Diego |
| PRD-FORA-05 | **Garantia de ordenação global** e múltiplos workers | Limitação conhecida; os clientes nunca pediram | [09:13] Larissa; [09:14] Marcos |
| PRD-FORA-06 | **Exactly-once** | Descartado por complexidade; vale at-least-once | [09:25] Diego |
| PRD-FORA-07 | **Webhooks de entrada** (cliente envia para a plataforma) | Fora de escopo | [09:02] Marcos |

## 6. Requisitos funcionais

| ID | Requisito | Origem |
| --- | --- | --- |
| PRD-FR-01 | O cliente pode **cadastrar um webhook** informando a URL e a lista de status que deseja receber; a **secret é gerada pela plataforma** e devolvida na criação. | [09:31] Marcos |
| PRD-FR-02 | O `customer_id` é informado no corpo ou no caminho da requisição, **não derivado do JWT** (o JWT é de um usuário da plataforma). | [09:32] Larissa |
| PRD-FR-03 | O cliente pode **listar** os webhooks de um customer. | [09:33] Bruno |
| PRD-FR-04 | O cliente pode **editar** um webhook. | [09:33] Bruno |
| PRD-FR-05 | O cliente pode **remover** um webhook. | [09:33] Bruno |
| PRD-FR-06 | Cada webhook escolhe os **status que quer ouvir**; eventos de outros status nem são gerados para ele. | [09:33] Marcos; [09:34] Bruno |
| PRD-FR-07 | A cada mudança de status elegível, a plataforma **envia um evento** ao endpoint do cliente contendo identificador do evento, tipo, data/hora, dados do pedido (id, número, status anterior e novo, cliente, total) e **sem os itens**. | [09:43] Diego |
| PRD-FR-08 | O cliente pode **consultar o histórico de entregas** de um webhook (últimos 100): sucesso ou falha, payload, resposta e tempo de resposta. | [09:34] Marcos |
| PRD-FR-09 | O cliente pode **rotacionar a secret** pela API; a anterior permanece válida por **24 horas**. | [09:21] Sofia |
| PRD-FR-10 | Cada envio é **assinado** (HMAC-SHA256) com a secret do endpoint e traz identificador do evento, do webhook e momento do envio. | [09:20] Sofia; [09:44] Diego, Sofia |
| PRD-FR-11 | Cada evento tem **identificador único** estável entre reenvios, para que o cliente deduplique. | [09:25] Diego |
| PRD-FR-12 | Entregas com falha são **retentadas automaticamente** (5 retentativas, com intervalos crescentes de 1 min a 12 h) e, esgotadas, vão para uma **fila de falhas**. | [09:17] Larissa |
| PRD-FR-13 | Um **ADMIN** pode **reenviar** manualmente um evento da fila de falhas; o autor do reenvio é registrado. | [09:18] Diego; [09:36] Sofia |
| PRD-FR-14 | Somente URLs **`https`** são aceitas no cadastro. | [09:23] Sofia |
| PRD-FR-15 | O cadastro e a manutenção dos webhooks são permitidos a **qualquer role autenticada**; o reenvio exige `ADMIN`. | [09:36] Larissa; [09:37] Sofia |

## 7. Requisitos não funcionais

| ID | Requisito | Origem |
| --- | --- | --- |
| PRD-NFR-01 | **Latência:** entrega em menos de 10 s após a mudança de status (detecção por polling a cada 2 s). | [09:02] Marcos; [09:10] Larissa |
| PRD-NFR-02 | **Consistência:** a mudança de status e o registro do evento são atômicos; se um falhar, ambos são desfeitos. | [09:40] Bruno |
| PRD-NFR-03 | **Isolamento:** a lentidão ou indisponibilidade de um cliente não pode travar mudanças de status de outros pedidos nem fazer rollback. | [09:04] Bruno |
| PRD-NFR-04 | **Semântica de entrega:** at-least-once; o cliente deve estar preparado para duplicatas. | [09:24] Diego |
| PRD-NFR-05 | **Segurança:** autenticidade e integridade por HMAC-SHA256, secret única por endpoint, TLS obrigatório. | [09:19]–[09:23] Sofia |
| PRD-NFR-06 | **Tamanho:** payload máximo de 64 KB; acima disso o envio falha (não trunca). | [09:24] Larissa |
| PRD-NFR-07 | **Timeout:** 10 s por chamada ao cliente; estouro conta como falha. | [09:42] Diego |
| PRD-NFR-08 | **Auditoria:** reenvios manuais registram quem os executou. | [09:36] Sofia |
| PRD-NFR-09 | **Operabilidade:** sem infraestrutura nova; reuso de MySQL, Prisma, Pino e dos padrões de erro e de módulos. | [09:07] Diego; [09:30] Larissa |
| PRD-NFR-10 | **Limitação conhecida:** ordem garantida apenas por pedido e enquanto houver um único worker. | [09:13] Larissa |

## 8. Decisões e trade-offs principais

Detalhes e alternativas estão nos ADRs.

| ID | Decisão | Trade-off aceito | ADR |
| --- | --- | --- | --- |
| PRD-DEC-01 | Registrar o evento por **outbox no MySQL**, na mesma transação do pedido. | Um `INSERT` a mais na transação, em troca de consistência e zero infraestrutura nova. | [ADR-001](adrs/ADR-001-outbox-no-mysql.md) |
| PRD-DEC-02 | **Worker separado** com polling de 2 s. | Latência mínima de até 2 s e um processo extra para operar. | [ADR-002](adrs/ADR-002-worker-separado-em-polling.md) |
| PRD-DEC-03 | **Retry com backoff** (5 tentativas) e **fila de falhas** com reenvio manual. | Cliente fora do ar por mais de ~15 h exige intervenção. | [ADR-003](adrs/ADR-003-retry-backoff-e-dlq.md) |
| PRD-DEC-04 | **HMAC-SHA256** com secret por endpoint e rotação de 24 h. | Complexidade de manter duas secrets; revisão de segurança antes do deploy. | [ADR-004](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md) |
| PRD-DEC-05 | **At-least-once** com identificador de evento. | Responsabilidade de deduplicar passa ao cliente. | [ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md) |
| PRD-DEC-06 | **Reuso dos padrões** do projeto. | Herda as limitações dos padrões atuais. | [ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md) |
| PRD-DEC-07 | **Payload enxuto** sem itens. | Cliente que precisa de detalhes consulta `GET /orders/:id`. | [ADR-007](adrs/ADR-007-contrato-de-entrega-payload-e-headers.md) |

## 9. Dependências

| ID | Dependência | Origem |
| --- | --- | --- |
| PRD-DEP-01 | Banco MySQL e Prisma existentes, além do módulo de pedidos (`changeStatus`) e dos usuários/clientes já cadastrados. | [09:07] Diego; [09:40] Bruno |
| PRD-DEP-02 | **Revisão de segurança** de Sofia: ao menos 2 dias úteis antes do deploy. | [09:46] Sofia |
| PRD-DEP-03 | **Documentação no portal de desenvolvedor** (Marcos), com destaque para a deduplicação e a verificação da assinatura. | [09:26] Marcos; [09:40] Marcos |
| PRD-DEP-04 | Capacidade de operar um **segundo processo** (worker) em produção. | [09:11] Diego |
| PRD-DEP-05 | Os clientes precisam implementar o recebimento: endpoint HTTPS, validação do HMAC e deduplicação. | [09:20] Sofia; [09:25] Diego |
| PRD-DEP-06 | Confirmação do prazo com a Atlas por Marcos. | [09:47] Marcos |

**Cronograma estimado (3 sprints)** ([09:46] Larissa): modelagem de outbox e DLQ (1 sprint); worker e retry (1 sprint); CRUD de configuração e entregas (0,5); integração em `changeStatus` e testes ponta a ponta (0,5); HMAC, schemas e validações; revisão de segurança ao final.

## 10. Riscos e mitigação

| ID | Risco | Prob. | Impacto | Mitigação |
| --- | --- | --- | --- | --- |
| PRD-RISK-01 | **Perder a data** da Atlas (fim de novembro) e o cliente migrar para o concorrente. | Média | Alto | Escopo restrito a endpoints (sem painel, e-mail ou rate limit); revisão de segurança agendada com antecedência; Marcos confirma prazo com a Atlas. |
| PRD-RISK-02 | **Cliente não deduplica** e processa o mesmo evento duas vezes. | Média | Médio | Documentação destacada no portal; identificador de evento estável entre reenvios. |
| PRD-RISK-03 | **Vazamento de secret** de um cliente (já ocorreu com um cliente). | Média | Alto | Secret por endpoint, rotação com 24 h de convivência, revisão de segurança. |
| PRD-RISK-04 | **Entrega lenta** para clientes indisponíveis, com retries em janelas longas. | Média | Médio | Timeout de 10 s; fila de falhas com reenvio por `ADMIN`; avaliar e-mail e rate limit na próxima fase. |
| PRD-RISK-05 | **Eventos fora de ordem** para um mesmo pedido. | Baixa | Médio | Limitação documentada; cliente usa data/hora e status do evento. |

## 11. Critérios de aceitação

| ID | Critério |
| --- | --- |
| PRD-ACE-01 | Um cliente cadastra um webhook `https` com seus status de interesse e recebe a secret na criação; URL `http` é recusada. |
| PRD-ACE-02 | Ao mudar o status de um pedido para um status ouvido, o endpoint recebe o evento em menos de 10 s, com assinatura HMAC-SHA256 válida e headers de identificação. |
| PRD-ACE-03 | Ao mudar o status para um status não ouvido, nenhum evento é gerado. |
| PRD-ACE-04 | Se a mudança de status falhar, nenhum evento é registrado; se o registro do evento falhar, o status não muda. |
| PRD-ACE-05 | Com o endpoint fora do ar, a plataforma retenta com os intervalos definidos e, esgotadas as tentativas, move o evento para a fila de falhas. |
| PRD-ACE-06 | Um `ADMIN` reenvia um evento da fila de falhas; um usuário `OPERATOR` recebe recusa; o autor fica registrado. |
| PRD-ACE-07 | O cliente consulta os últimos envios com resultado, payload, resposta e tempo. |
| PRD-ACE-08 | Após a rotação, a secret antiga e a nova funcionam por 24 h; depois, só a nova. |
| PRD-ACE-09 | O módulo reutiliza os padrões do projeto (códigos `WEBHOOK_*`, erros, logger) e a revisão de segurança foi concluída antes do deploy. |

## 12. Estratégia de testes e validação

O projeto usa Vitest + Supertest, com `tests/setup.ts` (limpeza do banco) e `tests/helpers/factories.ts` (usuários, clientes e produtos de teste).

| ID | Camada | O que validar |
| --- | --- | --- |
| PRD-TEST-01 | Unitários | Tabela de backoff e transição para a fila de falhas; cálculo e verificação do HMAC; montagem do payload; filtro por status; validação de URL `https`; limite de 64 KB. |
| PRD-TEST-02 | Integração (API) | CRUD de webhooks, permissões (`ADMIN` no reenvio, qualquer role no CRUD), rotação de secret, histórico de entregas e envelope de erros `WEBHOOK_*`. |
| PRD-TEST-03 | Integração (transação) | `changeStatus` grava o evento na mesma transação e reverte tudo se o registro do evento falhar; nenhum evento para status não ouvido. |
| PRD-TEST-04 | Worker (ponta a ponta) | Contra um servidor HTTP local de teste: sucesso, timeout de 10 s, resposta 5xx, retentativas agendadas, fila de falhas, reenvio e duplicidade com mesmo `X-Event-Id`. |
| PRD-TEST-05 | Validação de segurança e prazo | Revisão de código de Sofia (HMAC e geração de secret) antes do deploy; Marcos confirma o prazo e atualiza os clientes. |
