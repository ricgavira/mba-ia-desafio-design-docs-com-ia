# FDD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
| --- | --- |
| **Status** | Rascunho para revisão técnica (Larissa, Bruno, Diego) e de segurança (Sofia) |
| **Data** | 2026-10-23 |
| **Documentos relacionados** | [PRD](PRD.md) · [RFC](RFC.md) · [ADR-001](adrs/ADR-001-outbox-no-mysql.md) a [ADR-007](adrs/ADR-007-contrato-de-entrega-payload-e-headers.md) · [Tracker](TRACKER.md) |

**Convenção de origem.** Itens com ID `FDD-*` estão no [Tracker](TRACKER.md). Onde a transcrição define o comportamento, a origem aparece como `[hh:mm] Nome`. Onde o FDD precisou **completar um detalhe que a reunião não fechou**, o texto traz a marca **(decisão de design)**; essas escolhas estão listadas na seção 13 para validação do time. Nomes de campos de API REST seguem o padrão `camelCase` do código atual (`orderNumber`, `totalCents`); o **payload entregue ao cliente** usa `snake_case`, como definido na reunião ([09:43] Diego).

---

## 1. Contexto e motivação técnica

Hoje `OrderService.changeStatus` (`src/modules/orders/order.service.ts`) executa dentro de `prisma.$transaction`: valida a transição com `canTransition`, ajusta estoque (`debitStock`/`replenishStock`), faz `tx.order.update` e `tx.orderStatusHistory.create`. Não há nenhum mecanismo de notificação externa. Os clientes Atlas Comercial, MaxDistribuição e Nova Cargo consultam `GET /orders` repetidamente ([09:00] Marcos) e consideram "tempo real" qualquer entrega em menos de 10 s ([09:02] Marcos).

O FDD especifica **como** construir o módulo `webhooks` conforme as decisões dos ADRs, sem alterar a semântica da máquina de estados (`src/modules/orders/order.status.ts`).

## 2. Objetivos técnicos

| ID | Objetivo | Origem |
| --- | --- | --- |
| FDD-OBJ-01 | Gravar o evento na `webhook_outbox` na **mesma transação** de `changeStatus`; sem evento sem status e sem status sem evento. | [09:40] Bruno |
| FDD-OBJ-02 | Entregar eventos ao endpoint do cliente em **< 10 s** após o commit (polling de 2 s + chamada HTTP). | [09:02] Marcos; [09:10] Larissa |
| FDD-OBJ-03 | Garantir entrega **at-least-once** com `X-Event-Id` estável entre reenvios. | [09:24]–[09:26] Diego, Larissa |
| FDD-OBJ-04 | Autenticar cada envio com **HMAC-SHA256** e secret por endpoint, com rotação e 24 h de convivência. | [09:22] Sofia |
| FDD-OBJ-05 | Não introduzir infraestrutura nem dependência nova (MySQL, Prisma, Pino, `zod`, `uuid` já existentes). | [09:07] Diego; [09:29] Bruno |

## 3. Escopo e exclusões

**Dentro do escopo:** tabelas `webhook_*`; módulo `src/modules/webhooks`; worker `src/worker.ts`; endpoints de CRUD, deliveries, rotação de secret e replay de DLQ; extensão de `changeStatus`.

| ID | Fora do escopo | Origem |
| --- | --- | --- |
| FDD-ESC-01 | Webhooks de **entrada** (cliente → plataforma). | [09:02] Marcos |
| FDD-ESC-02 | Aviso por e-mail ao cliente quando o webhook falha. | [09:37] Larissa |
| FDD-ESC-03 | Rate limiting de saída por cliente ("observar e decidir depois"). | [09:39] Larissa |
| FDD-ESC-04 | Dashboard visual / painel de webhooks. | [09:40] Larissa |
| FDD-ESC-05 | Arquivamento de linhas entregues da outbox (~30 dias). | [09:08] Diego |
| FDD-ESC-06 | Múltiplos workers e ordering global. | [09:12]–[09:13] Diego, Larissa |
| FDD-ESC-07 | Evento de **criação** de pedido: a reunião trata mudanças de status em `changeStatus` ([09:40] Bruno); `OrderService.create` não é alterado. | [09:40] Bruno |

---

## 4. Modelo de dados

Novos modelos em `prisma/schema.prisma`, seguindo o padrão existente (`id String @id @default(uuid()) @db.Char(36)`, `@@map` em snake_case, `createdAt/updatedAt`) ([09:51] Larissa). Os nomes de campos não citados explicitamente na reunião são **(decisão de design)**.

