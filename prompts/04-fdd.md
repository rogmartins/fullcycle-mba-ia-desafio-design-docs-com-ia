# Prompt — Geração do FDD (Etapa 4 do roteiro)

> Usado na Etapa 4 do roteiro (`descricao/roteiro.md`). Depende do RFC (`entregavel/docs/RFC.md`), dos 7 ADRs (`entregavel/docs/adrs/`) e do inventário (`prompts/01-exploracao-inicial.resultado.md`). Anexe RFC, ADRs, inventário, transcrição e código ao rodar este prompt.

```
Você vai atuar como engenheiro de software sênior escrevendo o FDD
(Feature Design Document) da feature "Sistema de Webhooks de
Notificação de Pedidos". Você JÁ TEM em mãos:
- o inventário validado (prompts/01-exploracao-inicial.resultado.md);
- os 7 ADRs aceitos (entregavel/docs/adrs/ADR-001 a ADR-007);
- o RFC (entregavel/docs/RFC.md), que já aponta 5 questões em aberto
  (seção 5) e uma inconsistência entre fontes (seção 7: estado
  "falhou" na outbox vs. tabela webhook_dead_letter separada).

Use esses três como fonte primária, mais o código em src/ e
prisma/schema.prisma para tudo que for integração real. Este é o
documento MAIS TÉCNICO do pacote: precisa estar acionável o
suficiente para um desenvolvedor pegar e começar a codar.

REGRA CENTRAL SOBRE AS QUESTÕES EM ABERTO DO RFC
O FDD, sendo o documento de implementação, é o lugar onde essas
questões precisam ganhar uma resposta concreta (porque não dá para
codar sem decidir). Mas você NÃO TEM autoridade da reunião para
fechá-las como se fossem decisão do grupo. Para cada uma das 5
questões da seção 5 do RFC, faça o seguinte:
1. Proponha uma resposta técnica concreta e justificada (você PRECISA
   decidir algo para poder especificar o fluxo/contrato).
2. Marque explicitamente essa resposta como "Premissa de
   implementação (não validada na reunião)" — nunca apresente como se
   fosse decisão fechada em [ADR-XXX] ou na transcrição.
3. Liste essas premissas também no final do documento, numa seção
   "Premissas assumidas pelo FDD", para ficarem fáceis de auditar e
   validar depois com a equipe.
Isso vale também para a inconsistência da seção 7 do RFC (estado
"falhou" da outbox vs. DLQ): escolha e documente uma interpretação
(ex.: "falhou" é um estado transitório entre tentativas, e só depois
da 5ª falha o evento migra para webhook_dead_letter), mas marque como
premissa, não como fato extraído da reunião.

ESTRUTURA OBRIGATÓRIA DE docs/FDD.md

1. **Contexto e motivação técnica**: recapitule em 1 parágrafo (não
   repita o RFC inteiro) o estado atual do código e por que a
   integração acontece em `changeStatus`.

2. **Objetivos técnicos**: metas técnicas mensuráveis ou verificáveis
   (ex.: latência de entrega, atomicidade, nenhuma infraestrutura
   nova), derivadas das RNFs do inventário.

3. **Escopo e exclusões**: reafirme brevemente o que está fora de
   escopo (seção "f" do inventário / seção 5 do RFC), sem repetir a
   justificativa inteira.

4. **Fluxos detalhados** (passo a passo, pode usar numeração ou
   pseudo-sequência):
   a. Criação do evento na outbox (dentro de `changeStatus`).
   b. Processamento pelo worker (ciclo de polling, leitura do batch,
      envio HTTP, marcação de status).
   c. Fluxo de retry (cálculo do próximo intervalo, atualização do
      estado do evento).
   d. Fluxo de DLQ (critério de entrada, estrutura da linha, e o
      fluxo de replay manual: entrada admin → validação de role →
      volta à outbox).
   Cada fluxo deve citar o(s) ADR(s) relevante(s) e, quando aplicável,
   o arquivo de código que seria tocado ou criado.

5. **Contratos públicos**: pelo menos 4 endpoints HTTP, cobrindo no
   mínimo o CRUD de webhooks e o replay da DLQ (ex.: POST /webhooks,
   GET /webhooks, PATCH /webhooks/:id, DELETE /webhooks/:id, GET
   /webhooks/:id/deliveries, POST /webhooks/:id/secret/rotate, POST
   /admin/webhooks/dead-letter/:id/replay). Para cada endpoint:
   método, path, autenticação/autorização exigida (referencie
   ADR-007), payload de exemplo de request E response em JSON, e os
   status codes possíveis (sucesso e erro).

6. **Matriz de erros**: tabela com código (prefixo `WEBHOOK_`),
   HTTP status, quando ocorre, e qual classe de erro do código reusar
   ou criar (referencie ADR-006 e o padrão em
   src/shared/errors/http-errors.ts). Cubra pelo menos: webhook não
   encontrado, URL inválida (não-https), payload acima do limite,
   secret inválida/ausente, erro de validação, conflito/estado
   inválido para replay.

7. **Estratégias de resiliência**: timeouts (10s HTTP, conforme
   RNF6), retries e backoff (ADR-003, com a premissa resolvida da
   questão 1 do RFC), fallback (DLQ + replay manual), e comportamento
   esperado se o worker cair no meio de um batch (ligado à premissa
   sobre o estado "processando"/"falhou").

8. **Observabilidade**: métricas sugeridas (ex.: eventos pendentes na
   outbox, taxa de falha por endpoint, tamanho da DLQ, latência de
   entrega), logs estruturados (siga a convenção Pino já usada em
   src/shared/logger — campos, eventos em snake_case) e tracing
   (como o X-Event-Id e o X-Request-Id podem ser correlacionados
   entre API e worker).

9. **Dependências e compatibilidade**: bibliotecas que entram
   (ex. nenhuma nova para HMAC/HTTP, já que Node 20+ tem node:crypto e
   fetch nativos — isso é inferência do inventário, trate como
   premissa), e compatibilidade com o schema Prisma atual (novas
   tabelas, migrations necessárias).

10. **Critérios de aceite técnicos**: lista verificável (ex.: "evento
    só é removido da outbox após confirmação de entrega", "replay
    exige role ADMIN", "nenhum teste existente em tests/orders.test.ts
    quebra").

11. **Riscos e mitigação**: riscos técnicos específicos de
    implementação (não repita os riscos de negócio/prazo do RFC).

12. **SEÇÃO OBRIGATÓRIA — Integração com o sistema existente**: nomeie
    PELO MENOS 4 caminhos de arquivo REAIS do código base (confirme
    que existem — não invente) e descreva como o módulo de webhooks
    se integra com cada um. Use como base os cruzamentos já mapeados
    no inventário (seção 3) e nos ADRs, por exemplo:
    - src/modules/orders/order.service.ts (onde `changeStatus` chama
      `publishWebhookEvent(tx, ...)`)
    - src/shared/errors/http-errors.ts (subclasses novas com prefixo
      WEBHOOK_)
    - src/shared/logger/index.ts (extensão da lista de redact para
      cobrir secret/X-Signature)
    - src/middlewares/auth.middleware.ts (reuso de requireRole no
      endpoint de replay)
    - src/routes/index.ts e src/app.ts (registro do novo módulo)
    - tests/setup.ts (limpeza das tabelas novas)

13. **Premissas assumidas pelo FDD**: lista consolidada de todas as
    premissas de implementação que você teve que assumir para fechar
    as questões em aberto do RFC e a inconsistência da seção 7,
    deixando claro que precisam ser validadas com a equipe antes da
    implementação real.

REGRAS DE RASTREABILIDADE
- Toda afirmação sobre decisão já fechada deve remeter a um ADR ou a
  um ID do inventário (D/RF/RNF) com timestamp+falante.
- Toda escolha nova feita aqui (que não estava em nenhum ADR, no RFC
  ou na transcrição) deve ser marcada como premissa, conforme a regra
  central acima — nunca apresentada como se já fosse consenso da
  reunião.
- Todo caminho de arquivo citado precisa existir de fato no código em
  src/, prisma/ ou tests/.

SAÍDA
Gere o conteúdo completo de docs/FDD.md pronto para salvar no
repositório.
```
