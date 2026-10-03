# ADR-005 — Garantia de entrega at-least-once com X-Event-Id para deduplicação

- **Status:** Aceito
- **Data da decisão:** Reunião técnica de quinta-feira, 09:00. A transcrição não registra a data de calendário.
- **Decisores:** Diego (Eng. Sênior, Plataforma), Larissa (Tech Lead), Sofia, Marcos
- **Decisões do inventário cobertas:** D10

## Contexto

A combinação de outbox ([ADR-001](ADR-001-outbox-no-mysql.md)), worker com polling ([ADR-002](ADR-002-worker-separado-em-polling.md)) e retry ([ADR-003](ADR-003-retry-com-backoff-e-dlq.md)) abre a possibilidade de o mesmo evento ser enviado mais de uma vez. Por exemplo: o cliente processa a requisição, mas a resposta não chega antes do timeout de 10s (RNF6), e o evento é reenviado. Diego deixou explícito: "Pode acontecer de o cliente receber o mesmo evento duas vezes. Ele tem que estar preparado" ([09:24]).

## Decisão

A plataforma garante entrega **at-least-once**. Cada evento recebe um **UUID gerado no momento em que entra na outbox**, enviado no header **`X-Event-Id`**. O valor é único por evento, e **o cliente deduplica** por ele.

Origens da decisão:

- [09:24] Diego ("a gente vai garantir at-least-once").
- [09:25] Diego ("X-Event-Id, com um UUID gerado quando o evento entra na outbox […] ele dedupica pelo event_id do lado dele").
- [09:26] Larissa: "At-least-once com X-Event-Id pra dedup do lado do cliente. Decisão."

## Alternativas Consideradas

1. **Exactly-once.** Descartada. "Garantir exactly-once exigiria coordenação dos dois lados e fica muito mais complexo. At-least-once com event_id resolve 99% dos casos" ([09:25] Diego).
2. **At-most-once (enviar uma vez, sem retry).** *Alternativa plausível proposta neste ADR; não foi discutida como tal.* Ela eliminaria duplicatas, mas é incompatível com o retry já decidido (D6) e com o objetivo de não perder notificações durante indisponibilidades do cliente ([09:14–09:17]).

## Consequências

### Positivas

- **Alinhado ao mercado.** "Stripe faz assim, GitHub faz assim" ([09:25] Diego), e os integradores já conhecem o modelo.
- **Encaixa de forma natural na outbox e no retry.** Não é preciso coordenar estado com o cliente.
- **Barato de implementar.** A biblioteca `uuid` já é dependência do projeto e gera ids v4 em `src/middlewares/request-logger.middleware.ts:2,6`. O inventário registra como *possibilidade*, não como decisão, que o próprio PK UUID da outbox (D21) sirva de `event_id`.

### Negativas

- **A responsabilidade passa para o cliente.** "Isso joga responsabilidade pro cliente" ([09:25] Sofia). Um cliente que não deduplique pode processar o mesmo evento duas vezes.
- **Exige documentação clara para os integradores.** Marcos assumiu: "Eu posso documentar isso bem destacado no portal de desenvolvedor" ([09:26]).
- **Duplicatas somadas à ordenação só por `order_id`** ([ADR-002](ADR-002-worker-separado-em-polling.md)). O cliente deve tolerar repetição e não pode contar com ordem global.

### Trade-off explícito

Ganha simplicidade e robustez do lado da plataforma, sem coordenação bilateral. Perde a garantia de não duplicação, e o custo da deduplicação passa para cada cliente.

### Pontos em aberto (não decididos na reunião)

- **Replay da DLQ:** não se definiu se o evento recolocado na outbox ([ADR-003](ADR-003-retry-com-backoff-e-dlq.md)) mantém o mesmo `X-Event-Id`. Se ganhar um novo, a deduplicação do cliente não reconhece o reenvio.
- **A2:** o `X-Timestamp` não entra no HMAC ([ADR-004](ADR-004-hmac-sha256-com-secret-por-endpoint.md)). A detecção de replay pelo cliente depende do `X-Event-Id` e não do timestamp.

## Referências

- `src/middlewares/request-logger.middleware.ts:2,6`: uso atual de `uuid` v4.
- Inventário: D10, D21; RNF6; A2; seção "g" (`X-Event-Id` gerado na inserção na outbox).
