# ADR-005: Garantia de entrega at-least-once com X-Event-Id para deduplicação

## Status

Aceito — decidido na reunião técnica ([09:26] Larissa: "At-least-once com X-Event-Id pra dedup do lado do cliente. Decisão").

## Contexto

Com retry ([ADR-003](ADR-003-retry-backoff-e-dlq.md)) e um worker que pode falhar entre enviar e marcar como entregue, o mesmo evento pode chegar mais de uma vez ao cliente. É preciso fixar a semântica de entrega e dar ao cliente um jeito de reconhecer duplicatas.

## Decisão

1. A plataforma garante **entrega at-least-once**: "o cliente pode receber o mesmo evento duas vezes; ele tem que estar preparado" ([09:24] Diego).
2. Cada evento recebe um **UUID gerado quando entra na outbox**, enviado no header **`X-Event-Id`** e também no campo `event_id` do payload ([09:25] Diego; [09:43] Diego).
3. O cliente **deduplica pelo `event_id` do seu lado** ([09:25] Diego).
4. Marcos documentará essa responsabilidade "bem destacada" no portal de desenvolvedor ([09:26] Marcos).
5. Reenvios (retry e replay de DLQ) mantêm o mesmo `event_id`. *(Consequência natural da decisão: o id pertence ao evento, não à tentativa.)*

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
| --- | --- |
| Exactly-once | "Exigiria coordenação dos dois lados e fica muito mais complexo" ([09:25] Diego). |
| At-most-once (sem retry) | *Alternativa plausível, não discutida:* contradiz o requisito de retry com DLQ e de não perder evento ([09:04] Bruno; [09:15] Diego). |
| Deduplicação do lado da plataforma | *Alternativa plausível, não discutida:* a plataforma não consegue saber se o cliente processou a requisição cuja resposta se perdeu; só o cliente pode deduplicar com segurança. |

## Consequências

**Positivas**

- Padrão de mercado ("Stripe faz assim, GitHub faz assim", [09:25] Diego), simples de implementar e de explicar.
- "Resolve 99% dos casos" segundo Diego ([09:25]), sem coordenação bilateral.

**Negativas / trade-offs**

- A responsabilidade de idempotência é transferida ao cliente ([09:25] Sofia: "isso joga responsabilidade pro cliente"), e a comunicação precisa ser eficaz (portal de desenvolvedor).
- Clientes que não deduplicarem podem processar um evento duas vezes.
- Como o payload é um snapshot ([ADR-001](ADR-001-outbox-no-mysql.md)), um duplicado traz exatamente os mesmos dados, o que facilita a deduplicação.

## Referências no código

- `package.json` — a dependência `uuid` (`v4`) já existe e é usada em `src/middlewares/request-logger.middleware.ts` para gerar ids; os UUIDs de evento seguem o mesmo padrão.
