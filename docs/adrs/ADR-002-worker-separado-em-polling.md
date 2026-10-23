# ADR-002: Worker em processo separado com polling de 2 segundos

## Status

Aceito — decidido na reunião técnica ([09:10] Larissa: "Worker em polling, 2s").

## Contexto

As linhas da `webhook_outbox` ([ADR-001](ADR-001-outbox-no-mysql.md)) precisam ser lidas e enviadas por HTTP. Perguntas colocadas na mesa:

- Como o worker descobre eventos novos? Bruno sugeriu trigger de banco "pra ser mais reativo" ([09:09] Bruno).
- Onde o worker roda? Diego apontou que, se rodasse dentro da API, "se a API reinicia, perde o worker" ([09:11] Diego).
- Qual o requisito de latência? Para os clientes, "qualquer coisa abaixo de 10 segundos já é tempo real" ([09:02] Marcos).
- E a ordem de entrega dos eventos de um mesmo pedido? ([09:12] Larissa).

A aplicação hoje tem um único entry point, `src/server.ts`, que sobe o Express e trata `SIGINT`/`SIGTERM`.

## Decisão

1. **Polling em loop a cada 2 segundos**: a cada ciclo, o worker busca os eventos pendentes mais antigos (por `created_at`) em lote pequeno, processa e atualiza o status ([09:09] Diego, [09:08] Diego).
2. **Processo separado da API**, com entry point novo `src/worker.ts` e script `npm run worker` ([09:11] Diego, Larissa).
3. O worker usa o **mesmo banco e a mesma stack**, mas com **instância própria do `PrismaClient`**, porque o client é por processo (mesma `DATABASE_URL`) ([09:11] Bruno; [09:30] Bruno).
4. A lógica de processamento fica dentro do módulo (`src/modules/webhooks/webhook.worker.ts` ou `webhook.processor.ts`); `src/worker.ts` é apenas a entry ([09:28] Bruno).
5. **Worker único (single-worker)**. A ordenação é "implícita por `order_id`", pela ordem de `created_at` da outbox, e só vale enquanto houver um worker ([09:12] Diego). Isso é documentado como **limitação conhecida**, sem garantia de ordering global ([09:13] Larissa; [09:14] Marcos confirma que os clientes nunca pediram isso).

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
| --- | --- |
| Trigger de banco para acordar o worker | MySQL não tem `LISTEN/NOTIFY` como o Postgres; trigger só executa SQL e não notifica processo externo, exigindo improviso (arquivo, endpoint) ([09:09] Diego). |
| Worker dentro do processo da API | Reinício da API perderia o worker ([09:11] Diego). |
| Múltiplos workers em paralelo desde já | Perde a garantia de ordem; escalar exigiria particionar por `order_id` ou lock pessimista, tratado como "problema do futuro" ([09:12]–[09:13] Diego). |
| Intervalo de polling menor / reativo | Não necessário: 2 s atende o requisito de menos de 10 s "tranquilo" ([09:09] Diego; [09:10] Marcos "2 segundos serve"). |

## Consequências

**Positivas**

- Latência previsível: no pior caso o evento espera 2 s para ser pego, aceito explicitamente ([09:10] Larissa), bem abaixo dos 10 s exigidos pelos clientes.
- Isolamento de falhas: reiniciar a API não interrompe entregas, e um worker travado não derruba a API.
- Reaproveita Prisma, `DATABASE_URL`, Pino e o padrão de encerramento graceful de `src/server.ts`.

**Negativas / trade-offs**

- Um processo a mais para operar (deploy, monitoramento, restart).
- O polling gera consultas periódicas mesmo sem eventos, tolerado pelo índice em `status`/`created_at` ([09:08] Diego).
- Single-worker é ponto único de throughput; escalar exige novo ADR.
- *Análise do autor, não discutida na reunião:* enquanto um evento de um pedido aguarda retry ([ADR-003](ADR-003-retry-backoff-e-dlq.md)), um evento posterior do mesmo pedido pode ser entregue antes dele. A garantia de ordem "por `order_id`" portanto não cobre eventos em retry. Registrado como questão em aberto no [RFC](../RFC.md).

## Referências no código

- `src/server.ts` — padrão de bootstrap e shutdown (`SIGINT`/`SIGTERM`) a ser espelhado em `src/worker.ts`.
- `src/config/database.ts` — `createPrismaClient()` exportado, reutilizável para a instância própria do worker.