| ID | Tabela | Conteúdo essencial | Origem |
| --- | --- | --- | --- |
| FDD-DATA-01 | `webhooks` | `id`, `customerId` (FK `customers`), `url`, `secret`, `previousSecret?`, `previousSecretExpiresAt?`, `events` (JSON com lista de `OrderStatus`), `active`, timestamps. | [09:21] Bruno/Sofia; [09:33] Marcos |
| FDD-DATA-02 | `webhook_outbox` | `id`, `eventId` (UUID único), `webhookId` (FK), `orderId`, `eventType`, `payload` (JSON, snapshot), `status` (`PENDING`, `PROCESSING`, `FAILED`, `DELIVERED`), `attemptCount`, `nextAttemptAt`, `lastError?`, `deliveredAt?`, timestamps. Índices: `(status, nextAttemptAt)` e `createdAt`. | [09:08] Diego; [09:25] Diego; [09:52] Larissa |
| FDD-DATA-03 | `webhook_delivery_attempts` | uma linha por tentativa: `outboxId`, `webhookId`, `eventId`, `attemptNumber`, `outcome` (`SUCCESS`/`FAILURE`), `httpStatus?`, `responseBody?` (truncado), `durationMs`, `errorCode?`, `createdAt`. Alimenta o histórico de entregas. | [09:34] Marcos |
| FDD-DATA-04 | `webhook_dead_letter` | `id`, `outboxId`, `webhookId`, `eventId`, `payload`, `failureReason` (código + mensagem), `attemptCount`, `failedAt`, `replayedAt?`, `replayedById?`. | [09:18] Diego; [09:36] Sofia |

Observações:

- **Uma linha de outbox por (evento, webhook)**: cada endpoint tem seu `eventId`, necessário para `X-Webhook-Id` e para retry independente por endpoint ([09:44] Sofia). *(decisão de design)*
- Os estados da outbox espelham "pendente, processando, falhou, entregue" ([09:08] Diego). `FAILED` significa **falha permanente já copiada para a DLQ**; retentativas ficam em `PENDING` com `nextAttemptAt` no futuro. *(decisão de design)*
- `ON DELETE CASCADE` de `webhooks` para as demais tabelas: remover o webhook remove fila, histórico e DLQ dele. *(decisão de design; ver risco FDD-RISK-05)*
- O `secret` fica recuperável (precisa assinar); a proteção em repouso aguarda a revisão de Sofia ([09:46] Sofia).

---

## 5. Fluxos detalhados

### FDD-FLUXO-01 — Criação do evento na outbox

Ponto de integração: `OrderService.changeStatus`, imediatamente após `tx.orderStatusHistory.create(...)` e antes do `findUnique` final.

1. A transição já foi validada (`canTransition`) e o estoque ajustado; `order` (id, orderNumber, customerId, totalCents) está disponível em `tx`.
2. Chamar `publishWebhookEvent(tx, order, from, to)` — função que recebe o `tx` da transação corrente, sem injetar repository ([09:41] Bruno, Diego).
3. A função busca em `tx.webhook` os webhooks `active` do `order.customerId` cujo `events` contenha `to` ([09:34] Bruno: filtro **na inserção**).
4. Para cada webhook elegível, gera `eventId = uuidv4()` e insere uma linha `PENDING` com `nextAttemptAt = now`, `payload` já renderizado (snapshot, [09:52] Larissa).
5. Qualquer falha no insert propaga a exceção e a transação inteira faz **rollback** ([09:40] Bruno). Se nenhum webhook é elegível, nada é inserido.

```ts
// src/modules/webhooks/webhook.publisher.ts (esboço)
export async function publishWebhookEvent(
  tx: Prisma.TransactionClient,
  order: { id: string; orderNumber: string; customerId: string; totalCents: number },
  fromStatus: OrderStatus,
  toStatus: OrderStatus,
): Promise<void> {
  const hooks = await tx.webhook.findMany({ where: { customerId: order.customerId, active: true } });
  const targets = hooks.filter((h) => (h.events as string[]).includes(toStatus));
  for (const hook of targets) {
    const eventId = uuidv4();
    await tx.webhookOutbox.create({
      data: {
        eventId, webhookId: hook.id, orderId: order.id, eventType: 'order.status_changed',
        payload: buildPayload(eventId, order, fromStatus, toStatus), // snapshot
        status: 'PENDING', nextAttemptAt: new Date(),
      },
    });
  }
}
```

Payload (campos de [09:43] Diego; sem `items`):

```json
{
  "event_id": "4b1f6a9e-2c1d-4e07-9d57-6a3c0f2b8e11",
  "event_type": "order.status_changed",
  "timestamp": "2026-11-05T14:32:10.123Z",
  "order_id": "9f0c2f0e-7a41-4c68-8a0e-1b6f7d9c3a52",
  "order_number": "ORD-000123",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "customer_id": "c1d2e3f4-0000-4000-8000-aaaaaaaaaaaa",
  "total_cents": 21500
}
```

