# ADR-003: Retry com backoff exponencial e DLQ em tabela separada

## Status

Aceito — decidido na reunião técnica ([09:17] Larissa: "Decidido: 5 tentativas, backoff 1m/5m/30m/2h/12h").

## Contexto

O endpoint do cliente pode estar indisponível ou lento. É preciso definir quantas vezes tentar de novo, com que intervalos, o que fazer ao esgotar as tentativas e como recuperar eventos que falharam em definitivo. Havia o dado concreto de que "já tinha cliente nosso com indisponibilidade de duas horas em manutenção planejada" ([09:16] Diego).

## Decisão

1. **Backoff exponencial** com os intervalos **1 min, 5 min, 30 min, 2 h e 12 h**, "total de quase 15 horas entre primeira falha e última tentativa" ([09:17] Diego).
2. **Teto de 5 tentativas**; ao esgotá-las o evento é considerado falha permanente e movido para a DLQ ([09:15] Diego; [09:17] Larissa).
3. A **DLQ é uma tabela separada, `webhook_dead_letter`**, com payload, motivo da falha e timestamp, servindo de evidência para debug e reprocessamento ([09:18] Diego).
4. **Replay manual** via `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente ([09:18] Diego).
5. O replay exige **role `ADMIN`** reutilizando o middleware `requireRole`, e **o autor do replay é registrado em log** para auditoria ([09:36] Sofia, Larissa).
6. Timeout de 10 segundos por chamada: resposta que não chega em 10 s conta como falha e vai para retry ([09:42] Diego).

*Interpretação do autor (ponto a confirmar, ver RFC):* a transcrição fala em "5 tentativas" e lista 5 intervalos; adotamos **1 envio inicial + 5 retentativas** (um intervalo por retentativa), que é o que produz o total de ~14h36 entre a primeira falha e a última tentativa citado por Diego.

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
| --- | --- |
| Retry indefinido com backoff | Evento "ficar pendurado pra sempre se o cliente sumiu" ([09:15] Diego). |
| Apenas 3 tentativas ("mais agressivo") | Pouco: uma indisponibilidade de manhã esgotaria as tentativas em ~30 min; já houve manutenção planejada de 2 h ([09:16] Diego). |
| Marcar como `failed` na própria outbox, sem tabela de DLQ | Poluiria a leitura da outbox principal; a tabela separada fica "mais limpa" e vira evidência de debug ([09:18] Diego). |
| Reprocessamento automático da DLQ | Não proposto; a reunião definiu replay **manual** por admin ([09:18] Diego). |

## Consequências

**Positivas**

- Janela de ~15 h cobre indisponibilidades longas e manutenções planejadas ([09:17] Marcos considera aceitável).
- Falhas permanentes ficam isoladas, consultáveis e reprocessáveis sem tocar a outbox principal.
- Auditoria do replay atende a exigência de segurança ([09:36] Sofia).

**Negativas / trade-offs**

- Evento de cliente fora do ar por mais de ~15 h exige intervenção humana (replay).
- Retries combinados com polling single-worker podem reordenar eventos de um mesmo pedido (ver [ADR-002](ADR-002-worker-separado-em-polling.md)).
- Cada retentativa reenvia o mesmo evento, por isso o cliente precisa deduplicar ([ADR-005](ADR-005-at-least-once-com-x-event-id.md)).
- Notificar o cliente por e-mail quando o webhook falha repetidamente foi explicitamente **adiado** ([09:37] Larissa): hoje só o replay manual e a observação do operador tratam a falha.
- Rate limiting de saída não faz parte desta decisão (ponto em aberto, [09:39] Larissa).

## Referências no código

- `src/middlewares/auth.middleware.ts` — `requireRole('ADMIN')` e `authenticate` usados no endpoint de replay.
- `src/shared/logger/index.ts` — logger Pino usado para registrar quem executou o replay.
