# RFC — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| **Título** | Sistema de Webhooks de Notificação de Pedidos |
| **Autor** | Equipe de Engenharia, Plataforma de Pedidos. Documento retrospectivo da reunião técnica de quinta-feira, 09:00 |
| **Status** | **Em revisão** |
| **Data** | 2026-10-03 (redação). A transcrição não registra a data de calendário da reunião |
| **Revisores** | Larissa (Tech Lead), Marcos (PM), Bruno (Eng. Pedidos), Diego (Eng. Plataforma), Sofia (Eng. Segurança) |

**Por que "Em revisão":** as decisões arquiteturais já estão fechadas e registradas como *Aceito* nos ADRs 001–007. Ainda assim:

- Larissa combinou uma sessão de revisão do design com Bruno e Diego "antes da gente começar a codar" ([09:50] Larissa).
- A proposta tem questões em aberto (seção 6) que precisam ser fechadas antes ou durante o FDD.
- A revisão de segurança da Sofia é pré-requisito de deploy (RNF10).

---

## 1. Resumo executivo (TL;DR)

Propomos notificar clientes B2B por **webhooks outbound** sempre que o status de um pedido muda (RF1). A Atlas Comercial sinalizou que pode migrar para o concorrente se a entrega não sair até o fim do trimestre e pediu o prazo de **fim de novembro** (RNF11; [09:00] e [09:45] Marcos).

A arquitetura proposta:

- O evento é gravado numa **outbox no MySQL**, na mesma transação da mudança de status.
- Um **worker separado** lê a outbox por polling de 2s.
- Falhas de entrega passam por **retry com backoff** e terminam numa **DLQ**.
- Cada requisição é assinada com **HMAC-SHA256** usando uma secret por endpoint.
- A entrega é **at-least-once**, e o cliente deduplica por **`X-Event-Id`**.

Não entra infraestrutura nova, e a feature reaproveita os padrões do código existente. A estimativa é de **3 sprints**, incluindo a revisão de segurança (D23).

## 2. Contexto e problema

Hoje os clientes Atlas Comercial, MaxDistribuição e Nova Cargo "ficam batendo no GET /orders de tempos em tempos", o que deixa a integração "lenta e cara pra eles" ([09:00] Marcos). Para eles, "qualquer coisa abaixo de 10 segundos já é tempo real" ([09:02] Marcos; RNF1). O fluxo é só de saída: "Eles querem receber, não mandar" ([09:02] Marcos; D1).

Estado atual do código:

- **Não existe mecanismo de notificação externa, evento, fila ou webhook.**
- O ciclo de vida do pedido é controlado por uma máquina de estados em `src/modules/orders/order.status.ts`: `PENDING → PAID → PROCESSING → SHIPPED → DELIVERED`, com `CANCELLED` como saída a partir de PENDING, PAID e PROCESSING.
- Toda mudança de status passa por `OrderService.changeStatus` (`src/modules/orders/order.service.ts:126-179`), numa transação Prisma interativa que valida a transição, debita ou repõe estoque, atualiza `orders` e grava `order_status_history`. Bruno resumiu: "a transação de mudança de status hoje já é pesada" ([09:04] Bruno).
- O ponto de integração natural é essa transação, e qualquer solução tem que preservá-la: atomicidade (RNF2) e isolamento de falhas de clientes externos (RNF3).

## 3. Proposta técnica

A solução tem cinco peças e um princípio transversal.

**3.1 Captura transacional do evento: Outbox no MySQL** ([ADR-001](adrs/ADR-001-outbox-no-mysql.md))

- Dentro da transação de `changeStatus`, uma função `publishWebhookEvent(tx, …)` insere na tabela `webhook_outbox` um evento com **payload já renderizado (snapshot)** e PK UUID.
- O insert só acontece se algum webhook ativo do customer assina o status de destino (D16).
- Se o insert falhar, a mudança de status sofre rollback (D19). Com isso, um status que mudou sempre tem evento registrado, e um rollback descarta o evento junto.

**3.2 Despacho assíncrono: worker separado com polling** ([ADR-002](adrs/ADR-002-worker-separado-em-polling.md))

- Um processo Node próprio (`src/worker.ts`, `npm run worker`), com o mesmo banco e um `PrismaClient` próprio.
- A cada 2s, lê em batch os eventos pendentes mais antigos e faz as chamadas HTTP.
- Roda como **instância única**, o que garante ordenação **por `order_id`**, mas não global (D5).