### FDD-FLUXO-02 — Processamento pelo worker

Entry point `src/worker.ts` (espelha `src/server.ts`): instancia seu próprio `PrismaClient`, inicia o loop e trata `SIGINT`/`SIGTERM` com *graceful shutdown* (termina o ciclo em andamento, `prisma.$disconnect()`) ([09:11] Larissa; [09:30] Bruno).

Ciclo (executado a cada **2 s**, [09:09] Diego; o próximo ciclo só é agendado após o término do atual, para não haver sobreposição):

1. Selecionar até `WEBHOOK_WORKER_BATCH_SIZE` linhas `status = PENDING AND nextAttemptAt <= now()` ordenadas por `createdAt ASC` ("lote pequeno", [09:08] Diego; tamanho padrão 10 **(decisão de design)**).
2. Marcar o lote como `PROCESSING`.
3. Para cada evento: carregar o webhook; montar o corpo (JSON do snapshot); validar tamanho ≤ 64 KB; assinar; `POST` com timeout de 10 s; registrar a tentativa em `webhook_delivery_attempts`.
4. Resultado 2xx → `DELIVERED` (`deliveredAt = now`). Falha → FLUXO-03.

**Concorrência dentro do lote (decisão de design):** eventos de **webhooks diferentes** são processados em paralelo; eventos do **mesmo webhook** são processados em sequência, na ordem de `createdAt`. Sem isso, um único cliente lento (10 s de timeout) atrasaria todos os demais além do requisito de 10 s. Continua sendo um único processo worker ([09:12] Diego).

### FDD-FLUXO-03 — Retry com backoff

Após uma falha elegível a retry (timeout, erro de rede, resposta fora de 2xx):

| Falha nº | Próxima tentativa em | Origem |
| --- | --- | --- |
| 1ª | 1 minuto | [09:17] Diego |
| 2ª | 5 minutos | [09:17] Diego |
| 3ª | 30 minutos | [09:17] Diego |
| 4ª | 2 horas | [09:17] Diego |
| 5ª | 12 horas | [09:17] Diego |
| 6ª | → DLQ (FLUXO-04) | [09:15] Diego; [09:17] Larissa |

Cálculo: `nextAttemptAt = now + schedule[attemptCount - 1]`; o evento volta a `PENDING`. Total de ~14 h 36 min entre a primeira falha e a última tentativa. **Interpretação:** 1 envio inicial + 5 retentativas (ver RFC-Q-05). Falhas **permanentes** (`WEBHOOK_PAYLOAD_TOO_LARGE`, `WEBHOOK_SECRET_REQUIRED`, `WEBHOOK_ENDPOINT_INACTIVE`) vão direto à DLQ, pois retentar não resolveria *(decisão de design)*.

### FDD-FLUXO-04 — Dead Letter Queue

Em uma única transação: inserir em `webhook_dead_letter` (payload, motivo, timestamp, `attemptCount`) e marcar a linha da outbox como `FAILED` ([09:18] Diego). Registrar log `webhook_moved_to_dlq`. A DLQ é só leitura para o cliente (sem endpoint de listagem na reunião); o acesso é via replay admin e SQL de suporte.

### FDD-FLUXO-05 — Replay manual de DLQ

`POST /api/v1/admin/webhooks/dead-letter/:id/replay` ([09:18] Diego), protegido por `authenticate` + `requireRole('ADMIN')` ([09:36] Larissa):

1. Buscar a linha da DLQ → senão `WEBHOOK_DEAD_LETTER_NOT_FOUND` (404).
2. Se `replayedAt` já preenchido → `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` (409) *(decisão de design)*.
3. Em transação: voltar a linha da outbox para `PENDING`, `attemptCount = 0`, `nextAttemptAt = now`, **mantendo o mesmo `eventId`** ("recoloca na outbox como pendente", [09:18] Diego); preencher `replayedAt` e `replayedById`.
4. Logar `webhook_dlq_replayed` com `dlqId`, `eventId`, `replayedBy` (id e e-mail do usuário) ([09:36] Sofia: "logar quem fez o replay").

### FDD-FLUXO-06 — Rotação de secret

`POST /api/v1/webhooks/:id/rotate-secret`: gera nova secret (32 bytes aleatórios de `crypto.randomBytes`, codificados em hex **(decisão de design)**); move a atual para `previousSecret` com `previousSecretExpiresAt = now + 24h` ([09:21] Sofia); devolve a nova secret **uma única vez**. Durante a janela de 24 h, o worker assina com a **nova** secret e envia no `X-Signature` as duas assinaturas separadas por vírgula (`sha256=<nova>,sha256=<antiga>`), para que sistemas ainda não migrados validem com a antiga *(decisão de design; formato a validar com Sofia)*. Após `previousSecretExpiresAt`, `previousSecret` é ignorado e pode ser limpo.

