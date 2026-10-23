# ADR-001: Padrão Outbox no MySQL com publicação na mesma transação

## Status

Aceito — decidido na reunião técnica ([09:08] Larissa: "Tá decidido então: outbox em MySQL").

## Contexto

Três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) precisam ser notificados quando o status de um pedido muda, hoje consultando `GET /orders` repetidamente ([09:00] Marcos). O ponto de captura do evento é o método `changeStatus` de `src/modules/orders/order.service.ts`, que já executa em uma única `prisma.$transaction`: atualiza `orders`, insere em `order_status_history` e debita/repõe `stock_quantity` ([09:04] Bruno).

Duas restrições aparecem na reunião:

- Uma chamada HTTP dentro dessa transação travaria mudanças de status de outros pedidos por causa de um cliente lento, e um cliente fora do ar não pode causar rollback da mudança de status ([09:04] Bruno).
- "Não pode ter caso de status mudar e evento não sair" ([09:40] Bruno; [09:41] Diego: "se ficar fora da transação, perde a garantia toda").

O time é pequeno e já opera MySQL; subir nova infraestrutura foi tratado como excesso ([09:07] Diego).

## Decisão

Adotar o **padrão Outbox na base MySQL existente**:

1. Dentro da mesma transação de `changeStatus`, inserir uma linha em `webhook_outbox` para cada endpoint de webhook interessado na transição. Se o insert falhar, a transação inteira sofre rollback ([09:40] Bruno).
2. A inserção é feita por uma função `publishWebhookEvent(tx, order, fromStatus, toStatus)` que recebe o `tx` da transação corrente, sem injetar o repository de webhooks no `OrderService` ([09:41] Bruno e Diego).
3. O evento é guardado já **renderizado (snapshot do payload no momento da inserção)**, para refletir o estado do pedido quando o status mudou ([09:52] Larissa, Diego, Bruno).
4. O filtro por status de interesse do endpoint é aplicado **na inserção**: se nenhum webhook do customer quer aquele status, nada é inserido ([09:34] Bruno, Diego).
5. A tabela tem índices em `status` e `created_at`, e a chave primária é UUID, como no restante do projeto ([09:08] Diego; [09:51] Larissa).
6. O status do evento na outbox assume os valores pendente, processando, falhou e entregue ([09:08] Diego).

O processamento dessas linhas é tratado em [ADR-002](ADR-002-worker-separado-em-polling.md).

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
| --- | --- |
| Disparo HTTP síncrono dentro do `OrderService` | Cliente lento trava a transação; cliente offline forçaria rollback ou perda de evento ([09:04] Bruno; [09:06] Diego: "fora de questão"). |
| Fila externa (Redis Streams ou similar) | Exige nova infraestrutura para um time pequeno ("overengineering", [09:07] Diego; [09:07] Larissa). Além disso o evento seria publicado fora da transação SQL, perdendo a atomicidade. |
| Guardar apenas `order_id` e renderizar o payload no envio | O evento passaria a refletir o estado do pedido no momento do envio, não da mudança ([09:52] Larissa: "caso esquisito"). |
| Filtrar por status no momento do envio | Inflaria a tabela com linhas que nunca serão enviadas ([09:34] Bruno). |
| ID auto-incremental na outbox | Foge do padrão do projeto, onde todo id é UUID (`@db.Char(36)` em `prisma/schema.prisma`) ([09:51] Larissa). |

## Consequências

**Positivas**

- Atomicidade: se a mudança de status confirmou, o evento existe; se deu rollback, o evento some junto ([09:06] Diego).
- Zero infraestrutura nova: usa o MySQL e o Prisma já presentes.
- O evento é imutável e auditável como "o que o cliente deveria ter recebido".

**Negativas / trade-offs**

- A transação de `changeStatus`, já descrita como pesada ([09:04] Bruno), ganha mais um `INSERT` por webhook interessado. Troca-se um pouco de latência de escrita por consistência.
- A outbox cresce com o uso; o arquivamento de linhas entregues (mencionado como "30 dias ou assim") ficou **fora do escopo** desta feature ([09:08] Diego).
- Snapshot significa que um evento reenviado ou reprocessado mostra o estado antigo do pedido; o cliente que quiser o estado atual consulta `GET /orders/:id` ([09:43] Diego).
- O `OrderService` passa a conhecer a existência de eventos de webhook (via a função auxiliar), acoplamento aceito explicitamente no lugar de injetar um repository.

## Referências no código

- `src/modules/orders/order.service.ts` — método `changeStatus` (ponto de extensão).
- `prisma/schema.prisma` — padrão de ids `@id @default(uuid()) @db.Char(36)` e `@@map` em snake_case.
