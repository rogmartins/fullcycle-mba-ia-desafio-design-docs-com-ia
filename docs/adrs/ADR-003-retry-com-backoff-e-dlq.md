# ADR-003 — Retry com backoff exponencial, DLQ em tabela separada e replay manual

- **Status:** Aceito
- **Data da decisão:** Reunião técnica de quinta-feira, 09:00. A transcrição não registra a data de calendário.
- **Decisores:** Larissa (Tech Lead), Diego (Eng. Sênior, Plataforma), Bruno (Eng. Pleno, Pedidos), Marcos (PM)
- **Decisões do inventário cobertas:** D6, D7, D8

## Contexto

O endpoint do cliente pode estar fora do ar ou lento. Larissa perguntou: "Se o cliente tá offline, o que a gente faz?" ([09:14]). Já houve "cliente nosso com indisponibilidade de duas horas em manutenção planejada" ([09:16] Diego).

Uma chamada que não responde em 10s conta como falha e vai para retry ([09:42] Diego; RNF6). Os eventos que esgotam as tentativas precisam ficar guardados para depuração e reprocessamento (RF9, RF7).

## Decisão

- **Até 5 tentativas com backoff exponencial de 1m, 5m, 30m, 2h e 12h.** Depois disso o evento é considerado **falha permanente**.
- O evento que falha de vez é movido para uma **tabela separada `webhook_dead_letter`**, com o payload, o motivo da falha e o timestamp.
- O reprocessamento é **manual**, por um endpoint admin `POST /admin/webhooks/dead-letter/:id/replay`, que **recoloca o evento na outbox como pendente**. A autorização desse endpoint está no [ADR-007](ADR-007-autorizacao-dos-endpoints-de-webhook.md).

Origens da decisão:

- D6: [09:15] Diego (backoff exponencial, teto de tentativas, DLQ); [09:17] Diego (progressão 1m/5m/30m/2h/12h); [09:17] Larissa ("Decidido: 5 tentativas, backoff 1m/5m/30m/2h/12h").
- D7: [09:18] Diego ("tabela webhook_dead_letter separada, com a payload, motivo da falha e timestamp"); [09:18] Bruno ("Faz sentido"); entra no resumo de [09:48] Larissa ("DLQ persistida em tabela separada").
- D8: [09:18] Diego ("Manual via endpoint admin […] Recoloca na outbox como pendente"); [09:19] Larissa ("Endpoint admin pra replay manual de DLQ").

## Alternativas Consideradas

1. **Retry indefinido com backoff.** Descartada. "Traz o problema de evento ficar pendurado pra sempre se o cliente sumiu" ([09:15] Diego).
2. **3 tentativas, mais agressivo.** Bruno propôs ([09:16]) e a ideia foi descartada. "3 é pouco. Se o cliente teve indisponibilidade de manhã, a gente retentaria três vezes em 30 minutos e mataria" ([09:16] Diego).
3. **Marcar como `failed` na própria outbox, sem tabela de DLQ.** Larissa levantou a opção ([09:17]) e ela foi descartada. A tabela separada deixa "mais limpa a leitura da outbox principal, e fica como evidence pra debug e reprocessamento" ([09:18] Diego).

## Consequências

### Positivas

- **Cobre indisponibilidades longas.** A janela total é de "quase 15 horas entre primeira falha e última tentativa" ([09:17] Diego), muito acima da manutenção de 2h citada. Marcos aceitou: "Se um cliente meu cair por 15 horas, ele já tá com problema sério dele" ([09:17]).
- **A outbox fica enxuta para o polling** ([ADR-002](ADR-002-worker-separado-em-polling.md)). Os eventos mortos saem dela.
- **Os eventos que falharam ficam guardados** com o motivo, o que serve de evidência para depuração e permite reprocessar.
- **O replay manual dá controle operacional** sem reenviar nada automaticamente para um cliente que pode continuar quebrado.

### Negativas

- **Depois de ~15h, a entrega só volta com intervenção humana** de um ADMIN. Não há reprocessamento automático.
- **O cliente não é avisado de que o webhook está falhando.** O e-mail de alerta ficou **fora do escopo desta fase** ([09:37] Larissa). O cliente só percebe pelo histórico de entregas (RF6).
- **Mover da outbox para a DLQ, e da DLQ de volta no replay, mexe em duas tabelas.** Para não perder nem duplicar o evento, isso precisa ser atômico. Inferência, sem detalhe na reunião.
- **Atrasos grandes nas últimas tentativas** (2h, 12h) entregam eventos bem defasados em relação ao estado atual do pedido.

### Trade-off explícito

Ganha resiliência a indisponibilidades de até ~15h e uma outbox limpa. Paga com intervenção manual depois disso e com uma segunda tabela a manter consistente.

### Pontos em aberto (não decididos na reunião)

- **A1, contagem de tentativas:** foram fechadas "5 tentativas" e também **5 intervalos** de backoff (1m/5m/30m/2h/12h ≈ 14h36). Se as 5 incluem o envio inicial, sobra um intervalo. Se são 5 *retries*, o total de envios é 6. Isso precisa ser confirmado com Larissa e Diego antes do FDD.
- **Identidade do evento no replay:** não se definiu se o replay mantém o mesmo `X-Event-Id` ou gera um novo. Isso interfere na deduplicação do lado do cliente ([ADR-005](ADR-005-at-least-once-com-x-event-id.md)).

## Referências

- Inventário: D6, D7, D8; RF7, RF9; RNF6; A1; seção "f" (e-mail de alerta adiado).