### FDD-FLUXO-07 — Recuperação na partida do worker

Com worker único, qualquer linha `PROCESSING` encontrada ao iniciar é de uma execução interrompida; o worker as devolve para `PENDING` antes do primeiro ciclo *(decisão de design)*. Isso pode gerar reenvio de um evento que o cliente já recebeu, coberto pelo at-least-once ([ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md)).

---

## 6. Contratos públicos

Prefixo `/api/v1` (de `src/app.ts`). Todas as rotas exigem `Authorization: Bearer <JWT>` ([09:32] Marcos); o CRUD aceita qualquer role autenticada ([09:37] Sofia). `customerId` vem no body ou na query, **não do JWT** ([09:32] Larissa). Erros usam o envelope existente `{ "error": { "code", "message", "details?" } }`.

### FDD-CONTRATO-01 — `POST /api/v1/webhooks` (criar)

Request:

```json
{
  "customerId": "c1d2e3f4-0000-4000-8000-aaaaaaaaaaaa",
  "url": "https://erp.atlas.example.com/hooks/oms",
  "events": ["SHIPPED", "DELIVERED"]
}
```

Response `201 Created`:

```json
{
  "id": "b7a1c2d3-1111-4222-8333-444455556666",
  "customerId": "c1d2e3f4-0000-4000-8000-aaaaaaaaaaaa",
  "url": "https://erp.atlas.example.com/hooks/oms",
  "events": ["SHIPPED", "DELIVERED"],
  "active": true,
  "secret": "5f2b0c…(64 hex)",
  "createdAt": "2026-11-05T14:00:00.000Z"
}
```

Semântica: a secret é gerada pela plataforma e devolvida **na criação** ([09:31] Marcos); listagens não a retornam *(decisão de design)*. Status: `201`; `400` (`VALIDATION_ERROR`, `WEBHOOK_INVALID_URL`, `WEBHOOK_INVALID_EVENT`); `401`; `404` (`WEBHOOK_CUSTOMER_NOT_FOUND`).

### FDD-CONTRATO-02 — `GET /api/v1/webhooks?customerId=…` (listar)

Query: `customerId` (obrigatório, uuid), `page`, `pageSize` (padrão do projeto). Response `200 OK` (formato `paginated` de `src/shared/http/response.ts`):

```json
{
  "data": [
    {
      "id": "b7a1c2d3-1111-4222-8333-444455556666",
      "customerId": "c1d2e3f4-0000-4000-8000-aaaaaaaaaaaa",
      "url": "https://erp.atlas.example.com/hooks/oms",
      "events": ["SHIPPED", "DELIVERED"],
      "active": true,
      "createdAt": "2026-11-05T14:00:00.000Z",
      "updatedAt": "2026-11-05T14:00:00.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```

Status: `200`; `400`; `401`. Origem: [09:33] Bruno ("GET pra listar os webhooks de um customer").

### FDD-CONTRATO-03 — `PATCH /api/v1/webhooks/:id` (editar)

Request (todos os campos opcionais, ao menos um): `{ "url": "https://novo.atlas.example.com/hooks", "events": ["PAID","SHIPPED"], "active": false }`. Response `200 OK` com o objeto do webhook (mesmo formato de listagem, sem `secret`). Status: `200`; `400`; `401`; `404 WEBHOOK_NOT_FOUND`. A edição não altera eventos já enfileirados (snapshot, [ADR-001](adrs/ADR-001-outbox-no-mysql.md)). Origem: [09:33] Bruno.

### FDD-CONTRATO-04 — `DELETE /api/v1/webhooks/:id` (remover)

Response `204 No Content` (padrão de `OrderController.delete`). Status: `204`; `401`; `404 WEBHOOK_NOT_FOUND`. Origem: [09:33] Bruno.

### FDD-CONTRATO-05 — `GET /api/v1/webhooks/:id/deliveries` (histórico)

Query: `page`, `pageSize` (máx. 100; padrão 100 para atender "últimos 100", **decisão de design**). Ordenação: mais recentes primeiro. Response `200 OK`:

```json
{
  "data": [
    {
      "id": "e1e2e3e4-0000-4000-8000-000000000001",
      "eventId": "4b1f6a9e-2c1d-4e07-9d57-6a3c0f2b8e11",
      "attemptNumber": 2,
      "outcome": "FAILURE",
      "httpStatus": 503,
      "responseBody": "Service Unavailable",
      "durationMs": 412,
      "errorCode": "WEBHOOK_DELIVERY_FAILED",
      "payload": { "event_id": "4b1f6a9e-…", "event_type": "order.status_changed", "to_status": "SHIPPED" },
      "attemptedAt": "2026-11-05T14:33:11.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 100, "total": 1, "totalPages": 1 }
}
```

