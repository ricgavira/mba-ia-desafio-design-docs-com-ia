# ADR-004: Autenticação HMAC-SHA256 com secret por endpoint e rotação com grace period

## Status

Aceito — decidido na reunião técnica ([09:22] Sofia: "Decidido: HMAC-SHA256 sobre o corpo do request, secret por endpoint, suporte a rotação com grace period de 24h").

## Contexto

Os eventos carregam dados de pedidos para um endpoint fora da infraestrutura da empresa. O cliente precisa validar que a requisição veio da plataforma e que ninguém adulterou o payload no caminho ([09:19] Sofia). Houve histórico de vazamento: "a gente já teve cliente que vazou secret em log de aplicação dele uma vez" ([09:22] Diego).

## Decisão

1. **Assinatura HMAC-SHA256 calculada sobre o corpo do request**, enviada no header `X-Signature` ([09:20] Sofia).
2. **Uma secret única por endpoint de webhook**, nunca uma secret global da plataforma: "se vaza uma, vaza tudo" ([09:21] Sofia).
3. A tabela de configuração guarda `url`, `secret`, `customer_id` e estado ativo ([09:21] Bruno, confirmado por Sofia).
4. A secret é **gerada pela plataforma e devolvida na criação** do webhook ([09:31] Marcos).
5. **Rotação pela API**: o cliente pede uma nova secret; a antiga continua válida por **24 horas** em paralelo, depois "morre" ([09:21] Sofia).
6. **TLS obrigatório**: URL `http://` é recusada com erro de validação. É uma validação no schema Zod, não uma decisão arquitetural separada ([09:23] Sofia).
7. O header `X-Timestamp` (momento do envio) acompanha cada requisição para o cliente poder detectar replay attack, se quiser ([09:44] Diego).
8. A geração da secret e o código de HMAC serão revisados por Sofia antes do deploy, com pelo menos dois dias úteis reservados ([09:46] Sofia).

## Alternativas Consideradas

| Alternativa | Por que foi descartada |
| --- | --- |
| Secret global da plataforma | Um vazamento comprometeria todos os clientes ([09:21] Sofia). |
| Rotação com corte imediato da secret antiga | Não dá tempo ao cliente para migrar seus sistemas; por isso o grace period de 24 h ([09:21] Sofia). |
| Aceitar URLs `http://` | Dados de pedido trafegariam sem TLS; recusado na validação ([09:23] Sofia). |
| Outros esquemas de autenticação (por exemplo, token estático no header) | *Alternativa plausível, não discutida:* não oferece integridade do corpo, que é o requisito explícito de Sofia ([09:19]). |

## Consequências

**Positivas**

- Autenticidade e integridade verificáveis pelo cliente com biblioteca padrão ("todo cliente sério tem biblioteca pra isso", [09:20] Sofia).
- Vazamento de uma secret afeta um único endpoint e é remediável por rotação sem janela de indisponibilidade.

**Negativas / trade-offs**

- Como o servidor precisa **assinar**, a secret é armazenada de forma recuperável (não pode ser só um hash). A proteção em repouso dessa coluna não foi discutida e é tema da revisão de segurança (questão em aberto no [RFC](../RFC.md)).
- Durante a rotação, o sistema precisa manter duas secrets por endpoint por 24 h, aumentando a complexidade do modelo e do envio (detalhado no [FDD](../FDD.md)).
- Pela decisão tomada, a assinatura cobre o **corpo**; `X-Timestamp` não é coberto por ela, então o cliente que quiser proteção contra replay depende da própria política (*observação do autor*).
- Exige revisão de segurança antes do deploy, consumindo ~2 dias úteis do cronograma ([09:46] Sofia).

## Referências no código

- `src/middlewares/validate.middleware.ts` e `src/modules/*/**.schemas.ts` — validação Zod onde a regra `https` será aplicada.
- `src/config/env.ts` — padrão de configuração validada por Zod, aplicável a eventuais chaves de proteção da secret.
