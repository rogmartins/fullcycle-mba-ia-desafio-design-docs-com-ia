# FDD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| **Feature** | Sistema de Webhooks de Notificação de Pedidos |
| **Status** | Rascunho para implementação. Depende da validação das premissas da [seção 13](#13-premissas-assumidas-pelo-fdd) |
| **Documentos-base** | [RFC](RFC.md) · [ADR-001](adrs/ADR-001-outbox-no-mysql.md) · [ADR-002](adrs/ADR-002-worker-separado-em-polling.md) · [ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md) · [ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md) · [ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md) · [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md) · [ADR-007](adrs/ADR-007-autorizacao-dos-endpoints-de-webhook.md) |
| **Data** | 2026-10-03 |

> **Como ler este documento.** Há três tipos de afirmação, cada um com uma marcação:
>
> - **Decisão fechada na reunião:** traz o ADR ou o ID do inventário (D/RF/RNF) e o timestamp com o nome de quem falou.
> - **Escolha nova deste FDD:** aparece como **🔶 Premissa P-NN (não validada na reunião)**. Todas estão consolidadas na [seção 13](#13-premissas-assumidas-pelo-fdd) e precisam ser validadas com a equipe antes de codar.
> - **Inferência a partir do código:** traz o caminho real do arquivo.

---

## 1. Contexto e motivação técnica

Hoje o OMS não tem nenhum mecanismo de notificação externa. O status do pedido segue a máquina de estados de `src/modules/orders/order.status.ts`: `PENDING → PAID → PROCESSING → SHIPPED → DELIVERED`, com `CANCELLED` como saída a partir de PENDING, PAID e PROCESSING. O único caminho que muda um status existente é `OrderService.changeStatus` (`src/modules/orders/order.service.ts:126-179`). Ele roda numa transação Prisma interativa que valida a transição, debita ou repõe estoque, atualiza `orders` e grava `order_status_history`.

Por isso a integração acontece **dentro dessa transação**. É o único ponto que garante que "se a transação principal commitou, o evento foi registrado, e se ela deu rollback, o evento some junto" ([09:06] Diego; [ADR-001](adrs/ADR-001-outbox-no-mysql.md)). Ao mesmo tempo, nenhuma chamada HTTP pode acontecer ali (RNF3, [09:04] Bruno). O envio fica com um worker separado ([ADR-002](adrs/ADR-002-worker-separado-em-polling.md)).

## 2. Objetivos técnicos

| ID | Objetivo verificável | Origem |
|---|---|---|
| OBJ-01 | Em operação normal (endpoint saudável, outbox sem backlog), o 1º envio acontece em **< 10s** depois do commit do `changeStatus`. O alvo é ≤ 2s de polling mais o tempo de processamento. | RNF1, [09:02] Marcos; D3, [09:10] Larissa |
| OBJ-02 | **Atomicidade:** se o insert na outbox falhar, a mudança de status sofre rollback. Nunca existe status alterado sem evento quando há webhook assinante. | RNF2, D19, [09:40] Bruno |
| OBJ-03 | `changeStatus` **não faz I/O de rede**. Acrescenta apenas uma leitura e um insert no mesmo banco. | RNF3, [ADR-001](adrs/ADR-001-outbox-no-mysql.md) |
| OBJ-04 | **Nenhuma infraestrutura nova.** Usa o mesmo MySQL e a mesma stack. 🔶 Também nenhuma dependência npm nova (P-15). | RNF9, D13, [09:29] Bruno |
| OBJ-05 | Timeout de **10s** por chamada HTTP. Estourou, conta como falha e vai para retry. | RNF6, [09:42] Diego |
| OBJ-06 | Corpo enviado **≤ 64KB**. Acima disso, gera erro e não trunca. | RNF5, [09:24] Larissa |
| OBJ-07 | Só URLs `https`. URL `http` é recusada com erro de validação. | RNF4, [09:23] Sofia |
| OBJ-08 | Retry com backoff 1m/5m/30m/2h/12h. Esgotadas as tentativas, o evento vai para `webhook_dead_letter`. | D6, D7, [ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md) |
| OBJ-09 | Toda requisição leva `X-Signature` (HMAC-SHA256 do corpo) e `X-Event-Id`. | D9, D10, [ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md), [ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md) |
| OBJ-10 | Os testes existentes em `tests/orders.test.ts` e `tests/auth.test.ts` continuam passando sem alteração. | D13 (reuso) |

## 3. Escopo e exclusões

**No escopo:**

- outbox transacional;
- worker de polling;
- retry, DLQ e replay manual;
- assinatura HMAC com rotação de secret;
- CRUD de configuração de webhooks;
- histórico de entregas.

**Fora de escopo nesta fase** (justificativas no [RFC §5](RFC.md#5-questões-em-aberto)):

- e-mail de alerta ([09:37] Larissa);
- rate limiting de saída ([09:39] Larissa);
- dashboard visual ([09:40] Larissa);
- arquivamento da outbox ([09:08] Diego);
- vários workers ([09:13] Diego);
- endurecimento da autorização do CRUD ([09:37] Sofia);
- webhooks inbound (D1, [09:02] Marcos).

🔶 **P-05:** a **criação** do pedido não gera evento nesta fase (ver [seção 13](#13-premissas-assumidas-pelo-fdd)).

---

## 4. Fluxos detalhados

### 4.0 Modelo de dados (base para os fluxos)

São quatro tabelas novas em `prisma/schema.prisma`. Todas seguem a convenção do projeto: PK `String @id @default(uuid()) @db.Char(36)` (D21, [09:51] Larissa) e `@@map` em snake_case. Os nomes `webhook_outbox` e `webhook_dead_letter` vêm da reunião ([09:06] e [09:18] Diego). 🔶 `webhook_endpoints` e `webhook_deliveries` são nomes propostos aqui (P-10).

```prisma
enum WebhookOutboxStatus {
  PENDING      // aguardando 1º envio
  PROCESSING   // reivindicado pelo worker no ciclo atual
  FAILED       // 🔶 P-06: falha transitória, aguardando próxima tentativa
  DELIVERED    // 2xx recebido
}

model WebhookEndpoint {
  id                       String    @id @default(uuid()) @db.Char(36)
  customerId               String    @db.Char(36)
  url                      String    @db.VarChar(2048)
  events                   Json      // array de OrderStatus assinados (D16, RF4)
  active                   Boolean   @default(true)
  secretCiphertext         String    @db.VarChar(512)   // 🔶 P-11 (cifrada)
  previousSecretCiphertext String?   @db.VarChar(512)   // rotação (D9)
  previousSecretExpiresAt  DateTime?                    // now + 24h na rotação
  secretRotatedAt          DateTime?
  createdById              String    @db.Char(36)
  deletedAt                DateTime?                    // 🔶 P-12 (soft delete)
  createdAt                DateTime  @default(now())
  updatedAt                DateTime  @updatedAt

  customer  Customer @relation(fields: [customerId], references: [id])
  createdBy User     @relation("WebhookCreatedBy", fields: [createdById], references: [id])

  @@index([customerId, active])
  @@map("webhook_endpoints")
}

model WebhookOutbox {
  id            String              @id @default(uuid()) @db.Char(36)
  eventId       String              @db.Char(36)        // X-Event-Id (D10) — 🔶 P-09
  webhookId     String              @db.Char(36)
  orderId       String              @db.Char(36)        // sem FK — 🔶 P-13
  eventType     String              @db.VarChar(64)     // "order.status_changed"
  payload       Json                                    // snapshot (D22)
  status        WebhookOutboxStatus @default(PENDING)
  attempts      Int                 @default(0)         // envios já realizados
  nextAttemptAt DateTime            @default(now())
  lastError     String?             @db.VarChar(500)
  lockedAt      DateTime?
  deliveredAt   DateTime?
  createdAt     DateTime            @default(now())

  @@unique([eventId, webhookId])
  @@index([status, nextAttemptAt])
  @@index([createdAt])
  @@index([orderId])
  @@map("webhook_outbox")
}

model WebhookDeadLetter {
  id             String    @id @default(uuid()) @db.Char(36)
  eventId        String    @db.Char(36)
  webhookId      String    @db.Char(36)
  orderId        String    @db.Char(36)
  eventType      String    @db.VarChar(64)
  payload        Json                                   // D7: payload
  failureReason  String    @db.VarChar(500)             // D7: motivo
  lastHttpStatus Int?
  attempts       Int
  failedAt       DateTime  @default(now())              // D7: timestamp
  replayedAt     DateTime?
  replayedById   String?   @db.Char(36)                 // RF8: auditoria
  createdAt      DateTime  @default(now())

  replayedBy User? @relation("DeadLetterReplayedBy", fields: [replayedById], references: [id])

  @@index([webhookId])
  @@index([replayedAt])
  @@map("webhook_dead_letter")
}

model WebhookDelivery {
  id            String   @id @default(uuid()) @db.Char(36)
  webhookId     String   @db.Char(36)
  eventId       String   @db.Char(36)
  attemptNumber Int
  success       Boolean
  httpStatus    Int?
  durationMs    Int
  requestBody   Json                                    // RF6: payload
  responseBody  String?  @db.Text                       // RF6: response (truncada — 🔶 P-10)
  errorMessage  String?  @db.VarChar(500)
  createdAt     DateTime @default(now())

  @@index([webhookId, createdAt])
  @@map("webhook_deliveries")
}
```

Os modelos existentes `Customer` e `User` recebem **apenas campos de relação reversa**, sem coluna nova: `webhooks WebhookEndpoint[]`, `createdWebhooks` e `replayedDeadLetters`. O Prisma exige isso para declarar as relações.

O índice `[status, nextAttemptAt]` complementa os índices em `status` e `created_at` citados em [09:08] Diego, porque a consulta de polling filtra pelos dois campos (🔶 P-06).

### 4.a Criação do evento na outbox (dentro de `changeStatus`)

**ADRs:** [ADR-001](adrs/ADR-001-outbox-no-mysql.md), [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md).

**Arquivos:**

- tocado: `src/modules/orders/order.service.ts`;
- criados: `src/modules/webhooks/webhook.publisher.ts` e `src/modules/webhooks/webhook.payload.ts`.

1. `changeStatus` executa os passos atuais: busca o pedido, valida com `canTransition`, debita ou repõe estoque, `tx.order.update` e `tx.orderStatusHistory.create` (`order.service.ts:131-167`).
2. **Novo passo**, logo depois de `tx.orderStatusHistory.create` (l.167) e antes da releitura (l.169):
   ```ts
   await publishWebhookEvent(tx, order, from, to);
   ```
   `order` é o registro carregado em l.132 e já tem `id`, `orderNumber`, `customerId` e `totalCents`. A assinatura segue D20 ([09:41] Bruno) e o padrão `TxClient` (`order.service.ts:24`).
3. `publishWebhookEvent(tx, order, fromStatus, toStatus)`:
   1. Busca os assinantes: `tx.webhookEndpoint.findMany({ where: { customerId: order.customerId, active: true, deletedAt: null } })`.
   2. Filtra em memória os endpoints cujo `events` contém `toStatus` (D16, [09:34] Bruno: filtro na inserção).
   3. **Se a lista ficar vazia, retorna sem inserir nada** ("nem insere", [09:34] Bruno).
   4. Gera `eventId = uuidv4()` (lib `uuid` já presente) e `occurredAt = new Date()`.
   5. Renderiza o **snapshot** (D22, [09:52] Larissa) com `buildOrderStatusChangedPayload(...)`:
      ```json
      {
        "event_id": "6f1c2a9e-3b7d-4d0a-9a51-2f5a8c0e7b14",
        "event_type": "order.status_changed",
        "timestamp": "2026-10-03T14:22:05.120Z",
        "order_id": "b2d4f8a0-1c3e-4a5b-8d7f-0e9a1b2c3d4e",
        "order_number": "ORD-000123",
        "from_status": "PAID",
        "to_status": "PROCESSING",
        "customer_id": "9a8b7c6d-5e4f-4a3b-2c1d-0e9f8a7b6c5d",
        "total_cents": 15990
      }
      ```
      - Os campos vêm de [09:43] Diego, e `items` fica fora de propósito ("Não manda items pra não inflar").
      - 🔶 **P-14:** os "campos básicos da order" se limitam a `total_cents`, o único citado.
      - 🔶 **P-07:** `timestamp` é o momento da mudança de status (`occurredAt`), não o do envio. O momento do envio vai no header `X-Timestamp`.
   6. `tx.webhookOutbox.createMany({ data: endpoints.map(e => ({ eventId, webhookId: e.id, orderId, eventType, payload, status: 'PENDING', nextAttemptAt: occurredAt })) })`.
      - 🔶 **P-09:** há **uma linha por endpoint assinante (fan-out na inserção)**, todas com o mesmo `eventId`.
4. Qualquer exceção em 3.1–3.6 propaga para fora do callback de `$transaction`, e **toda a mudança de status sofre rollback** (D19, [09:40] Bruno; [09:41] Diego: "Essencial").
5. Depois do commit, `changeStatus` devolve o pedido exatamente como hoje. O contrato de `PATCH /orders/:id/status` não muda.

> **Por que o limite de 64KB não é checado aqui.** Se fosse, um evento grande impediria a mudança de status (consequência negativa do [ADR-001](adrs/ADR-001-outbox-no-mysql.md)). 🔶 **P-08:** a checagem fica no worker (4.b, passo 4.3) e no replay (4.d). O payload tem 9 campos escalares, então na prática não chega perto do teto ([09:24] Diego: "Nenhum evento nosso vai chegar perto disso").

### 4.b Processamento pelo worker

**ADRs:** [ADR-002](adrs/ADR-002-worker-separado-em-polling.md), [ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md), [ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md).

**Arquivos criados:**

- `src/worker.ts` (entry-point, no molde de `src/server.ts`);
- `src/modules/webhooks/webhook.processor.ts`. 🔶 **P-16:** escolhi `processor` entre as duas opções citadas em [09:28] Bruno.
- `src/modules/webhooks/webhook.signer.ts`.

**Bootstrap (`src/worker.ts`):**

1. Importa `prisma` de `src/config/database.ts`. Como é outro processo Node, isso já cria um `PrismaClient` próprio (D14, [09:30] Bruno).
2. Cria o logger com `service: 'order-management-worker'` (ver 8.2 e 12).
3. **Recuperação de órfãos** (🔶 P-17): `UPDATE webhook_outbox SET status='PENDING', locked_at=NULL WHERE status='PROCESSING'`. Como só existe um worker (D5), qualquer linha em `PROCESSING` no boot sobrou de uma execução que caiu.
4. Registra SIGINT/SIGTERM com graceful shutdown: para de agendar ciclos novos, espera o ciclo corrente terminar e chama `prisma.$disconnect()`. É o mesmo padrão de `src/server.ts:13-21`.
5. Loga `webhook_worker_started` e inicia o loop.

**Ciclo de polling** (a cada `WEBHOOK_POLL_INTERVAL_MS = 2000`, D3, [09:09] Diego):

1. 🔶 **P-18:** usa `setTimeout` reagendado **depois do fim** do ciclo, e não `setInterval`, para que dois ciclos nunca se sobreponham.
2. **Reivindica o batch:**
   ```sql
   SELECT id FROM webhook_outbox
    WHERE status IN ('PENDING','FAILED') AND next_attempt_at <= NOW()
    ORDER BY created_at ASC
    LIMIT :WEBHOOK_BATCH_SIZE          -- 🔶 P-19: 10
   ```
   Em seguida: `UPDATE … SET status='PROCESSING', locked_at=NOW() WHERE id IN (…) AND status IN ('PENDING','FAILED')`. Com um único worker não é preciso `FOR UPDATE SKIP LOCKED`. Fica registrado como caminho futuro para vários workers, que o MySQL 8.0 do `docker-compose.yml` suporta.
3. **Agrupa por `orderId`.** 🔶 **P-20:** os grupos são processados **em paralelo** entre si e **em ordem de `created_at` dentro de cada grupo**. Isso mantém a ordenação por `order_id` (D5) e evita que um endpoint lento (até 10s) atrase os pedidos de outros clientes.
4. **Para cada linha:**
   1. Carrega o `WebhookEndpoint`. Se `deletedAt != null` ou `active = false`, **descarta**: apaga a linha da outbox e loga `webhook_event_discarded` com o motivo (🔶 P-12).
   2. Serializa: `body = JSON.stringify(row.payload)`.
      - A assinatura é calculada **sobre esses bytes exatos**.
      - O MySQL normaliza a ordem das chaves em colunas `JSON`, então a ordem pode diferir da usada na inserção. Isso não afeta a verificação, porque o cliente valida os bytes recebidos.
   3. Se `Buffer.byteLength(body) > 65536`, vai **direto para a DLQ** (4.d) com `failureReason = 'PAYLOAD_TOO_LARGE'` e sem retry (RNF5, 🔶 P-08).
   4. **Assina** (`webhook.signer.ts`): `HMAC-SHA256(secret, body)` com `node:crypto` (D9).
      - A secret é decifrada com AES-256-GCM (🔶 P-11).
      - Formato do header (🔶 P-03): `sha256=<hex minúsculo>`.
      - Se `previousSecretExpiresAt > now`, o header leva as duas assinaturas, **a nova primeiro**: `X-Signature: sha256=<hex_nova>,sha256=<hex_antiga>`.
      - Se a secret não puder ser decifrada, vai para a DLQ com `failureReason = 'WEBHOOK_SECRET_REQUIRED'` e loga em nível `error`.
   5. **Envia** com `fetch(url, { method: 'POST', body, headers, redirect: 'manual', signal: AbortSignal.timeout(10_000) })` (RNF6). Headers:

      | Header | Valor | Origem |
      |---|---|---|
      | `Content-Type` | `application/json` | [09:44] Diego |
      | `X-Event-Id` | `row.eventId` | D10, [09:25] Diego |
      | `X-Webhook-Id` | `row.webhookId` | [09:44] Sofia; [09:45] Diego |
      | `X-Timestamp` | ISO 8601 UTC do **envio** | [09:44] Diego. 🔶 Formato: P-07 |
      | `X-Signature` | ver passo 4.4 | D9. 🔶 Formato: P-03 |
      | `User-Agent` | `OMS-Webhooks/1.0` | 🔶 P-21 |

      🔶 **P-22:** `redirect: 'manual'`, ou seja, uma resposta 3xx conta como falha e o worker não segue redirecionamentos. Isso evita trocar silenciosamente de destino, inclusive para `http`.
   6. **Classifica a resposta:**
      - **sucesso** = status 2xx;
      - **falha** = timeout (`TimeoutError`), erro de rede, ou qualquer status fora de 2xx (🔶 P-23: 4xx também é retentado; ver 7.2).
   7. **Grava a tentativa** em `webhook_deliveries`: `attemptNumber = row.attempts + 1`, `success`, `httpStatus`, `durationMs`, `requestBody` e `responseBody` truncado em 2KB (🔶 P-10).
   8. **Em caso de sucesso:** `status='DELIVERED'`, `deliveredAt=now`, `attempts+1`, `lockedAt=null`. Loga `webhook_delivery_succeeded`.
   9. **Em caso de falha:** segue o fluxo 4.c.
5. Uma exceção inesperada no ciclo é capturada no nível do loop, que loga `webhook_worker_tick_failed` (`error`) e agenda o próximo ciclo normalmente. O processo não cai. Isso é necessário porque o error middleware não cobre o worker ([ADR-002](adrs/ADR-002-worker-separado-em-polling.md)).

### 4.c Fluxo de retry

**ADR:** [ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md). **Arquivo:** `src/modules/webhooks/webhook.processor.ts` (constante `RETRY_BACKOFF_MS`).

🔶 **P-01 (questão 1 do RFC / A1):** "5 tentativas" quer dizer **5 retentativas depois do envio inicial, num total de até 6 envios**. Cada um dos 5 intervalos fechados em [09:17] Larissa separa dois envios consecutivos.

```ts
const RETRY_BACKOFF_MS = [
  1 * 60_000,        // após o 1º envio falho
  5 * 60_000,        // após o 2º
  30 * 60_000,       // após o 3º
  2 * 3_600_000,     // após o 4º
  12 * 3_600_000,    // após o 5º
] as const;
const MAX_SENDS = RETRY_BACKOFF_MS.length + 1; // 6
```

1. Na falha, `n = row.attempts + 1` (envios já feitos, contando o atual).
2. Se `n < MAX_SENDS`, atualiza a linha:
   - `status = 'FAILED'` (🔶 P-06);
   - `attempts = n`;
   - `nextAttemptAt = now + RETRY_BACKOFF_MS[n - 1]`;
   - `lastError = <motivo resumido, ≤ 500 chars>`;
   - `lockedAt = null`.

   Loga `webhook_delivery_failed` (`warn`) com `attempt`, `nextAttemptAt` e `httpStatus`/`error`.
3. Se `n === MAX_SENDS`, segue para a DLQ (4.d).
4. O worker volta a pegar a linha quando `nextAttemptAt <= NOW()` (a consulta de 4.b inclui `FAILED`).

A janela entre a 1ª falha e o último envio é 1m + 5m + 30m + 2h + 12h = **14h36m**. Isso bate com "Total de quase 15 horas entre primeira falha e última tentativa" ([09:17] Diego), e é o argumento principal da P-01: com só 5 envios, a soma dos intervalos seria 2h36m.

### 4.d Fluxo de DLQ e replay manual

**ADRs:** [ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md), [ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md), [ADR-007](adrs/ADR-007-autorizacao-dos-endpoints-de-webhook.md).

**Critérios de entrada na DLQ:**

- o 6º envio falhou (P-01);
- o payload passa de 64KB (P-08);
- a secret não pode ser decifrada.

**Movimentação** (no worker, numa única `prisma.$transaction`):

1. `tx.webhookDeadLetter.create({ eventId, webhookId, orderId, eventType, payload, failureReason, lastHttpStatus, attempts: n, failedAt: now })`. O conteúdo da linha segue D7 ([09:18] Diego: "payload, motivo da falha e timestamp").
2. `tx.webhookOutbox.delete({ where: { id } })`. 🔶 **P-06:** a outbox nunca guarda eventos mortos, o que mantém "mais limpa a leitura da outbox principal" ([09:18] Diego).
3. Loga `webhook_event_dead_lettered` (`error`) com `eventId`, `webhookId`, `orderId`, `failureReason` e `attempts`.

**Replay manual** (`POST /api/v1/admin/webhooks/dead-letter/:id/replay`; D8, [09:18] Diego):

1. `authenticate` valida o JWT (`src/middlewares/auth.middleware.ts:27-47`).
2. `requireRole('ADMIN')` (`auth.middleware.ts:49-61`) devolve `403 FORBIDDEN` para OPERATOR (D17, [09:36] Sofia e Larissa).
3. `validate({ params: deadLetterIdParamSchema })` checa o UUID.
4. `WebhookService.replayDeadLetter(id, req.user.id)`, numa única `$transaction`:
   1. Busca a linha da DLQ. Se não existir, lança `WebhookDeadLetterNotFoundError` (404).
   2. Se `replayedAt != null`, lança `WebhookDeadLetterAlreadyReplayedError` (409). 🔶 P-04: o replay é idempotente e não reenfileira duas vezes.
   3. Busca o endpoint. Se estiver `deletedAt != null` ou `active = false`, lança `WebhookInactiveError` (409).
   4. Se `Buffer.byteLength(JSON.stringify(payload)) > 65536`, lança `WebhookPayloadTooLargeError` (422).
   5. `tx.webhookOutbox.create({ eventId: dlq.eventId, webhookId, orderId, eventType, payload, status: 'PENDING', attempts: 0, nextAttemptAt: now })`.
      - 🔶 **P-04 (questão 4 do RFC):** o replay **mantém o mesmo `eventId`**, e portanto o mesmo `X-Event-Id` e o mesmo `event_id` no corpo, que é o snapshot original. Assim a deduplicação do cliente ([ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md)) continua funcionando.
      - O contador `attempts` recomeça em 0, o que dá ao evento um ciclo completo de retry.
      - O `@@unique([eventId, webhookId])` impede ter duas cópias ativas na outbox. Uma violação (`P2002`) é convertida em `WebhookDeadLetterAlreadyReplayedError`.
   6. Atualiza a DLQ com `replayedAt = now` e `replayedById = req.user.id`. A linha **é mantida** como trilha de auditoria (RF8).
5. Loga `webhook_dead_letter_replayed` (`info`) com `userId`, `deadLetterId`, `eventId` e `webhookId`, o que atende RF8 ([09:36] Sofia: "tem que logar quem fez o replay").
6. Responde `202 Accepted` (ver 5.8).

Se o evento reprocessado falhar de novo, ele volta à DLQ como **uma nova linha** com o mesmo `eventId`. O histórico de replays fica preservado.

---

## 5. Contratos públicos

Convenções herdadas do código:

- Todas as rotas ficam sob `/api/v1` (`src/app.ts:67`).
- O corpo de erro é `{ "error": { "code", "message", "details?" } }` (`src/middlewares/error.middleware.ts:15-24`).
- Listas usam `{ data, pagination }` (`src/shared/http/response.ts`).
- Na API interna os campos são camelCase, como nos módulos existentes. O payload **enviado ao cliente** usa snake_case, conforme [09:43] Diego.
- **Autenticação/autorização** ([ADR-007](adrs/ADR-007-autorizacao-dos-endpoints-de-webhook.md)): o CRUD exige `Authorization: Bearer <JWT>` com qualquer role (D18), e as rotas `/admin/...` exigem role ADMIN (D17).
- Erros comuns a todas as rotas autenticadas: `401 UNAUTHORIZED` (token ausente ou inválido, de `authenticate`) e `400 VALIDATION_ERROR` (Zod via `validate`).

Recurso `Webhook` (representação pública; **a secret nunca é devolvida**, exceto em 5.1 e 5.7):

```json
{
  "id": "3c1e9d2a-7b4f-4e8a-9c0d-1f2e3a4b5c6d",
  "customerId": "9a8b7c6d-5e4f-4a3b-2c1d-0e9f8a7b6c5d",
  "url": "https://hooks.atlas.example.com/oms",
  "events": ["SHIPPED", "DELIVERED"],
  "active": true,
  "secretRotatedAt": null,
  "previousSecretExpiresAt": null,
  "createdAt": "2026-10-03T13:00:00.000Z",
  "updatedAt": "2026-10-03T13:00:00.000Z"
}
```

### 5.1 `POST /api/v1/webhooks`: cadastrar webhook

- **Auth:** qualquer usuário autenticado (D18).
- **Request:** `customerId` vai no body (D15, [09:32] Larissa).
  ```json
  {
    "customerId": "9a8b7c6d-5e4f-4a3b-2c1d-0e9f8a7b6c5d",
    "url": "https://hooks.atlas.example.com/oms",
    "events": ["SHIPPED", "DELIVERED"]
  }
  ```
- **Validação (schema Zod `createWebhookSchema`, com `.strict()`):**
  - `customerId`: uuid.
  - `url`: `z.string().url().max(2048)`.
  - `events`: array não vazio e sem repetição de `PAID | PROCESSING | SHIPPED | DELIVERED | CANCELLED`. 🔶 **P-05:** `PENDING` fica fora. Nenhuma transição de `order.status.ts` leva a `PENDING`, e a criação não gera evento.
  - O campo `secret` não é aceito, porque é gerado pela plataforma (RF2, [09:31] Marcos). Por causa do `.strict()`, enviá-lo resulta em `VALIDATION_ERROR`.
- **Response `201 Created`:** o recurso `Webhook` mais a secret, **exibida apenas nesta resposta**:
  ```json
  {
    "id": "3c1e9d2a-7b4f-4e8a-9c0d-1f2e3a4b5c6d",
    "customerId": "9a8b7c6d-5e4f-4a3b-2c1d-0e9f8a7b6c5d",
    "url": "https://hooks.atlas.example.com/oms",
    "events": ["SHIPPED", "DELIVERED"],
    "active": true,
    "secret": "whsec_Q2hhbmdlTWVJblByb2R1Y3Rpb24tMzJieXRlcw",
    "secretRotatedAt": null,
    "previousSecretExpiresAt": null,
    "createdAt": "2026-10-03T13:00:00.000Z",
    "updatedAt": "2026-10-03T13:00:00.000Z"
  }
  ```
  🔶 **P-11:** a secret tem o formato `whsec_` + base64url de 32 bytes aleatórios (`crypto.randomBytes`).
- **Erros:**

  | Status | Código | Quando |
  |---|---|---|
  | 400 | `VALIDATION_ERROR` | corpo inválido |
  | 400 | `WEBHOOK_INVALID_URL` | URL bem formada, mas não `https` |
  | 401 | `UNAUTHORIZED` | token ausente ou inválido |
  | 404 | `WEBHOOK_CUSTOMER_NOT_FOUND` | `customerId` não existe |

### 5.2 `GET /api/v1/webhooks?customerId=<uuid>&page=1&pageSize=20`: listar webhooks de um customer

- **Auth:** qualquer usuário autenticado (D18).
- 🔶 **P-24:** na listagem o `customerId` vai na **query string**, seguindo o precedente de `listOrdersQuerySchema` (`src/modules/orders/order.schemas.ts:23-30`). D15 fala em "body ou path", então isto precisa ser validado.
- **Response `200 OK`** (exclui `deletedAt != null`):
  ```json
  {
    "data": [
      {
        "id": "3c1e9d2a-7b4f-4e8a-9c0d-1f2e3a4b5c6d",
        "customerId": "9a8b7c6d-5e4f-4a3b-2c1d-0e9f8a7b6c5d",
        "url": "https://hooks.atlas.example.com/oms",
        "events": ["SHIPPED", "DELIVERED"],
        "active": true,
        "secretRotatedAt": null,
        "previousSecretExpiresAt": null,
        "createdAt": "2026-10-03T13:00:00.000Z",
        "updatedAt": "2026-10-03T13:00:00.000Z"
      }
    ],
    "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
  }
  ```
- **Erros:** `400 VALIDATION_ERROR` (sem `customerId` ou com valor que não é uuid) e `401 UNAUTHORIZED`.

### 5.3 `GET /api/v1/webhooks/:id`: detalhar webhook

- **Auth:** qualquer usuário autenticado.
- **Response `200 OK`:** o recurso `Webhook`.
- **Erros:** `400 VALIDATION_ERROR`, `401 UNAUTHORIZED` e `404 WEBHOOK_NOT_FOUND` (também quando foi removido, P-12).

### 5.4 `PATCH /api/v1/webhooks/:id`: editar webhook

- **Auth:** qualquer usuário autenticado.
- **Request:** todos os campos são opcionais, com pelo menos um obrigatório.
  ```json
  { "url": "https://hooks.atlas.example.com/v2/oms", "events": ["PAID", "SHIPPED", "DELIVERED"], "active": true }
  ```
- **Response `200 OK`:** o recurso `Webhook` atualizado.
  - 🔶 **P-12:** eventos já na outbox usam a URL vigente **no momento do envio**. O payload continua sendo o snapshot.
  - Desativar (`active: false`) faz o worker descartar os eventos pendentes desse endpoint.
- **Erros:**

  | Status | Código | Quando |
  |---|---|---|
  | 400 | `VALIDATION_ERROR` | corpo inválido |
  | 400 | `WEBHOOK_INVALID_URL` | URL não `https` |
  | 401 | `UNAUTHORIZED` | token ausente ou inválido |
  | 404 | `WEBHOOK_NOT_FOUND` | id inexistente ou removido |

### 5.5 `DELETE /api/v1/webhooks/:id`: remover webhook

- **Auth:** qualquer usuário autenticado.
- **Response `204 No Content`.**
  - 🔶 **P-12:** é **soft delete** (`deletedAt = now`, `active = false`). Na mesma transação, as linhas `PENDING` e `FAILED` desse endpoint são apagadas da outbox.
  - O histórico de entregas e a DLQ ficam preservados para auditoria.
- **Erros:** `400 VALIDATION_ERROR`, `401 UNAUTHORIZED` e `404 WEBHOOK_NOT_FOUND`.

### 5.6 `GET /api/v1/webhooks/:id/deliveries?page=1&pageSize=100`: histórico de entregas

- **Auth:** qualquer usuário autenticado.
- 🔶 **P-10 (A4):** os "últimos 100" ([09:34] Marcos) viram uma lista paginada em ordem `createdAt desc`, com `pageSize` **padrão 100 e máximo 100**. Cada item é uma tentativa HTTP.
- **Response `200 OK`:**
  ```json
  {
    "data": [
      {
        "id": "a1b2c3d4-e5f6-4a7b-8c9d-0e1f2a3b4c5d",
        "eventId": "6f1c2a9e-3b7d-4d0a-9a51-2f5a8c0e7b14",
        "attemptNumber": 2,
        "success": true,
        "httpStatus": 200,
        "durationMs": 184,
        "requestBody": { "event_id": "6f1c2a9e-3b7d-4d0a-9a51-2f5a8c0e7b14", "event_type": "order.status_changed", "to_status": "PROCESSING" },
        "responseBody": "{\"received\":true}",
        "errorMessage": null,
        "createdAt": "2026-10-03T14:23:07.441Z"
      }
    ],
    "pagination": { "page": 1, "pageSize": 100, "total": 1, "totalPages": 1 }
  }
  ```
  O `requestBody` está abreviado no exemplo. Na resposta real ele traz o payload completo.
- **Erros:** `400 VALIDATION_ERROR`, `401 UNAUTHORIZED` e `404 WEBHOOK_NOT_FOUND`.

### 5.7 `POST /api/v1/webhooks/:id/secret/rotate`: rotacionar secret

- **Auth:** qualquer usuário autenticado (D18). A rotação é pedida "pela API" ([09:21] Sofia).
- **Request:** sem corpo.
- **Comportamento** (D9, [09:21] Sofia):
  - gera uma nova secret;
  - a atual passa para `previousSecretCiphertext`, com `previousSecretExpiresAt = now + 24h`.
  - 🔶 **P-25:** uma nova rotação dentro do grace period **substitui** a secret anterior, e a mais antiga morre na hora.
- **Response `200 OK`:**
  ```json
  {
    "webhookId": "3c1e9d2a-7b4f-4e8a-9c0d-1f2e3a4b5c6d",
    "secret": "whsec_TmV3U2VjcmV0QWZ0ZXJSb3RhdGlvbi0zMmJ5dGVz",
    "secretRotatedAt": "2026-10-03T15:00:00.000Z",
    "previousSecretExpiresAt": "2026-10-04T15:00:00.000Z"
  }
  ```
- **Erros:** `400 VALIDATION_ERROR`, `401 UNAUTHORIZED` e `404 WEBHOOK_NOT_FOUND`.

### 5.8 `POST /api/v1/admin/webhooks/dead-letter/:id/replay`: replay de DLQ

- **Auth:** `authenticate` mais `requireRole('ADMIN')` (D17, [ADR-007](adrs/ADR-007-autorizacao-dos-endpoints-de-webhook.md)).
- **Request:** sem corpo.
- **Response `202 Accepted`** (🔶 P-26: 202 porque o evento é reenfileirado, não entregue na hora):
  ```json
  {
    "deadLetterId": "d4c3b2a1-0f9e-4d8c-7b6a-5e4d3c2b1a0f",
    "eventId": "6f1c2a9e-3b7d-4d0a-9a51-2f5a8c0e7b14",
    "webhookId": "3c1e9d2a-7b4f-4e8a-9c0d-1f2e3a4b5c6d",
    "outboxStatus": "PENDING",
    "replayedAt": "2026-10-04T09:12:00.000Z",
    "replayedById": "1e2d3c4b-5a69-4788-9a0b-c1d2e3f4a5b6"
  }
  ```
- **Erros:**

  | Status | Código | Quando |
  |---|---|---|
  | 400 | `VALIDATION_ERROR` | id não é uuid |
  | 401 | `UNAUTHORIZED` | token ausente ou inválido |
  | 403 | `FORBIDDEN` | role diferente de ADMIN (vem de `requireRole`) |
  | 404 | `WEBHOOK_DEAD_LETTER_NOT_FOUND` | id da DLQ inexistente |
  | 409 | `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` | já reprocessado, ou já há cópia ativa na outbox |
  | 409 | `WEBHOOK_INACTIVE` | endpoint removido ou desativado |
  | 422 | `WEBHOOK_PAYLOAD_TOO_LARGE` | payload acima de 64KB |

### 5.9 `GET /api/v1/admin/webhooks/dead-letter?webhookId=&replayed=false&page=1&pageSize=20`: listar DLQ

- 🔶 **P-27:** este endpoint **não foi discutido na reunião**. Proponho porque, sem ele, o ADMIN não tem como descobrir o `:id` a reprocessar.
- **Auth:** ADMIN (mesma regra do replay).
- **Response `200 OK`:** `{ data: [{ id, eventId, webhookId, orderId, failureReason, lastHttpStatus, attempts, failedAt, replayedAt, replayedById }], pagination }`. O `payload` não é incluído na listagem.
- **Erros:** `400 VALIDATION_ERROR`, `401 UNAUTHORIZED` e `403 FORBIDDEN`.

---

## 6. Matriz de erros

O padrão segue [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md): as classes novas ficam em `src/shared/errors/http-errors.ts`, no molde de `InvalidStatusTransitionError`/`InsufficientStockError`, e são reexportadas em `src/shared/errors/index.ts`. Os códigos usam o prefixo `WEBHOOK_` (D12, [09:29] Larissa).

| Código | HTTP | Quando ocorre | Classe (reusar / criar) |
|---|---|---|---|
| `WEBHOOK_NOT_FOUND` | 404 | Webhook inexistente ou removido (5.3–5.7) | **Criar** `WebhookNotFoundError extends AppError` (404). `NotFoundError` tem o código `NOT_FOUND` fixo (`http-errors.ts:27-31`) e não serve |
| `WEBHOOK_CUSTOMER_NOT_FOUND` | 404 | `customerId` do cadastro não existe (5.1) | **Criar** `WebhookCustomerNotFoundError extends AppError` (404) |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | Id de DLQ inexistente (5.8) | **Criar** `WebhookDeadLetterNotFoundError extends AppError` (404) |
| `WEBHOOK_INVALID_URL` | 400 | URL bem formada, mas com protocolo diferente de `https` (5.1, 5.4) | **Criar** `WebhookInvalidUrlError extends BadRequestError`. 🔶 P-28: checado no **service** |
| `VALIDATION_ERROR` | 400 | Corpo, query ou params inválidos no Zod (formato de URL, `events` vazio ou com `PENDING`, uuid inválido, campo `secret` enviado) | **Reusar** `ValidationError`, emitido pelo `validate` (`src/middlewares/validate.middleware.ts:26-31`). É a única exceção ao prefixo: o middleware compartilhado sempre emite esse código ([ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md), consequências negativas) |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 422 (no replay) · interno (no worker) | Corpo serializado > 64KB (RNF5). No replay (5.8) vira resposta HTTP. No worker vira `failureReason` da DLQ | **Criar** `WebhookPayloadTooLargeError extends UnprocessableEntityError`, com `details: { sizeBytes, limitBytes: 65536 }` |
| `WEBHOOK_SECRET_REQUIRED` | interno (worker) | O endpoint não tem secret utilizável para assinar (cifra corrompida ou chave errada). O código foi citado em [09:28] Bruno | **Criar** `WebhookSecretRequiredError extends AppError` (500). Nunca chega a um cliente HTTP: é lançado no worker, vira `failureReason` da DLQ e gera log `error` |
| `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` | 409 | DLQ com `replayedAt` preenchido, ou cópia já ativa na outbox (`P2002` em `[eventId, webhookId]`) | **Criar** `WebhookDeadLetterAlreadyReplayedError extends ConflictError` |
| `WEBHOOK_INACTIVE` | 409 | Replay para um endpoint removido ou desativado (5.8) | **Criar** `WebhookInactiveError extends ConflictError` |
| `UNAUTHORIZED` | 401 | Token ausente ou inválido | **Reusar** `UnauthorizedError`, via `authenticate` |
| `FORBIDDEN` | 403 | Role diferente de ADMIN em `/admin/...` | **Reusar** `ForbiddenError`, via `requireRole` |

**Motivos de falha internos** (gravados em `webhook_outbox.lastError` e `webhook_dead_letter.failureReason`; não são erros HTTP): `HTTP_TIMEOUT`, `HTTP_NETWORK_ERROR`, `HTTP_STATUS_<nnn>` (inclui 3xx, P-22), `PAYLOAD_TOO_LARGE` e `WEBHOOK_SECRET_REQUIRED`.

---

## 7. Estratégias de resiliência

### 7.1 Timeouts

- **HTTP:** `AbortSignal.timeout(10_000)` por envio. Estourou, conta como falha e vai para retry (RNF6, [09:42] Diego).
- **Transação de `changeStatus`:** continua com o default do Prisma, já que `order.service.ts` não configura timeout. O trabalho acrescentado é uma leitura indexada (`[customerId, active]`) e um `createMany`, sem I/O externo (OBJ-03).

### 7.2 Retries e backoff

- Até **6 envios**, com intervalos de 1m/5m/30m/2h/12h ([ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md); 🔶 P-01).
- 🔶 **P-23:** **todo status fora de 2xx é retentado**, inclusive 4xx. Diferenciar 4xx permanente de transitório não foi discutido, e um 401/404 do lado do cliente pode ser só uma configuração temporária.
- O worker não faz retry dentro do mesmo ciclo. O próximo envio só acontece depois do intervalo.

### 7.3 Fallback

O fallback é a DLQ mais o replay manual por ADMIN (4.d). Não há fallback automático nem aviso ao cliente nesta fase: o e-mail ficou adiado ([09:37] Larissa).

### 7.4 Queda do worker no meio de um batch

🔶 **P-06 e P-17** (a inconsistência da seção 7 do RFC). Os estados da outbox ficam assim:

| Estado | Significado | Quem lê |
|---|---|---|
| `PENDING` | Aguardando o 1º envio, ou reenfileirado por replay | worker |
| `PROCESSING` | Reivindicado no ciclo corrente (`lockedAt` preenchido) | ninguém (lock) |
| `FAILED` | **Falha transitória**: ainda há envios restantes e `nextAttemptAt` está no futuro | worker, quando `nextAttemptAt <= now` |
| `DELIVERED` | 2xx recebido. Fica na outbox até o arquivamento, que está fora de escopo | ninguém |

A falha **permanente** não é um estado da outbox: a linha migra para `webhook_dead_letter`.

Se o processo cair com linhas em `PROCESSING`:

- No **próximo boot**, a recuperação (4.b, bootstrap passo 3) devolve essas linhas para `PENDING` **sem incrementar `attempts`**.
- Uma linha cujo HTTP já tinha sido enviado mas cujo resultado não foi gravado será **reenviada**. Isso é aceito pela semântica at-least-once, e o cliente deduplica por `X-Event-Id` ([ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md)).
- Enquanto o worker estiver fora, os eventos se acumulam na outbox sem perda ([ADR-002](adrs/ADR-002-worker-separado-em-polling.md), consequências).

### 7.5 Queda da API ou do banco

- **Queda da API:** não afeta o worker, que roda em outro processo (D4, [09:11] Diego).
- **Banco indisponível:** o `changeStatus` falha como já falharia hoje. O worker loga `webhook_worker_tick_failed` e tenta de novo no ciclo seguinte (4.b, passo 5).

---

## 8. Observabilidade

### 8.1 Métricas sugeridas

O projeto não tem biblioteca de métricas. 🔶 **P-29:** nesta fase, as métricas são **derivadas de logs estruturados e de consultas SQL**, sem dependência nova. O worker loga `webhook_worker_stats` a cada 60s com os números abaixo.

| Métrica | Como obter |
|---|---|
| `outbox_pending_count` / `outbox_failed_count` | `COUNT(*)` por `status` em `webhook_outbox` |
| `outbox_oldest_pending_age_seconds` | `NOW() - MIN(created_at)` onde `status IN ('PENDING','FAILED')`. É o principal alarme de backlog e de worker parado |
| `dead_letter_size` / `dead_letter_unreplayed` | `COUNT(*)` em `webhook_dead_letter` (total e com `replayed_at IS NULL`) |
| `delivery_success_rate{webhookId}` | `webhook_deliveries` agrupado por `webhook_id` e janela de tempo |
| `delivery_latency_ms` | `delivered_at - created_at` na outbox. Mede o caminho do commit até o 2xx, contra OBJ-01 |
| `http_duration_ms{webhookId}` | `webhook_deliveries.duration_ms` |

### 8.2 Logs estruturados

Seguem a convenção de `src/shared/logger/index.ts`: objeto no 1º argumento, mensagem como nome de evento em `snake_case` (como `server_started`, `http_request`), timestamp ISO e redact.

| Evento | Nível | Processo | Campos |
|---|---|---|---|
| `webhook_event_enqueued` | `info` | API | `eventId`, `orderId`, `toStatus`, `webhookIds[]` |
| `webhook_worker_started` / `webhook_worker_stopped` | `info` | worker | `pollIntervalMs`, `batchSize`, `recoveredRows` |
| `webhook_delivery_succeeded` | `info` | worker | `eventId`, `webhookId`, `orderId`, `attempt`, `httpStatus`, `durationMs` |
| `webhook_delivery_failed` | `warn` | worker | idem acima, mais `error`, `nextAttemptAt` |
| `webhook_event_dead_lettered` | `error` | worker | `eventId`, `webhookId`, `orderId`, `failureReason`, `attempts` |
| `webhook_event_discarded` | `warn` | worker | `eventId`, `webhookId`, `reason` (`inactive`/`deleted`) |
| `webhook_worker_tick_failed` | `error` | worker | `err` |
| `webhook_worker_stats` | `info` | worker | as métricas de 8.1 |
| `webhook_dead_letter_replayed` | `info` | API | `userId`, `deadLetterId`, `eventId`, `webhookId` (RF8) |
| `webhook_secret_rotated` | `info` | API | `userId`, `webhookId`, `previousSecretExpiresAt` (sem a secret) |

**Mudanças no logger** (ver também a seção 12):

- `createLogger` passa a aceitar um `service` opcional (default `'order-management-api'`), para que o worker use `'order-management-worker'`.
- A lista `redact` ganha `'*.secret'`, `'*.secretCiphertext'`, `'*.previousSecretCiphertext'` e `'*.headers["x-signature"]'` ([ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md)).

### 8.3 Tracing e correlação

- **API → outbox:** o `request-logger` já loga `requestId` e `path` (que contém o `orderId`) a cada requisição (`src/middlewares/request-logger.middleware.ts:14-24`). O evento `webhook_event_enqueued` loga `orderId` e `eventId`. Assim, `requestId → orderId → eventId`.
  - 🔶 **P-30:** o `requestId` **não** é persistido na outbox. Isso evitaria mudar a assinatura de `changeStatus`, que hoje não recebe `req`. Persistir seria uma extensão futura.
- **Outbox → worker → cliente:** todos os logs do worker carregam `eventId`, que é o `X-Event-Id` enviado ao cliente. Quando um cliente reportar um problema, o `X-Event-Id` que ele recebeu localiza, sem ambiguidade, as linhas de `webhook_deliveries` e `webhook_dead_letter` e os logs do worker.
- O `X-Request-Id` interno **não** é enviado ao cliente.

---

## 9. Dependências e compatibilidade

### 9.1 Bibliotecas

🔶 **P-15:** **nenhuma dependência npm nova.** Isso é inferência sobre Node ≥ 20 (`package.json`, `engines`), coerente com "Não vamos botar nada novo" ([09:29] Bruno).

| Necessidade | Solução | Já no projeto? |
|---|---|---|
| HMAC-SHA256 | `node:crypto` → `createHmac('sha256', secret)` | nativo |
| Geração de secret | `node:crypto` → `randomBytes(32)` | nativo |
| Cifra da secret em repouso | `node:crypto` → AES-256-GCM (🔶 P-11) | nativo |
| Cliente HTTP com timeout | `fetch` global e `AbortSignal.timeout` (Node 20) | nativo |
| UUID do evento | `uuid` v4 | sim (`uuid` 11.0.3) |
| Validação | `zod` | sim |
| Logs | `pino` | sim |

### 9.2 Variáveis de ambiente (`src/config/env.ts`)

Entram no `envSchema` existente:

| Variável | Tipo / default | Uso |
|---|---|---|
| `WEBHOOK_SECRET_ENCRYPTION_KEY` | string base64 de 32 bytes, **obrigatória** | Cifrar e decifrar secrets. A API cifra no cadastro e na rotação, e o worker decifra na assinatura (🔶 P-11) |
| `WEBHOOK_POLL_INTERVAL_MS` | int, default `2000` | D3 |
| `WEBHOOK_BATCH_SIZE` | int, default `10` | 🔶 P-19 |
| `WEBHOOK_HTTP_TIMEOUT_MS` | int, default `10000` | RNF6 |

Como `env.ts` faz `process.exit(1)` quando o env é inválido, `WEBHOOK_SECRET_ENCRYPTION_KEY` também precisa ser adicionada ao `.env.example` e ao ambiente de testes.

### 9.3 Banco e migrations

- Uma migration nova (`npm run db:migrate` → `prisma migrate dev --name webhooks`) cria as 4 tabelas e o enum da seção 4.0.
- É **aditiva**: nenhuma coluna existente é alterada. `Customer` e `User` ganham apenas relações reversas no schema Prisma, sem DDL nas tabelas `customers` e `users` além das FKs que partem das tabelas novas.
- 🔶 **P-13:** `webhook_outbox.order_id` e `webhook_dead_letter.order_id` **não têm FK** para `orders`. `DELETE /orders/:id` é permitido em `CANCELLED` (`order.service.ts:184-190`), e uma FK com cascade apagaria o evento `→ CANCELLED` ainda não entregue.

### 9.4 Scripts (`package.json`)

- `"worker": "node --env-file=.env dist/worker.js"`. O `npm run worker` é o nome citado em [09:11] Larissa.
- `"worker:dev": "tsx watch --env-file=.env src/worker.ts"`.
- Não é preciso mudar `tsconfig.build.json`, que já inclui `src/**/*.ts` e portanto `src/worker.ts`.

---

## 10. Critérios de aceite técnicos

- [ ] **CA-01:** `PATCH /orders/:id/status` para um status assinado por um webhook ativo do customer cria **exatamente uma** linha `PENDING` em `webhook_outbox` por endpoint assinante, todas com o mesmo `eventId`.
- [ ] **CA-02:** se nenhum webhook ativo do customer assina o `toStatus`, **nenhuma** linha é criada (D16).
- [ ] **CA-03:** se o insert na outbox falhar (forçado em teste), o status do pedido, o `order_status_history` e o estoque **não mudam** (rollback, D19).
- [ ] **CA-04:** o payload gravado é o snapshot do momento da mudança. Alterar o pedido depois não altera o payload enviado (D22).
- [ ] **CA-05:** o worker envia `Content-Type`, `X-Event-Id`, `X-Webhook-Id`, `X-Timestamp` e `X-Signature`, e o `X-Signature` confere com `HMAC-SHA256(secret, body)` calculado pelo teste.
- [ ] **CA-06:** um evento só fica `DELIVERED` depois de uma resposta 2xx. Timeout de 10s, erro de rede ou status fora de 2xx levam a `FAILED`, com `nextAttemptAt` conforme `RETRY_BACKOFF_MS`.
- [ ] **CA-07:** depois do 6º envio falho (P-01), a linha sai da outbox e aparece em `webhook_dead_letter` com payload, `failureReason` e `failedAt`.
- [ ] **CA-08:** o replay exige ADMIN. OPERATOR recebe `403 FORBIDDEN`, e ADMIN recebe `202` com a linha de volta à outbox em `PENDING` e **o mesmo `eventId`**.
- [ ] **CA-09:** um segundo replay da mesma linha de DLQ devolve `409 WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED`.
- [ ] **CA-10:** o replay grava `replayedById` e loga `webhook_dead_letter_replayed` com `userId` (RF8).
- [ ] **CA-11:** `POST`/`PATCH` com URL `http://…` devolvem `400 WEBHOOK_INVALID_URL`.
- [ ] **CA-12:** a secret aparece só nas respostas de criação e rotação. `GET` não a devolve, e ela nunca aparece em log, porque o redact a mascara.
- [ ] **CA-13:** depois da rotação, durante 24h, o `X-Signature` traz as duas assinaturas. Depois das 24h, traz só a nova.
- [ ] **CA-14:** um corpo serializado acima de 65.536 bytes não é enviado. No worker vai direto para a DLQ com `PAYLOAD_TOO_LARGE`, e no replay devolve `422 WEBHOOK_PAYLOAD_TOO_LARGE`.
- [ ] **CA-15:** linhas em `PROCESSING` no boot do worker voltam para `PENDING`.
- [ ] **CA-16:** eventos do mesmo `orderId` e do mesmo endpoint são enviados em ordem de `created_at` quando não há falhas (D5).
- [ ] **CA-17:** nenhum teste existente em `tests/orders.test.ts` ou `tests/auth.test.ts` quebra. `npm test` passa por inteiro.
- [ ] **CA-18:** `npm run worker` sobe o processo separado, e SIGTERM encerra depois do ciclo corrente, com `prisma.$disconnect()`.

---

## 11. Riscos e mitigação

| Risco técnico | Impacto | Mitigação |
|---|---|---|
| **Ordenação quebrada por retry.** O evento A de um pedido falha e fica `FAILED` por 1m, e o evento B do mesmo pedido é entregue antes | O cliente recebe `SHIPPED` antes de `PROCESSING`. D5 só garante ordem "em ordem de created_at" ([09:12] Diego), e a interação com retry não foi discutida | 🔶 **P-20:** nesta fase isso é aceito e documentado para os clientes. O payload traz `from_status`, `to_status` e `timestamp`, o que permite reordenar do lado deles. A alternativa (bloquear eventos posteriores do mesmo `orderId`+`webhookId` enquanto houver um anterior não entregue) fica listada para a equipe decidir |
| **Perda da `WEBHOOK_SECRET_ENCRYPTION_KEY`** | Todas as secrets ficam irrecuperáveis. O worker manda tudo para a DLQ com `WEBHOOK_SECRET_REQUIRED` | Guardar a chave no mesmo cofre do `JWT_SECRET`. Rotacionar todas as secrets é o plano de recuperação. Levar à revisão da Sofia (RNF10) |
| **Transação de `changeStatus` mais lenta** | Mais contenção e risco de chegar perto do timeout default do Prisma | Índice `[customerId, active]` e `createMany` num único statement. O fan-out é limitado ao número de endpoints do customer |
| **Bytes diferentes entre o que foi assinado e o que foi enviado** | O cliente rejeita a assinatura | Assinar e enviar **a mesma string** `body` gerada no worker, sem reserializar no `fetch` |
| **SSRF / URLs internas.** Um usuário autenticado cadastra `https://10.0.0.5/…` | O worker faz requisições para a rede interna | Não discutido na reunião. Levar à revisão da Sofia. `redirect: 'manual'` (P-22) já fecha o desvio por redirect |
| **Endpoint lento travando o batch** | Até 10s por envio atrasa outros clientes, contra OBJ-01 | Paralelismo entre grupos de `orderId` (P-20) e batch pequeno (P-19) |
| **Crescimento das tabelas** (`webhook_outbox` `DELIVERED` e `webhook_deliveries`) | Polling e consultas de histórico ficam mais lentos | Índices em `[status, nextAttemptAt]` e `[webhookId, createdAt]`. `responseBody` truncado em 2KB. O arquivamento está fora de escopo ([09:08] Diego) e deve ser monitorado com `outbox_pending_count` e o tamanho das tabelas |
| **Duplicata depois de crash** | O cliente recebe o mesmo evento duas vezes | Aceito pelo at-least-once ([ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md)). O `X-Event-Id` é estável, inclusive no replay (P-04) |
| **`tests/setup.ts` sem limpar as tabelas novas** | Falha de FK no `deleteMany` de `customers`/`users` entre testes | Acrescentar o `deleteMany` das 4 tabelas **antes** de `customer` e `user` (seção 12) |

---

## 12. Integração com o sistema existente

Todos os caminhos abaixo existem no repositório. A coluna "Tipo" diz se o arquivo é alterado ou usado como molde.

| # | Arquivo real | Tipo | Como o módulo de webhooks se integra |
|---|---|---|---|
| 1 | `src/modules/orders/order.service.ts` | **Alterado** | Em `changeStatus` (l.126-179), uma chamada `await publishWebhookEvent(tx, order, from, to)` entra logo depois de `tx.orderStatusHistory.create` (l.159-167), dentro da mesma `$transaction`. Novo import de `../webhooks/webhook.publisher.js`. O construtor (`orderRepository, prisma`) e o retorno não mudam (D20, [ADR-001](adrs/ADR-001-outbox-no-mysql.md), [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md)) |
| 2 | `src/shared/errors/http-errors.ts` | **Alterado** | Recebe as 8 classes da matriz de erros (seção 6), no molde de `InvalidStatusTransitionError`/`InsufficientStockError` (l.45-63), estendendo `AppError`, `BadRequestError`, `ConflictError` ou `UnprocessableEntityError` |
| 3 | `src/shared/errors/index.ts` | **Alterado** | Reexporta as classes novas, como já faz com as atuais |
| 4 | `src/shared/logger/index.ts` | **Alterado** | (a) A lista `redactPaths` (l.4-11) ganha `*.secret`, `*.secretCiphertext`, `*.previousSecretCiphertext` e `*.headers["x-signature"]`. (b) `createLogger` aceita um `service` opcional, para que o worker logue como `order-management-worker` em vez do `base.service` fixo (l.20). O export `logger` continua igual para a API |
| 5 | `src/middlewares/auth.middleware.ts` | **Reusado sem alteração** | O router admin usa `authenticate` e `requireRole('ADMIN')` (l.27-61), no padrão por rota de `src/modules/users/user.routes.ts:12-18`. O router do CRUD usa `router.use(authenticate)`, como `src/modules/orders/order.routes.ts:14` ([ADR-007](adrs/ADR-007-autorizacao-dos-endpoints-de-webhook.md)) |
| 6 | `src/middlewares/validate.middleware.ts` | **Reusado sem alteração** | Valida body, query e params com os schemas de `webhook.schemas.ts`. Por causa dele, falhas de Zod saem como `VALIDATION_ERROR`, o que motiva P-28 |
| 7 | `src/middlewares/error.middleware.ts` | **Reusado sem alteração** | Serializa os `AppError` `WEBHOOK_*` (l.15-24) e converte `P2002` em 409. Não cobre o worker, que trata os próprios erros (4.b, passo 5) |
| 8 | `src/routes/index.ts` | **Alterado** | O tipo `Controllers` (l.13-19) ganha `webhooks: WebhookController`. `buildApiRouter` ganha `router.use('/webhooks', buildWebhookRouter(controllers.webhooks))` e `router.use('/admin/webhooks', buildWebhookAdminRouter(controllers.webhooks))` |
| 9 | `src/app.ts` | **Alterado** | `buildControllers` (l.26-53) instancia `WebhookRepository(prisma)`, `WebhookService(webhookRepository, prisma)` e `WebhookController(webhookService)`, e devolve a chave `webhooks`. `OrderService` não muda (l.43) |
| 10 | `src/config/env.ts` | **Alterado** | O `envSchema` (l.3-10) ganha as 4 variáveis da seção 9.2 |
| 11 | `src/config/database.ts` | **Reusado sem alteração** | `src/worker.ts` importa o singleton `prisma`, que é uma instância nova por processo (D14) |
| 12 | `src/server.ts` | **Molde** | `src/worker.ts` replica o bootstrap, o SIGINT/SIGTERM, o `prisma.$disconnect()` e o `logger.fatal` em `bootstrap_failed` (l.6-27) |
| 13 | `src/modules/orders/order.status.ts` | **Referência** | O enum de eventos aceitos em `events` exclui `PENDING`, que não é destino de nenhuma transição (l.3-10; P-05) |
| 14 | `src/middlewares/request-logger.middleware.ts` | **Referência** | Correlação `requestId → orderId → eventId` (8.3). A lib `uuid` já importada (l.2) é reutilizada para o `eventId` |
| 15 | `prisma/schema.prisma` | **Alterado** | 4 modelos novos e o enum `WebhookOutboxStatus` (seção 4.0). `Customer` e `User` ganham só relações reversas |
| 16 | `package.json` | **Alterado** | Scripts `worker` e `worker:dev` (seção 9.4). Nenhuma dependência nova |
| 17 | `tests/setup.ts` | **Alterado** | Em `beforeEach` (l.7-15), acrescentar `webhookDelivery`, `webhookDeadLetter`, `webhookOutbox` e `webhookEndpoint` com `deleteMany()` **antes** de `prisma.customer.deleteMany()` e `prisma.user.deleteMany()` |
| 18 | `tests/helpers/factories.ts` | **Alterado** | Nova factory `createTestWebhook({ customerId, events, url })`, reaproveitando `bootstrapAuthenticatedUser('ADMIN' \| 'OPERATOR')` para os testes de autorização |

**Arquivos novos do módulo** (padrão de `src/modules/<dominio>/`, D11):

- `src/modules/webhooks/webhook.controller.ts`
- `webhook.service.ts`
- `webhook.repository.ts`
- `webhook.routes.ts` (exporta `buildWebhookRouter` e `buildWebhookAdminRouter`)
- `webhook.schemas.ts`
- `webhook.publisher.ts` (`publishWebhookEvent`)
- `webhook.payload.ts`
- `webhook.processor.ts`
- `webhook.signer.ts`
- `webhook.crypto.ts` (cifra da secret)
- `src/worker.ts`
- `tests/webhooks.test.ts`

---

## 13. Premissas assumidas pelo FDD

> **Todas as premissas abaixo são escolhas deste FDD, não decisões da reunião.** Elas existem porque não dá para especificar fluxos e contratos sem resolver esses pontos. **Cada uma precisa ser validada com a equipe** antes da implementação. As premissas de segurança (P-02, P-03, P-11, P-22) vão para a revisão da Sofia (RNF10).

### Premissas que fecham as questões em aberto do RFC

| ID | Questão de origem | Premissa | Justificativa |
|---|---|---|---|
| **P-01** | RFC §5 Q1 / A1: 5 ou 6 envios? | "5 tentativas" = **5 retentativas**, num total de até **6 envios**. Os 5 intervalos separam envios consecutivos | Os 5 intervalos fechados em [09:17] Larissa só fazem sentido todos usados. "Quase 15 horas entre primeira falha e última tentativa" ([09:17] Diego) bate com 14h36m, e não com 2h36m |
| **P-02** | RFC §5 Q2 / A2: `X-Timestamp` fora do HMAC | O HMAC continua **só sobre o corpo**, como decidido em D9 ([09:22] Sofia). O `X-Timestamp` é informativo e **não é protegido** pela assinatura. A proteção contra replay vem do `X-Event-Id` mais a deduplicação do cliente. O corpo contém o `timestamp` do evento, que é assinado | Respeita a decisão fechada sem reinterpretá-la. Assinar `"<X-Timestamp>.<body>"` fica como **recomendação para a revisão da Sofia**, porque mudaria D9 |
| **P-03** | RFC §5 Q3 / A5 (+ A3): assinatura no grace period e formato | `X-Signature: sha256=<hex>`. Durante o grace period de 24h, as duas assinaturas vão separadas por vírgula, **a nova primeiro** (`sha256=<nova>,sha256=<antiga>`). O cliente aceita se qualquer uma conferir | O objetivo do grace period é dar tempo ao cliente "de migrar os sistemas dele" ([09:21] Sofia). Se só a nova fosse assinada, um cliente ainda não migrado rejeitaria tudo |
| **P-04** | RFC §5 Q4: `X-Event-Id` no replay | O replay **mantém o `eventId` original**, e o contador de tentativas recomeça em 0. A linha da DLQ é mantida com `replayedAt`/`replayedById`, e um segundo replay devolve 409 | Um `eventId` novo quebraria a deduplicação do cliente ([ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md)). A regra "gerado quando entra na outbox" ([09:25] Diego) passa a valer para a **1ª** entrada |
| **P-05** | RFC §5 Q5 / A6: a criação gera evento? | **Não** nesta fase. O único ponto de integração é `changeStatus`, e `PENDING` não é aceito em `events` | A reunião só citou `changeStatus` como ponto de integração ([09:40] Bruno). Nenhuma transição em `order.status.ts` leva a `PENDING`. Dá para estender depois com uma chamada em `OrderService.create` |
| **P-06** | RFC §7: estado "falhou" e a DLQ | `FAILED` é um **estado transitório** entre tentativas (aguardando `nextAttemptAt`). A falha **permanente** tira a linha da outbox e a move para `webhook_dead_letter` | Concilia os estados citados em [09:08] Diego com a DLQ separada de [09:18] Diego, sem descartar nenhuma das duas falas |

### Demais premissas de implementação

| ID | Premissa | Onde é usada |
|---|---|---|
| P-07 | `timestamp` no payload = momento da mudança de status (A7). `X-Timestamp` = momento do envio. Os dois em ISO 8601 UTC | 4.a, 4.b |
| P-08 | O limite de 64KB é checado no worker (vai direto para a DLQ) e no replay (422), **não** em `changeStatus`, para não bloquear a mudança de status | 4.a, 4.b, 4.d |
| P-09 | Fan-out na inserção: uma linha de outbox por endpoint assinante, todas com o mesmo `eventId`. A unicidade é `[eventId, webhookId]` | 4.0, 4.a |
| P-10 | Os nomes das tabelas `webhook_endpoints` e `webhook_deliveries` não foram dados na reunião. Cada entrega guarda `requestBody` e `responseBody`, este truncado em 2KB. O histórico é paginado com `pageSize` padrão e máximo de 100 (A4) | 4.0, 5.6 |
| P-11 | A secret tem formato `whsec_` + base64url de 32 bytes e é guardada **cifrada com AES-256-GCM**, com a chave em `WEBHOOK_SECRET_ENCRYPTION_KEY` | 4.0, 4.b, 5.1, 9.2 |
| P-12 | `DELETE` é soft delete e apaga os eventos `PENDING`/`FAILED` do endpoint. Endpoints inativos ou removidos têm eventos descartados pelo worker. Mudanças de URL valem para eventos já enfileirados | 4.b, 5.4, 5.5 |
| P-13 | `order_id` na outbox e na DLQ **sem FK**, para que excluir um pedido `CANCELLED` não apague o evento | 4.0, 9.3 |
| P-14 | Os "campos básicos da order" do payload se limitam a `total_cents` | 4.a |
| P-15 | Nenhuma dependência npm nova: `node:crypto` e `fetch` nativos do Node ≥ 20 (inferência) | 9.1 |
| P-16 | O arquivo de processamento se chama `webhook.processor.ts` (entre as opções de [09:28] Bruno) | 4.b |
| P-17 | No boot, o worker devolve linhas `PROCESSING` para `PENDING` sem contar a tentativa | 4.b, 7.4 |
| P-18 | O loop usa `setTimeout` reagendado depois do fim de cada ciclo, sem sobreposição | 4.b |
| P-19 | `WEBHOOK_BATCH_SIZE = 10` (o "batch pequeno" de [09:08] Diego) | 4.b, 9.2 |
| P-20 | Dentro do batch, há paralelismo entre grupos de `orderId` e ordem dentro do grupo. Ordenação sob retry não é garantida e fica documentada para os clientes | 4.b, 11 |
| P-21 | Header `User-Agent: OMS-Webhooks/1.0` | 4.b |
| P-22 | `redirect: 'manual'`: 3xx conta como falha, e redirecionamentos não são seguidos | 4.b, 6 |
| P-23 | Todo status fora de 2xx (inclusive 4xx) é retentado | 7.2 |
| P-24 | Na listagem, o `customerId` vai na query string, seguindo o precedente de `listOrdersQuerySchema`. D15 dizia "body ou path" | 5.2 |
| P-25 | Uma nova rotação dentro do grace period substitui a secret anterior, e a mais antiga expira na hora | 5.7 |
| P-26 | O replay responde `202 Accepted` | 5.8 |
| P-27 | Existe um endpoint `GET /admin/webhooks/dead-letter` para listar a DLQ, não discutido na reunião | 5.9 |
| P-28 | A verificação de `https` fica no **service**, para emitir `WEBHOOK_INVALID_URL`. Em [09:23] Sofia falou em "schema Zod", mas lá o código sairia como `VALIDATION_ERROR` | 5.1, 6 |
| P-29 | Métricas derivadas de logs (`webhook_worker_stats`) e de SQL, sem biblioteca de métricas | 8.1 |
| P-30 | O `requestId` não é persistido na outbox. A correlação passa por `orderId` → `eventId` | 8.3 |