Campos conforme [09:34] Marcos: sucesso/falha, payload, response, tempo de resposta. Status: `200`; `401`; `404 WEBHOOK_NOT_FOUND`.

### FDD-CONTRATO-06 — `POST /api/v1/webhooks/:id/rotate-secret`

Sem body. Response `200 OK`:

```json
{
  "id": "b7a1c2d3-1111-4222-8333-444455556666",
  "secret": "9ad41e…(64 hex)",
  "previousSecretExpiresAt": "2026-11-06T14:00:00.000Z"
}
```

Status: `200`; `401`; `404 WEBHOOK_NOT_FOUND`. O requisito vem de [09:21] Sofia; o **caminho do endpoint é decisão de design**.

### FDD-CONTRATO-07 — `POST /api/v1/admin/webhooks/dead-letter/:id/replay`

Requer role `ADMIN`. Sem body. Response `200 OK`:

```json
{
  "id": "d0d1d2d3-0000-4000-8000-000000000009",
  "eventId": "4b1f6a9e-2c1d-4e07-9d57-6a3c0f2b8e11",
  "outboxStatus": "PENDING",
  "replayedAt": "2026-11-06T09:15:00.000Z",
  "replayedBy": "a0a1a2a3-0000-4000-8000-0000000000aa"
}
```

Status: `200`; `401`; `403 FORBIDDEN` (role insuficiente, de `requireRole`); `404 WEBHOOK_DEAD_LETTER_NOT_FOUND`; `409 WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED`. Origem: [09:18] Diego; [09:36] Sofia.

### FDD-CONTRATO-08 — Requisição de entrega ao cliente (saída)

`POST <url do webhook>` com o JSON de FLUXO-01 e headers ([09:44] Diego; [09:44] Sofia):

| Header | Valor |
| --- | --- |
| `Content-Type` | `application/json` |
| `X-Event-Id` | UUID do evento (estável entre reenvios) |
| `X-Signature` | `sha256=<hex>` do HMAC-SHA256 sobre o **corpo bruto** com a secret do endpoint (formato do prefixo `sha256=` é **decisão de design**) |
| `X-Timestamp` | momento do envio (ISO 8601) |
| `X-Webhook-Id` | `id` do cadastro de webhook |

Semântica de resposta: **2xx = entregue**; qualquer outro resultado (4xx, 5xx, timeout, erro de rede) = falha com retry. Redirecionamentos não são seguidos (`redirect: 'manual'`) *(decisão de design)*. Verificação no cliente (exemplo Node):

```ts
const expected = 'sha256=' + crypto.createHmac('sha256', secret).update(rawBody).digest('hex');
const ok = signatures.split(',').some((s) => crypto.timingSafeEqual(Buffer.from(s), Buffer.from(expected)));
```

---

## 7. Matriz de erros

Erros de API usam subclasses de `AppError` seguindo o exemplo de `InsufficientStockError` em `src/shared/errors/http-errors.ts`; o `error.middleware.ts` os serializa sem alteração ([09:29] Bruno). `NotFoundError` fixa `NOT_FOUND`, então `WEBHOOK_NOT_FOUND` exige subclasse própria de `AppError`.

