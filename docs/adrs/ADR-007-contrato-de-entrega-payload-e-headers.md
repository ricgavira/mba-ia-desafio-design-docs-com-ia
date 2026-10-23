# ADR-007: Contrato de entrega: payload enxuto, headers padronizados, timeout e limite de tamanho

## Status

Aceito — decidido na reunião técnica ([09:43]–[09:45] Diego, Sofia; [09:24] Larissa; [09:42] Diego).

## Contexto

O cliente precisa receber um corpo previsível e headers suficientes para autenticar, deduplicar e correlacionar o envio. Sofia levantou um limite de tamanho: um evento de 500 KB indicaria algo errado ([09:23] Sofia). O worker não pode ficar pendurado em clientes lentos ([09:42] Sofia, Diego).

## Decisão

1. **Payload JSON enxuto**: `event_id`, `event_type` (por exemplo `"order.status_changed"`), `timestamp` ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e campos básicos da order como `total_cents`. **Não inclui `items`**; detalhes via `GET /orders/:id` ([09:43] Diego).
2. **Headers de cada requisição**: `X-Event-Id`, `X-Signature`, `X-Timestamp` (momento do envio), `X-Webhook-Id` (id do endpoint cadastrado, sugerido por Sofia) e `Content-Type: application/json` ([09:44] Diego; [09:44] Sofia; [09:45] Diego).
3. **Timeout de 10 segundos** por chamada; estourou, é falha e entra em retry ([09:42] Diego).
4. **Limite de 64 KB** por payload. Acima disso o evento **não é enviado e gera erro**, sem truncar ([09:23] Sofia; [09:24] Diego, Larissa). É tratado como requisito não funcional, não como decisão arquitetural separada, mas registrado aqui por compor o contrato.

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
| --- | --- |
| Incluir os itens do pedido no payload | "Não manda items pra não inflar" ([09:43] Diego). |
| Truncar o payload acima do limite | Sofia é "a favor de erra": se chegou àquele tamanho, algo está errado ([09:23] Sofia). |
| Limite de 500 KB ou maior | Diego considera 64 KB "um teto generoso", e nenhum evento se aproxima disso ([09:24] Diego). |
| Sem `X-Webhook-Id` | Cliente com vários cadastros não saberia qual deles originou o envio ([09:44] Sofia). |

## Consequências

**Positivas**

- Payload pequeno reduz banda, tempo de resposta e risco de estourar o limite.
- Os headers permitem ao cliente autenticar ([ADR-004](ADR-004-hmac-sha256-secret-por-endpoint.md)), deduplicar ([ADR-005](ADR-005-at-least-once-com-x-event-id.md)), detectar replay e saber qual cadastro recebeu o envio.

**Negativas / trade-offs**

- Clientes que precisam dos itens fazem uma chamada adicional a `GET /orders/:id`, e esse GET retorna o estado **atual**, que pode ser diferente do snapshot do evento ([ADR-001](ADR-001-outbox-no-mysql.md)).
- Um evento acima de 64 KB nunca chega ao cliente e vira falha, em vez de uma entrega degradada.
- O timeout fixo de 10 s é igual para todos os clientes (não configurável por endpoint), pois a reunião não previu configuração por cliente.

## Referências no código

- `prisma/schema.prisma` — `Order.totalCents` (origem de `total_cents`), `Order.orderNumber` (`order_number`), `OrderStatus` (valores de `from_status`/`to_status`).