**3.3 Resiliência: retry com backoff e DLQ** ([ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md))

- Falhas, incluindo o timeout de 10s (RNF6), são retentadas com backoff de 1m/5m/30m/2h/12h, num total de 5 tentativas.
- Esgotadas as tentativas, o evento vai para a tabela `webhook_dead_letter`.
- O reprocessamento é manual, por um endpoint admin que devolve o evento à outbox.

**3.4 Autenticidade e integridade: HMAC-SHA256** ([ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md))

- O corpo de cada requisição é assinado com HMAC-SHA256 usando uma **secret única por endpoint**, gerada pela plataforma.
- A secret pode ser rotacionada, e a antiga continua válida por 24h depois da rotação.
- Só URLs `https` são aceitas (RNF4).

**3.5 Semântica de entrega: at-least-once** ([ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md))

- O cliente pode receber o mesmo evento mais de uma vez e deduplica pelo `X-Event-Id`, um UUID gerado quando o evento entra na outbox.

**3.6 Princípio transversal: reuso dos padrões existentes** ([ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md))

- Novo módulo `src/modules/webhooks/` (controller, service, repository, routes, schemas).
- Reaproveita a hierarquia `AppError` com códigos `WEBHOOK_*`, o logger Pino, o error middleware centralizado e a validação Zod.

**3.7 Controle de acesso** ([ADR-007](adrs/ADR-007-autorizacao-dos-endpoints-de-webhook.md))

- O CRUD de configuração fica aberto a qualquer usuário autenticado, por enquanto. O `customer_id` vai no body ou no path, e não vem do JWT.
- O replay da DLQ exige ADMIN via o `requireRole` existente e registra quem executou.

Contratos de endpoint, payload, headers completos e matriz de erros ficam para o FDD.

## 4. Alternativas consideradas

| Alternativa descartada | Trade-off que motivou o descarte | Origem |
|---|---|---|
| **Disparo HTTP síncrono** dentro de `changeStatus` | Um cliente lento travaria a mudança de status de outros pedidos, e não há rollback aceitável se o cliente cair. A outbox troca latência imediata por isolamento e atomicidade. | [09:04] Bruno; [09:06] Diego ("Síncrono está fora de questão") |
| **Redis Streams / Redis Cluster** em vez da outbox no MySQL | Exigiria subir e operar infraestrutura nova num time pequeno ("overengineering"). A outbox no MySQL fica com menos vazão potencial, mas não adiciona nada novo para operar. | [09:07] Larissa; [09:07] Diego |
| **Exactly-once** em vez de at-least-once | Exigiria coordenação dos dois lados e ficaria "muito mais complexo". Escolher at-least-once simplifica a plataforma e passa a deduplicação para o cliente. | [09:25] Diego; [09:25] Sofia ("Isso joga responsabilidade pro cliente") |
| **Retry indefinido** ou **só 3 tentativas** | Retry indefinido deixaria eventos "pendurados pra sempre". Três tentativas cobririam só ~30 min, contra manutenções reais de 2h. As 5 tentativas (~15h) equilibram as duas coisas. | [09:15] Diego; [09:16] Bruno / Diego |
| **Trigger de banco** para acordar o worker | O MySQL não tem `NOTIFY/LISTEN`, e um trigger não notifica processo externo. O polling de 2s atende o requisito de <10s com uma solução simples. | [09:09] Bruno / Diego |

## 5. Questões em aberto

Estes pontos foram levantados ou ficaram implícitos na reunião, mas **não foram decididos**. Precisam de dono e resolução antes ou durante o FDD.

1. **Contagem de tentativas: 5 ou 6 envios?** (A1; [ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md)) O grupo fechou "5 tentativas" e também 5 intervalos de backoff ([09:17] Larissa). *Por que importa:* define a máquina de estados da outbox, o momento de ir para a DLQ e a janela total comunicada aos clientes (~14h36 ou menos).
2. **`X-Timestamp` fora do HMAC** (A2; [ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md)). A assinatura cobre só o corpo ([09:22] Sofia), mas o `X-Timestamp` foi justificado como defesa contra replay ([09:44] Diego). *Por que importa:* sem assinar o timestamp, essa defesa não se sustenta. É uma decisão da revisão de segurança (RNF10) e afeta o contrato documentado para os clientes no portal.
3. **Assinatura durante o grace period da rotação** (A5; [ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md)). Durante as 24h, não se definiu se o envio assina com as duas secrets ou só com a nova. *Por que importa:* muda o formato do `X-Signature` (A3) e o modelo de dados das secrets.
4. **Identidade do `X-Event-Id` no replay da DLQ** ([ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md), [ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md)). *Por que importa:* lendo ao pé da letra, o replay "recoloca na outbox" ([09:18] Diego) e o id é gerado "quando o evento entra na outbox" ([09:25] Diego). Juntas, as duas falas implicam um id **novo** no replay, o que quebraria a deduplicação do lado do cliente. Ninguém decidiu isso explicitamente.
5. **A criação do pedido gera evento?** (A6; [ADR-001](adrs/ADR-001-outbox-no-mysql.md)) `OrderService.create` grava `PENDING` sem passar por `changeStatus`. *Por que importa:* pode exigir um segundo ponto de integração e muda o filtro de eventos disponível para os clientes.