| ID | Código | HTTP | Onde | Quando | Retry |
| --- | --- | --- | --- | --- | --- |
| FDD-ERRO-01 | `WEBHOOK_NOT_FOUND` | 404 | API | Webhook inexistente (get/patch/delete/deliveries/rotate). | — |
| FDD-ERRO-02 | `WEBHOOK_INVALID_URL` | 400 | API | URL malformada ou que não usa `https` ([09:23] Sofia). | — |
| FDD-ERRO-03 | `WEBHOOK_INVALID_EVENT` | 400 | API | Lista de eventos vazia ou com status que não pode ser destino de transição (`PENDING`). *(decisão de design, derivada de `order.status.ts`)* | — |
| FDD-ERRO-04 | `WEBHOOK_CUSTOMER_NOT_FOUND` | 404 | API | `customerId` informado não existe. | — |
| FDD-ERRO-05 | `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | API | Id de DLQ inexistente no replay. | — |
| FDD-ERRO-06 | `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` | 409 | API | DLQ já reprocessada. | — |
| FDD-ERRO-07 | `WEBHOOK_SECRET_REQUIRED` | — | Worker | Endpoint sem secret utilizável no momento de assinar ([09:28] Bruno cita o código). | Não (DLQ direto) |
| FDD-ERRO-08 | `WEBHOOK_PAYLOAD_TOO_LARGE` | — | Worker | Corpo acima de 64 KB: não envia e falha ([09:24] Larissa). | Não (DLQ direto) |
| FDD-ERRO-09 | `WEBHOOK_ENDPOINT_INACTIVE` | — | Worker | Webhook desativado após o evento ser enfileirado. *(decisão de design)* | Não (DLQ direto) |
| FDD-ERRO-10 | `WEBHOOK_DELIVERY_TIMEOUT` | — | Worker | Sem resposta em 10 s ([09:42] Diego). | Sim |
| FDD-ERRO-11 | `WEBHOOK_DELIVERY_FAILED` | — | Worker | Resposta fora de 2xx (inclui 3xx não seguido). | Sim |
| FDD-ERRO-12 | `WEBHOOK_DELIVERY_NETWORK_ERROR` | — | Worker | DNS, conexão recusada, TLS inválido. | Sim |

Os códigos da coluna "Worker" não são retornados por HTTP: ficam em `webhook_delivery_attempts.errorCode`, em `webhook_outbox.lastError` e em `webhook_dead_letter.failureReason`. Erros genéricos já existentes seguem valendo: `VALIDATION_ERROR` (400, Zod), `UNAUTHORIZED` (401), `FORBIDDEN` (403) e `INTERNAL_SERVER_ERROR` (500).

## 8. Estratégias de resiliência

| ID | Estratégia | Detalhe | Origem |
| --- | --- | --- | --- |
| FDD-RES-01 | Timeout | 10 s por chamada, via `AbortSignal.timeout(10_000)` no `fetch` nativo (Node ≥ 20, `engines` em `package.json`). | [09:42] Diego |
| FDD-RES-02 | Retry e backoff | 1m/5m/30m/2h/12h, até a DLQ (FLUXO-03). | [09:17] Larissa |
| FDD-RES-03 | Fallback | Falha permanente → DLQ persistida + replay manual por ADMIN; **não** há fallback por e-mail nesta fase. | [09:18] Diego; [09:37] Larissa |
| FDD-RES-04 | Isolamento de falhas | Falha de entrega nunca afeta `changeStatus` (desacoplada pela outbox); falha ao **inserir** na outbox derruba a transação por design. | [09:04] Bruno; [09:40] Bruno |
| FDD-RES-05 | Isolamento entre clientes | Paralelismo entre webhooks distintos dentro do ciclo (FLUXO-02). *(decisão de design)* | — |
| FDD-RES-06 | Recuperação de crash | Linhas `PROCESSING` voltam a `PENDING` na partida (FLUXO-07); duplicatas absorvidas pelo at-least-once. | [09:24] Diego |
| FDD-RES-07 | Shutdown graceful | `SIGINT`/`SIGTERM` terminam o ciclo corrente e desconectam o Prisma, como em `src/server.ts`. | `src/server.ts` |

## 9. Observabilidade

**Logs (Pino).** Sem biblioteca nova ([09:29] Bruno). Eventos estruturados, todos com `eventId`, `webhookId` e `outboxId` quando aplicável:

| ID | Evento | Campos principais |
| --- | --- | --- |
| FDD-OBS-01 | `webhook_event_enqueued` | `eventId`, `webhookId`, `orderId`, `toStatus` |
| FDD-OBS-02 | `webhook_delivery_attempt` | `attemptNumber`, `outcome`, `httpStatus`, `durationMs`, `errorCode` (nível `warn` em falha) |
| FDD-OBS-03 | `webhook_moved_to_dlq` / `webhook_dlq_replayed` | `failureReason`, `attemptCount`; no replay, `replayedBy` ([09:36] Sofia) |
| FDD-OBS-04 | `webhook_secret_rotated` | `webhookId`, `previousSecretExpiresAt` — **nunca** o valor das secrets. |

Para impedir vazamento acidental, `redactPaths` em `src/shared/logger/index.ts` deve ganhar `*.secret` e `*.previousSecret` (alteração a fazer na implementação; hoje cobre `authorization`, `cookie`, `password`, `token`).

**Métricas.** O projeto não tem biblioteca de métricas e a reunião não definiu stack. Proposta sem infraestrutura nova *(decisão de design)*: o worker emite, a cada N ciclos, um log `webhook_worker_stats` e as mesmas medidas ficam consultáveis por SQL nas tabelas:

| ID | Métrica | Uso |
| --- | --- | --- |
| FDD-OBS-05 | `outbox_pending_count` e `outbox_oldest_pending_age_seconds` | Detecta atraso: o requisito é < 10 s ([09:02] Marcos). |
| FDD-OBS-06 | `delivery_latency_ms` (por tentativa) e `end_to_end_latency_seconds` (`deliveredAt − createdAt` da outbox) | Verifica o objetivo FDD-OBJ-02. |
| FDD-OBS-07 | Taxa de sucesso e `attempts_per_event` | Dimensiona retry e clientes problemáticos. |
| FDD-OBS-08 | `dead_letter_count` / `dead_letter_replayed_count` | Alimenta a decisão adiada sobre e-mail e rate limit ([09:37]–[09:39] Larissa). |

**Tracing.** Não há OpenTelemetry no projeto nem foi discutido. A rastreabilidade ponta a ponta é feita por **correlação**: `eventId` aparece na linha da outbox, nas tentativas, nos logs, no payload e no header `X-Event-Id` (FDD-OBS-09); o cliente cita esse id ao abrir chamado. Tracing distribuído fica como evolução futura.

## 10. Integração com o sistema existente

| ID | Arquivo (existente) | Como o módulo de webhooks se integra |
| --- | --- | --- |
| FDD-INT-01 | `src/modules/orders/order.service.ts` | `changeStatus` passa a chamar `publishWebhookEvent(tx, order, from, to)` após `tx.orderStatusHistory.create` e dentro do mesmo `prisma.$transaction`. Nenhuma outra assinatura muda; `create` não é alterado ([09:40]–[09:41] Bruno). |
| FDD-INT-02 | `src/modules/orders/order.status.ts` | A máquina de estados continua sendo a fonte da verdade. Como nenhuma transição tem `PENDING` como destino, os eventos assináveis são `PAID`, `PROCESSING`, `SHIPPED`, `DELIVERED`, `CANCELLED` (base do `WEBHOOK_INVALID_EVENT`). |
| FDD-INT-03 | `src/shared/errors/http-errors.ts` e `src/shared/errors/index.ts` | Novas classes (`WebhookNotFoundError`, `WebhookInvalidUrlError`, …) estendem `AppError`/`ConflictError`/`BadRequestError` como `InsufficientStockError` estende `UnprocessableEntityError`, e são re-exportadas em `index.ts` ([09:28] Bruno). |
| FDD-INT-04 | `src/shared/errors/app-error.ts` | Base de todos os erros do módulo: `statusCode`, `errorCode` com prefixo `WEBHOOK_` e `details`. |
| FDD-INT-05 | `src/middlewares/auth.middleware.ts` | Rotas de webhook usam `authenticate`; o replay usa `requireRole('ADMIN')` ([09:36] Larissa). |
| FDD-INT-06 | `src/middlewares/validate.middleware.ts` | Schemas Zod do módulo passam por `validate({ body, params, query })`; a regra `https` fica no service para emitir `WEBHOOK_INVALID_URL`, pois o middleware converte todo `ZodError` em `VALIDATION_ERROR`. |
| FDD-INT-07 | `src/middlewares/error.middleware.ts` | **Sem alteração**: já serializa `AppError`, `ZodError` e erros do Prisma ([09:29] Bruno). |
| FDD-INT-08 | `src/routes/index.ts` | `buildApiRouter` monta `/webhooks` e `/admin/webhooks`, e o tipo `Controllers` ganha `webhooks`. |
| FDD-INT-09 | `src/app.ts` | `buildControllers` instancia `WebhookRepository`, `WebhookService`, `WebhookController` e injeta o que for necessário ao `OrderService`. |
| FDD-INT-10 | `src/server.ts` | Modelo de bootstrap/shutdown para o novo `src/worker.ts`; `package.json` ganha o script `worker` ([09:11] Larissa). |
| FDD-INT-11 | `src/config/database.ts` | O worker usa `createPrismaClient()` para ter instância própria no processo ([09:30] Bruno). |
| FDD-INT-12 | `src/config/env.ts` | Novas variáveis validadas por Zod, com defaults: `WEBHOOK_WORKER_POLL_INTERVAL_MS` (2000), `WEBHOOK_WORKER_BATCH_SIZE` (10), `WEBHOOK_DELIVERY_TIMEOUT_MS` (10000), `WEBHOOK_MAX_PAYLOAD_BYTES` (65536). |
| FDD-INT-13 | `src/shared/logger/index.ts` | Reutilizado pelo worker; `redactPaths` ampliado (ver seção 9). |
| FDD-INT-14 | `src/shared/http/response.ts` | `paginated()` nas listagens de webhooks e deliveries. |
| FDD-INT-15 | `prisma/schema.prisma` e `prisma/migrations/` | Novos modelos (seção 4), relação inversa em `Customer`, e nova migration. |
| FDD-INT-16 | `tests/setup.ts` e `tests/helpers/factories.ts` | O `beforeEach` precisa limpar as tabelas `webhook_*` antes de `customers`; criar `createTestWebhook()` nas factories. |

## 11. Dependências e compatibilidade

- **Dependências novas:** nenhuma. `fetch` e `AbortSignal.timeout` nativos (Node ≥ 20), `node:crypto` para HMAC e geração de secret, `uuid` já presente.
- **Banco:** MySQL 8.0 (`docker-compose.yml`), uma migration aditiva (sem alterar tabelas existentes além de relações inversas no Prisma).
- **Compatibilidade da API:** rotas novas e aditivas; as respostas existentes de `/orders` não mudam. `changeStatus` mantém a mesma assinatura e contrato HTTP.
- **Deploy:** passa a existir um segundo processo (`npm run worker`) com a mesma `DATABASE_URL`; `tsconfig.build.json` já inclui `src/**/*.ts`, então `src/worker.ts` é compilado para `dist/worker.js`.
- **Pré-requisito de processo:** revisão de segurança de Sofia (≥ 2 dias úteis) antes do deploy ([09:46] Sofia).

## 12. Critérios de aceite técnicos

| ID | Critério |
| --- | --- |
| FDD-ACE-01 | Mudar o status de um pedido cujo customer tem webhook interessado cria exatamente 1 linha `PENDING` na outbox por webhook elegível, na mesma transação; forçar erro no insert reverte o status do pedido. |
| FDD-ACE-02 | Se nenhum webhook do customer escuta o status, nenhuma linha é inserida. |
| FDD-ACE-03 | Um evento `PENDING` é enviado em ≤ 2 s + tempo de chamada, e o tempo total commit → entrega fica < 10 s com endpoint saudável. |
| FDD-ACE-04 | A requisição entregue contém `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` e `Content-Type`; a assinatura confere com HMAC-SHA256 do corpo. |
| FDD-ACE-05 | Falhas sucessivas agendam retentativas em 1m/5m/30m/2h/12h e, esgotadas, movem o evento para `webhook_dead_letter`. |
| FDD-ACE-06 | Replay de DLQ por `ADMIN` recoloca o evento como `PENDING` com o mesmo `eventId`; role `OPERATOR` recebe 403; o log traz o autor. |
| FDD-ACE-07 | Após rotação, ambas as secrets validam por 24 h e só a nova depois disso. |
| FDD-ACE-08 | URL `http://` é recusada com `WEBHOOK_INVALID_URL`; payload > 64 KB não é enviado e gera `WEBHOOK_PAYLOAD_TOO_LARGE`. |
| FDD-ACE-09 | Reiniciar a API não interrompe entregas; reiniciar o worker não perde eventos (FLUXO-07). |
| FDD-ACE-10 | `error.middleware.ts` permanece inalterado e todos os erros do módulo saem no envelope padrão. |

