# RFC — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| **Autor** | Ricardo Gavira (redação assistida por IA, a partir da transcrição da reunião técnica) |
| **Patrocínio técnico** | Larissa (Tech Lead), que se comprometeu a abrir o doc de design e a marcar a sessão de revisão com Bruno e Diego ([09:50] Larissa) |
| **Status** | Em revisão |
| **Data** | 2026-10-23 |
| **Revisores** | Larissa (Tech Lead), Marcos (PM), Bruno (Eng. Pleno, Pedidos), Diego (Eng. Sênior, Plataforma), Sofia (Segurança) |
| **Documentos relacionados** | [PRD](PRD.md) · [FDD](FDD.md) · [Tracker](TRACKER.md) · [ADRs](adrs/) |

Os identificadores `RFC-*` ligam cada item ao [Tracker](TRACKER.md).

## 1. Resumo executivo (TL;DR)

Propomos notificar clientes B2B por **webhooks de saída** a cada mudança de status de pedido. O evento é gravado na tabela `webhook_outbox` **dentro da mesma transação SQL** de `OrderService.changeStatus`; um **worker em processo separado**, em **polling de 2 s**, entrega o evento por HTTPS assinado com **HMAC-SHA256 (secret por endpoint)**. Falhas seguem **backoff 1m/5m/30m/2h/12h** (5 intervalos; contagem exata em [RFC-Q-05](#5-questões-em-aberto)) e vão para uma **DLQ em tabela própria**, com replay manual por ADMIN. A garantia é **at-least-once**, com deduplicação do lado do cliente via `X-Event-Id`. Tudo reaproveita os padrões do código atual, sem infraestrutura nova. Prazo estimado: **3 sprints**, incluindo a revisão de segurança ([09:46] Larissa).

## 2. Contexto e problema

Atlas Comercial, MaxDistribuição e Nova Cargo fazem *polling* em `GET /orders` para detectar mudanças de status, o que torna a integração "lenta e cara" para eles. A Atlas sinalizou que pode migrar para o concorrente se a entrega não ocorrer até o fim do trimestre ([09:00] Marcos). Para os clientes, atraso abaixo de 10 s é "tempo real" ([09:02] Marcos), e o fluxo é apenas de saída: eles recebem, não enviam ([09:02] Marcos; [09:03] Sofia).

A aplicação hoje não possui eventos, filas ou webhooks. A mudança de status ocorre em uma transação já pesada (atualiza `orders`, insere em `order_status_history`, ajusta estoque), e um HTTP dentro dela travaria outros pedidos ou exigiria rollback quando o cliente estiver fora do ar ([09:04] Bruno).

## 3. Proposta técnica

Visão geral (detalhes de implementação no [FDD](FDD.md)):

```
changeStatus (API) ──tx──► orders + order_status_history + webhook_outbox
                                                │
              worker (src/worker.ts, polling 2s) ◄┘
                    │  POST assinado (HMAC-SHA256, timeout 10s)
                    ▼
            endpoint HTTPS do cliente ── falha ──► retry (5x) ──► webhook_dead_letter ──► replay ADMIN
```

Pilares, cada um com seu ADR:

1. **Outbox no MySQL, na mesma transação** (RFC-PROP-01). Atomicidade entre mudança de status e evento; payload gravado como snapshot; filtro de status do endpoint aplicado na inserção. → [ADR-001](adrs/ADR-001-outbox-no-mysql.md)
2. **Worker separado em polling de 2 s** (RFC-PROP-02). Entry point `src/worker.ts`, mesmo banco, `PrismaClient` próprio, single-worker. → [ADR-002](adrs/ADR-002-worker-separado-em-polling.md)
3. **Retry exponencial e DLQ** (RFC-PROP-03). Timeout de 10 s por chamada; esgotadas as tentativas o evento migra para `webhook_dead_letter`; replay manual via endpoint ADMIN. → [ADR-003](adrs/ADR-003-retry-backoff-e-dlq.md)
4. **Autenticidade por HMAC-SHA256** (RFC-PROP-04). Secret única por endpoint, rotação com 24 h de convivência, HTTPS obrigatório. → [ADR-004](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md)
5. **At-least-once com `X-Event-Id`** (RFC-PROP-05). O cliente deduplica. → [ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md)
6. **Reuso dos padrões do projeto** (RFC-PROP-06). Módulo `src/modules/webhooks`, `AppError`, prefixo `WEBHOOK_`, Pino, `requireRole`. → [ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md)
7. **Contrato de entrega enxuto** (RFC-PROP-07). Payload sem `items`, headers `X-Event-Id`/`X-Signature`/`X-Timestamp`/`X-Webhook-Id`, limite de 64 KB. → [ADR-007](adrs/ADR-007-contrato-de-entrega-payload-e-headers.md)

A superfície pública consiste em endpoints de CRUD de configuração, consulta de histórico de entregas, rotação de secret e replay de DLQ (ADMIN). Seus contratos estão no FDD.

## 4. Alternativas consideradas

| ID | Alternativa | Por que foi descartada |
| --- | --- | --- |
| RFC-ALT-01 | **Disparo HTTP síncrono** no `OrderService` | Cliente lento travaria a transação de mudança de status; cliente offline obrigaria a escolher entre rollback e perda de evento ([09:04] Bruno; [09:06] Diego). |
| RFC-ALT-02 | **Fila externa (Redis Streams ou similar)** | Nova infraestrutura para um time pequeno ("overengineering"), e o evento nasceria fora da transação SQL ([09:07] Larissa, Diego). |
| RFC-ALT-03 | **Trigger de banco para acionar o worker** | MySQL não tem `LISTEN/NOTIFY`; a trigger só executa SQL e não avisa processo externo ([09:09] Diego). |
| RFC-ALT-04 | **Retry indefinido** (ou apenas 3 tentativas) | Indefinido deixa eventos pendurados para sempre; 3 tentativas não cobrem uma manutenção de 2 h ([09:15]–[09:16] Diego). |
| RFC-ALT-05 | **Exactly-once** | Exige coordenação dos dois lados e complexidade muito maior ([09:25] Diego). |
| RFC-ALT-06 | **Secret global da plataforma** | Um vazamento comprometeria todos os clientes ([09:21] Sofia). |

## 5. Questões em aberto

| ID | Questão | Origem / situação |
| --- | --- | --- |
| RFC-Q-01 | **Rate limiting de saída por cliente.** Com 50 pedidos mudando em um minuto, enviamos 50 chamadas? | Fora do escopo; "observar e decidir depois" ([09:38] Diego; [09:39] Larissa). Decidir após medir. |
| RFC-Q-02 | **Escala do worker e ordering.** Com mais de um worker, perde-se a ordem; saídas possíveis: particionar por `order_id` ou lock pessimista. | Adiado como "problema do futuro" ([09:13] Diego). |
| RFC-Q-03 | **Aviso proativo ao cliente (e-mail)** quando o webhook falha repetidamente. | Adiado para uma próxima fase, após medir impacto ([09:37] Larissa). |
| RFC-Q-04 | **Arquivamento da outbox** (linhas entregues após ~30 dias). | Mencionado como fora do escopo ([09:08] Diego); sem decisão de quando/como. |
| RFC-Q-05 | **"5 tentativas": 5 retentativas ou 5 envios no total?** A lista de 5 intervalos e o total de ~15 h só fecham com 1 envio + 5 retentativas. | *Interpretação do autor sobre [09:15]–[09:17] Diego*; confirmar na revisão. |
| RFC-Q-06 | **Ordem entre eventos de um mesmo pedido durante retry.** Um evento em espera de retry pode ser ultrapassado por um mais novo do mesmo pedido. | *Análise do autor* a partir de [09:12]–[09:13] Diego; decidir se há bloqueio por pedido ou se o cliente usa `timestamp` e `to_status`. |
| RFC-Q-07 | **Proteção em repouso da secret** (precisa ser recuperável para assinar) e formato de `X-Signature` durante a rotação. | Levado à revisão de segurança ([09:46] Sofia: "HMAC e geração de secret eu quero olhar com calma"). |
| RFC-Q-08 | **Endurecimento de permissões** do CRUD de webhook (hoje, qualquer role autenticada). | "Por enquanto sim. Mais pra frente a gente pode endurecer" ([09:37] Sofia). |

## 6. Impacto e riscos

**Impacto no sistema existente.** Uma única alteração em código existente: `OrderService.changeStatus` chama `publishWebhookEvent(tx, ...)` dentro da transação ([09:41] Bruno). Há novas tabelas (`webhook_*`), um módulo novo, uma entry point nova (`src/worker.ts`) e o script `npm run worker`; o `error.middleware.ts` não muda ([09:29] Bruno). O deploy passa a incluir um segundo processo.

| ID | Risco | Mitigação proposta |
| --- | --- | --- |
| RFC-RISK-01 | Mais um `INSERT` na transação de `changeStatus` aumenta sua duração. | Inserção condicionada ao filtro de status (só grava se algum webhook quer o evento) ([09:34] Bruno); índice em `status`/`created_at` ([09:08] Diego). |
| RFC-RISK-02 | Cliente não deduplica e processa o mesmo evento duas vezes. | Documentação destacada no portal de desenvolvedor ([09:26] Marcos); `X-Event-Id` estável entre reenvios. |
| RFC-RISK-03 | Vazamento de secret de cliente. | Secret por endpoint, rotação com 24 h de convivência e revisão de segurança antes do deploy ([09:21]–[09:22] Sofia, Diego; [09:46] Sofia). |
| RFC-RISK-04 | Prazo de 3 sprints para a data prometida à Atlas (fim de novembro). | Escopo limitado a endpoints (sem painel, sem e-mail, sem rate limit) ([09:37]–[09:40] Larissa); reservar 2 dias úteis para a revisão de segurança ([09:46] Sofia). |

## 7. Decisões relacionadas

- [ADR-001 — Outbox no MySQL](adrs/ADR-001-outbox-no-mysql.md)
- [ADR-002 — Worker separado em polling](adrs/ADR-002-worker-separado-em-polling.md)
- [ADR-003 — Retry com backoff e DLQ](adrs/ADR-003-retry-backoff-e-dlq.md)
- [ADR-004 — HMAC-SHA256 com secret por endpoint](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md)
- [ADR-005 — At-least-once com X-Event-Id](adrs/ADR-005-at-least-once-com-x-event-id.md)
- [ADR-006 — Reuso dos padrões do projeto](adrs/ADR-006-reuso-dos-padroes-do-projeto.md)
- [ADR-007 — Contrato de entrega](adrs/ADR-007-contrato-de-entrega-payload-e-headers.md)
