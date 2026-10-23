# Architectural Decision Records

Este diretório armazena os ADRs (Architectural Decision Records) da feature de Webhooks de Notificação de Pedidos.
Cada decisão arquitetural fechada na reunião técnica é registrada em um arquivo individual, nomeado
`ADR-NNN-titulo-em-kebab-case.md` e escrito no formato MADR (Status, Contexto, Decisão, Alternativas Consideradas, Consequências).

| ADR | Decisão |
| --- | --- |
| [ADR-001](ADR-001-outbox-no-mysql.md) | Padrão Outbox no MySQL com publicação na mesma transação |
| [ADR-002](ADR-002-worker-separado-em-polling.md) | Worker em processo separado com polling de 2 segundos |
| [ADR-003](ADR-003-retry-backoff-e-dlq.md) | Retry com backoff exponencial e DLQ em tabela separada |
| [ADR-004](ADR-004-hmac-sha256-secret-por-endpoint.md) | HMAC-SHA256 com secret por endpoint e rotação com grace period |
| [ADR-005](ADR-005-at-least-once-com-x-event-id.md) | Entrega at-least-once com `X-Event-Id` |
| [ADR-006](ADR-006-reuso-dos-padroes-do-projeto.md) | Reuso dos padrões existentes do projeto |
| [ADR-007](ADR-007-contrato-de-entrega-payload-e-headers.md) | Contrato de entrega: payload, headers, timeout e limite de tamanho |

Visão geral da proposta: [RFC](../RFC.md). Como construir: [FDD](../FDD.md). Origem de cada item: [Tracker](../TRACKER.md).