Ficam explicitamente **adiados** e fora desta proposta:

- e-mail de alerta ([09:37] Larissa);
- rate limiting de saída ("observar e decidir depois", [09:39] Larissa);
- dashboard visual ([09:40] Larissa);
- arquivamento da outbox ([09:08] Diego);
- vários workers ([09:13] Diego);
- endurecimento da autorização do CRUD ([09:37] Sofia).

## 6. Impacto e riscos

- **Disponibilidade do `changeStatus` (RNF2, RNF3).**
  - A outbox elimina I/O externo da transação, mas acrescenta uma leitura (o filtro D16) e um insert. Pelo desenho de D19, uma falha nesse insert **bloqueia** a mudança de status ([ADR-001](adrs/ADR-001-outbox-no-mysql.md)).
  - A transação interativa usa o timeout default do Prisma, porque não há um configurado em `order.service.ts`.
- **Operação.**
  - O worker é um **novo processo** para implantar e monitorar ([ADR-002](adrs/ADR-002-worker-separado-em-polling.md)). Se ele cair, as entregas param, mas os eventos não se perdem.
  - Depois de ~15h de falhas, o evento só volta com replay manual de um ADMIN ([ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md)).
  - A tabela da outbox cresce sem arquivamento nesta fase.
- **Segurança (RNF10).**
  - Sofia exigiu **pelo menos 2 dias úteis** de revisão de HMAC e geração de secret antes do deploy ([09:46] e [09:49] Sofia).
  - A secret precisa ser guardada de forma recuperável, e a lista de `redact` do logger (`src/shared/logger/index.ts`) ainda não a cobre ([ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md)).
  - Sem vínculo `User → Customer` no código, qualquer usuário autenticado pode gerenciar webhooks de qualquer customer, risco aceito nesta fase ([ADR-007](adrs/ADR-007-autorizacao-dos-endpoints-de-webhook.md)).
- **Prazo (RNF11, D23).**
  - Estimativa de **3 sprints** com a revisão da Sofia no fim ([09:46–09:47] Larissa), contra a data de fim de novembro da Atlas.
  - As questões em aberto da seção 5, em especial a A2, podem gerar retrabalho se forem fechadas tarde.

## 7. Notas de consistência entre as fontes

- **Estado "falhou" na outbox e a DLQ separada.** Diego listou os estados da outbox como "pendente, processando, falhou, entregue" ([09:08] Diego). Depois, a falha permanente foi para uma tabela separada `webhook_dead_letter` ([09:18] Diego; D7). Os ADRs não definem se "falhou" na outbox significa *falha transitória aguardando retry* ou se ficou obsoleto com a DLQ. Isso **não está resolvido** aqui e deve ser fechado junto com a questão 1 no FDD.
- Os ADRs 001–007 e o inventário não se contradizem em nenhum outro ponto.

## 8. Decisões relacionadas

- [ADR-001 — Padrão Outbox no MySQL](adrs/ADR-001-outbox-no-mysql.md)
- [ADR-002 — Worker em processo separado com polling de 2s](adrs/ADR-002-worker-separado-em-polling.md)
- [ADR-003 — Retry com backoff exponencial, DLQ e replay manual](adrs/ADR-003-retry-com-backoff-e-dlq.md)
- [ADR-004 — HMAC-SHA256 com secret por endpoint](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md)
- [ADR-005 — At-least-once com X-Event-Id](adrs/ADR-005-at-least-once-com-x-event-id.md)
- [ADR-006 — Reuso dos padrões existentes do projeto](adrs/ADR-006-reuso-dos-padroes-existentes.md)
- [ADR-007 — Autorização dos endpoints de webhook](adrs/ADR-007-autorizacao-dos-endpoints-de-webhook.md)
