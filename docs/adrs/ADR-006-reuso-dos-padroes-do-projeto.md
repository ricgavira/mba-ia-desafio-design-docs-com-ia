# ADR-006: Reuso dos padrões existentes do projeto no módulo de webhooks

## Status

Aceito — decidido na reunião técnica ([09:30] Larissa: "Decisão: reuso máximo do que já existe").

## Contexto

O OMS já tem convenções consolidadas. Bruno as resumiu: cada domínio é um módulo em `src/modules` com controller, service, repository, routes e schemas ([09:27] Bruno); erros herdam de `AppError` com códigos como `INSUFFICIENT_STOCK` e `INVALID_STATUS_TRANSITION` ([09:28] Bruno); o logger é Pino e o middleware de erro centralizado já trata `AppError`, Zod e Prisma ([09:29] Bruno). A decisão é seguir essas convenções em vez de introduzir novas.

## Decisão

O módulo de webhooks reutiliza integralmente:

| Padrão | Como é reutilizado | Origem no código |
| --- | --- | --- |
| Estrutura modular | Novo módulo `src/modules/webhooks` com controller, service, repository, routes e schemas ([09:27] Bruno). | `src/modules/orders/` |
| Entry point separado | `src/worker.ts` e lógica em `src/modules/webhooks/webhook.worker.ts` (ou `webhook.processor.ts`) ([09:28] Bruno). | `src/server.ts` |
| `AppError` e subclasses | Erros do módulo estendem `AppError`/`ConflictError` etc. ([09:28] Bruno). | `src/shared/errors/app-error.ts`, `http-errors.ts` |
| Prefixo de códigos | Todos os códigos do módulo usam `WEBHOOK_` (`WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED` etc.) ([09:28] Bruno, [09:29] Larissa). | padrão `INSUFFICIENT_STOCK` |
| Middleware de erro | Nenhuma mudança: já serializa `AppError` ([09:29] Bruno). | `src/middlewares/error.middleware.ts` |
| Logger Pino | Sem biblioteca nova de log ([09:29] Bruno). | `src/shared/logger/index.ts` |
| Schemas Zod | Validação de entrada via `validate()`; a regra de URL `https` é um schema Zod ([09:23] Sofia). | `src/middlewares/validate.middleware.ts` |
| Autorização | `authenticate` em todas as rotas; `requireRole('ADMIN')` no replay ([09:36] Larissa). | `src/middlewares/auth.middleware.ts` |
| IDs UUID | Tabelas novas usam UUID como chave primária ([09:51] Larissa). | `prisma/schema.prisma` |
| PrismaClient | Mesmo banco e mesma `DATABASE_URL`; instância própria no processo do worker ([09:29] Diego, [09:30] Bruno). | `src/config/database.ts` |

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
| --- | --- |
| Biblioteca/serviço dedicado de filas e eventos | Contradiz a decisão de reuso máximo e adiciona infraestrutura ([09:07] Diego; [09:30] Larissa). |
| Códigos de erro sem prefixo ou genéricos (`NOT_FOUND`) | Perde a identificação do módulo; Larissa fixou o prefixo `WEBHOOK_` para tudo do módulo ([09:29] Larissa). |
| Injetar o repository de webhooks no `OrderService` | Descartada em favor de uma função `publishWebhookEvent(tx, ...)` que recebe o `tx` ([09:41] Diego: "não precisa injetar repository inteiro"). |
| Logger específico para o worker | Pino já está "no projeto inteiro. Não vamos botar nada novo" ([09:29] Bruno). |

## Consequências

**Positivas**

- Curva de aprendizado e revisão menores: quem conhece `orders` navega em `webhooks`.
- Respostas de erro idênticas às do resto da API (`{ error: { code, message, details } }`), sem alterar `error.middleware.ts`.
- Nenhuma dependência nova.

**Negativas / trade-offs**

- O módulo herda as limitações dos padrões atuais (por exemplo, `NotFoundError` fixa o código `NOT_FOUND`, então `WEBHOOK_NOT_FOUND` exige subclasse própria, como `InsufficientStockError` já faz).
- O worker, ao reutilizar `src/config/env.ts`, herda o `process.exit(1)` em configuração inválida e a validação da `JWT_SECRET`, mesmo sem usar JWT.
- `OrderService.changeStatus` passa a depender de uma função do módulo de webhooks (acoplamento aceito).
- O CRUD de configuração aceita qualquer role autenticada "por enquanto", com possível endurecimento futuro ([09:37] Sofia).

## Referências no código

`src/modules/orders/order.service.ts`, `src/shared/errors/http-errors.ts`, `src/middlewares/error.middleware.ts`, `src/middlewares/auth.middleware.ts`, `src/shared/logger/index.ts`, `src/config/database.ts`, `src/server.ts`.