## 13. Riscos e mitigação

| ID | Risco | Prob. | Impacto | Mitigação |
| --- | --- | --- | --- | --- |
| FDD-RISK-01 | Cliente lento atrasa a fila e viola < 10 s. | Média | Alto | Paralelismo entre webhooks; timeout de 10 s; métrica `outbox_oldest_pending_age_seconds`. |
| FDD-RISK-02 | Retry reordena eventos do mesmo pedido (RFC-Q-06). | Média | Médio | Documentar; cliente usa `timestamp`/`to_status`; avaliar bloqueio por pedido. |
| FDD-RISK-03 | Secret armazenada recuperável pode vazar via banco/backup. | Baixa | Alto | Revisão de Sofia; proteção em repouso (RFC-Q-07); redaction em logs. |
| FDD-RISK-04 | Crescimento da outbox degrada o polling. | Média | Médio | Índices `(status, nextAttemptAt)` e `createdAt`; arquivamento planejado como próxima fase ([09:08] Diego). |
| FDD-RISK-05 | Remoção de webhook apaga histórico e DLQ via cascade. | Baixa | Médio | Aceito nesta fase; reavaliar soft delete. |
| FDD-RISK-06 | Sem checagem de posse: qualquer usuário autenticado pode cadastrar webhook para qualquer `customerId`. | Média | Alto | Decorre de "customer_id no body/path, qualquer role" ([09:32] Larissa; [09:37] Sofia); levar à revisão de segurança, com endurecimento já previsto. |
| FDD-RISK-07 | URL de destino aponta para rede interna (SSRF). | Baixa | Alto | Não discutido na reunião; levar à revisão de Sofia (bloqueio de IPs privados/loopback). |
| FDD-RISK-08 | Cliente não deduplica e processa duas vezes. | Média | Médio | Documentação no portal ([09:26] Marcos). |

### Decisões de design deste FDD (a validar na revisão)

Pontos que complementam a reunião: nomes e colunas das tabelas além das citadas; uma linha de outbox por (evento, webhook); significado de `FAILED`; formato `sha256=` e assinatura dupla na rotação; caminho `rotate-secret`; paralelismo entre webhooks; falha permanente direta para DLQ; recuperação de `PROCESSING`; `WEBHOOK_INVALID_EVENT`, `WEBHOOK_ENDPOINT_INACTIVE` e demais códigos de erro além de `NOT_FOUND`/`INVALID_URL`/`SECRET_REQUIRED`; paginação padrão de 100 em deliveries; `ON DELETE CASCADE`; métricas por log/SQL.
