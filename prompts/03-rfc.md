# Prompt — Geração do RFC (Etapa 3 do roteiro)

> Usado na Etapa 3 do roteiro (`descricao/roteiro.md`). Depende dos 7 ADRs já escritos em `entregavel/docs/adrs/` e do inventário em `prompts/01-exploracao-inicial.resultado.md`. Anexe os ADRs, o inventário, a transcrição e o código ao rodar este prompt.

```
Você vai atuar como engenheiro de software sênior escrevendo o RFC
(Request for Comments) da feature "Sistema de Webhooks de Notificação
de Pedidos". Você JÁ TEM em mãos:
- o inventário validado (prompts/01-exploracao-inicial.resultado.md);
- os 7 ADRs já aceitos em entregavel/docs/adrs/ (ADR-001 a ADR-007).

Use esses dois como fonte primária. Não invente nenhuma decisão,
alternativa, risco ou questão em aberto que não esteja neles ou
rastreável a TRANSCRICAO.md / ao código em src/.

OBJETIVO
Produzir o arquivo docs/RFC.md: um documento CONCISO (2 a 4 páginas),
em nível de ARQUITETURA, que apresenta a proposta técnica para revisão
da equipe. Ele é diferente dos ADRs (que registram cada decisão
pontual já fechada) e do FDD (que detalha a implementação). O RFC
responde "o que propomos e por quê, e o que ainda está em aberto" —
NÃO repita aqui o nível de detalhe de contratos de API, payloads ou
matriz de erros; isso é escopo do FDD, que vem depois.

ESTRUTURA OBRIGATÓRIA DE docs/RFC.md
1. **Metadados**: título, autor (você pode usar "Equipe de Engenharia
   — Plataforma de Pedidos" ou algo equivalente, já que o documento é
   retrospectivo a uma decisão de equipe), status (ex.: "Aceito" ou
   "Em revisão", justifique a escolha), data, e revisores — use os 5
   participantes da reunião como revisores: Larissa (Tech Lead),
   Marcos (PM), Bruno (Eng. Pedidos), Diego (Eng. Plataforma), Sofia
   (Eng. Segurança).
2. **Resumo executivo (TL;DR)**: 3-5 frases com a proposta e o
   principal motivador de negócio (RF1, RNF11 do inventário — prazo e
   risco de churn da Atlas).
3. **Contexto e problema**: por que a feature existe, citando o
   problema relatado na reunião (polling caro em GET /orders,
   [09:00] Marcos) e o estado atual do código (ausência de mecanismo
   de notificação, máquina de estados e transação de changeStatus).
4. **Proposta técnica**: visão geral da solução (outbox + worker +
   retry/DLQ + HMAC + at-least-once + reuso de padrões), SEM descer a
   contratos de endpoint ou payload completo. Cada peça da proposta
   deve linkar para o ADR correspondente (ex.: "usamos o padrão outbox
   no MySQL ([ADR-001](adrs/ADR-001-outbox-no-mysql.md))").
5. **Alternativas consideradas**: selecione pelo menos 2 alternativas
   REAIS que foram discutidas e descartadas na reunião, em nível
   arquitetural (não escolha detalhes de implementação). Boas
   candidatas do inventário: disparo síncrono vs. outbox, Redis
   Streams vs. outbox no MySQL, exactly-once vs. at-least-once,
   retry indefinido/3 tentativas vs. 5 tentativas com backoff. Para
   cada uma, traga o trade-off que motivou o descarte, com citação
   (timestamp + falante).
6. **Questões em aberto**: selecione pelo menos 2 pontos levantados
   na reunião e NÃO decididos ou adiados. Use as ambiguidades A1-A8 e
   os "Pontos em aberto" já sinalizados nos ADRs (ex.: A1 — contagem
   de tentativas, 5 ou 6 envios; A2 — X-Timestamp fora do HMAC; A5 —
   assinatura durante o grace period da rotação; identidade do
   X-Event-Id no replay). Para cada questão, diga por que importa
   resolver antes ou durante o FDD.
7. **Impacto e riscos**: impacto em disponibilidade do changeStatus
   (RNF2, RNF3), impacto operacional (novo processo para rodar/
   monitorar — o worker), riscos de segurança levantados por Sofia
   (RNF10, revisão de 2 dias úteis), e o risco de prazo (RNF11, fim de
   novembro / 3 sprints).
8. **Decisões relacionadas**: lista com link para os 7 ADRs.

REGRAS
- Todo fato, decisão ou risco citado deve remeter a um ADR, a um ID
  do inventário (D.../RF.../RNF.../A...), ou a um timestamp+falante da
  transcrição. Não generalize sem origem.
- Não repita o conteúdo completo dos ADRs — resuma e linke. O RFC é
  mais curto que a soma dos ADRs.
- Não avance para contratos HTTP detalhados, matriz de erros ou
  fluxos passo-a-passo — isso é FDD.
- Se notar contradição entre dois ADRs ou entre um ADR e o inventário,
  pare e aponte a contradição em vez de silenciosamente escolher um
  dos dois.

SAÍDA
Gere o conteúdo completo de docs/RFC.md pronto para salvar no
repositório.
```
