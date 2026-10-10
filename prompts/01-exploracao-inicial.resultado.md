# Inventário — Exploração inicial (código + transcrição)

> Saída do prompt `01-exploracao-inicial.md`. Caminhos de código relativos a `../entregavel/`. Timestamps e falantes vêm de `../entregavel/TRANSCRICAO.md`.
> Isto **não é documento final**: é matéria-prima rastreável para ADRs, RFC, FDD, PRD e Tracker.

---

## 1. CÓDIGO

### 1.1 Estrutura modular

- Cada domínio fica em `src/modules/<dominio>/` com o mesmo conjunto de arquivos: `*.controller.ts`, `*.service.ts`, `*.repository.ts`, `*.routes.ts`, `*.schemas.ts`.
  - `src/modules/auth/`: controller, routes, schemas, service. **Não tem repository próprio**: reaproveita `UserRepository` (`src/app.ts:31`).
  - `src/modules/users/`, `src/modules/customers/`, `src/modules/products/`: conjunto completo.
  - `src/modules/orders/`: conjunto completo, mais `order.status.ts` (máquina de estados).
- Composição manual das dependências (sem container de DI) em `src/app.ts:26-53` (`buildControllers`): repository → service → controller, todos recebendo o `PrismaClient` por construtor.
- Rotas: cada módulo exporta `buildXRouter(controller)`. A agregação acontece em `src/routes/index.ts:21-31`, montada em `/api/v1` (`src/app.ts:67`). O tipo `Controllers` (`src/routes/index.ts:13-19`) precisa ganhar uma chave nova para cada módulo novo.
- Entry-point HTTP: `src/server.ts`, com bootstrap, `app.listen` e graceful shutdown em SIGINT/SIGTERM que chama `prisma.$disconnect()` (linhas 13-21).
- Configuração: `src/config/env.ts` (env validado com Zod, `process.exit(1)` se for inválido) e `src/config/database.ts` (singleton `prisma` por processo, criado por `createPrismaClient()`).
- Compartilhado: `src/shared/errors/`, `src/shared/logger/`, `src/shared/http/response.ts` (helper `paginated` → `{ data, pagination }`).
- Middlewares: `src/middlewares/{auth,error,request-logger,validate}.middleware.ts`.
- Banco: `prisma/schema.prisma` (MySQL), uma migration `prisma/migrations/20260519182739_init`, seed em `prisma/seed.ts`.
- Os documentos-alvo já existem como placeholders: `docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md`, `docs/TRACKER.md`, `docs/adrs/README.md`.

### 1.2 Máquina de estados do pedido

- Enum `OrderStatus` em `prisma/schema.prisma:16-23`: `PENDING, PAID, PROCESSING, SHIPPED, DELIVERED, CANCELLED`.
- Tabela de transições em `src/modules/orders/order.status.ts:3-10`:
  - `PENDING → PAID | CANCELLED`
  - `PAID → PROCESSING | CANCELLED`
  - `PROCESSING → SHIPPED | CANCELLED`
  - `SHIPPED → DELIVERED`
  - `DELIVERED`, `CANCELLED` → terminais
- Helpers: `canTransition`, `allowedTransitions`, `isTerminal` (`order.status.ts:12-22`).
- Regras de estoque acopladas à transição:
  - `shouldDebitStock`: só em `PENDING → PAID` (`order.status.ts:24-31`).
  - `shouldReplenishStock`: `PAID|PROCESSING → CANCELLED` (`order.status.ts:33-37`).
- **A criação do pedido não passa pela máquina.** `OrderService.create` grava o pedido já em `PENDING` com um history `fromStatus: null` (`src/modules/orders/order.service.ts:95-113`). Isso não é uma transição via `changeStatus`.

### 1.3 Fluxo `changeStatus`

- Rota: `PATCH /api/v1/orders/:id/status` (`src/modules/orders/order.routes.ts:19-23`), body `{ toStatus, reason? }` (`order.schemas.ts:18-21`). Usa só `authenticate` e não exige role.
- Controller: `src/modules/orders/order.controller.ts:38-46`, que repassa `req.user.id` como `userId`.
- Service: `src/modules/orders/order.service.ts:126-179`, dentro de uma única **transação interativa** `this.prisma.$transaction(async (tx) => …)`:
  1. `tx.order.findUnique` com `items` (l.132-136), ou `NotFoundError('Order')`.
  2. Se `from === to`, lança `ConflictError` com código `INVALID_STATUS_TRANSITION` (l.140-146).
  3. Se `!canTransition`, lança `InvalidStatusTransitionError` (l.147-149).
  4. Débito ou reposição de estoque via `debitStock`/`replenishStock`, que recebem `tx` (l.151-156; implementação em l.204-243).
  5. `tx.order.update` do status (l.158).
  6. `tx.orderStatusHistory.create` com from, to, `changedById` e `reason` (l.159-167). É a auditoria.
  7. Releitura com relações e retorno (l.169-177).
- Não define isolation level nem timeout. Ficam os defaults do Prisma 5.22 (`package.json`), com timeout de transação interativa de 5s.
- Pontos naturais de extensão para o evento de webhook:
  - **Logo depois do passo 6** (l.167) e antes da releitura, chamando algo como `publishWebhookEvent(tx, order, from, to)`. O objeto `order` (l.132) já tem `orderNumber`, `customerId` e `totalCents`, e `from`/`to` já estão calculados.
  - Padrão de referência para "função que recebe `tx`": os métodos privados `debitStock(tx, …)`, `replenishStock(tx, …)` e `reserveOrderNumber(tx)` usam o alias `type TxClient = Prisma.TransactionClient` (l.24).
  - `OrderService` é construído em `src/app.ts:43` com `(orderRepository, prisma)`. Uma função pura importada não exige mudar o construtor.

### 1.4 Classes de erro e códigos

- Base `AppError(message, statusCode, errorCode, details?)` em `src/shared/errors/app-error.ts:3-16`.
- Subclasses em `src/shared/errors/http-errors.ts`:
  - Código fixo: `ValidationError` (400 `VALIDATION_ERROR`), `UnauthorizedError` (401 `UNAUTHORIZED`), `ForbiddenError` (403 `FORBIDDEN`), `NotFoundError` (404 `NOT_FOUND`, mensagem `"${resource} not found"`).
  - Código parametrizável: `BadRequestError` (400), `ConflictError` (409), `UnprocessableEntityError` (422).
  - De domínio: `InvalidStatusTransitionError extends ConflictError` (`INVALID_STATUS_TRANSITION`), `InsufficientStockError extends UnprocessableEntityError` (`INSUFFICIENT_STOCK`).
- Re-export único em `src/shared/errors/index.ts`.
- Convenção: `SCREAMING_SNAKE_CASE`. Códigos ad hoc usados hoje: `INACTIVE_PRODUCT` (`order.service.ts:70`), `INVALID_ORDER_STATE_FOR_DELETE` (`order.service.ts:187`), `EMAIL_ALREADY_USED` (`user.service.ts:24`, `customer.service.ts:32,48`), `SKU_ALREADY_USED` (`product.service.ts:33,50`).
- O projeto não usa prefixo por módulo. Os códigos de orders não começam com `ORDER_`.

### 1.5 `requireRole`

- Definição em `src/middlewares/auth.middleware.ts:49-61`. É uma factory variádica `requireRole(...roles)`: sem `req.user` dá `UnauthorizedError`, com role fora da lista dá `ForbiddenError('Insufficient permissions')`.
- Depende de `authenticate` (`auth.middleware.ts:27-47`), que valida o Bearer JWT e popula `req.user = { id, email, role }`.
- Roles possíveis: `'ADMIN' | 'OPERATOR'` (`auth.middleware.ts:9`; enum `UserRole` em `prisma/schema.prisma:11-14`).
- Uso atual: **um único ponto**, `src/modules/users/user.routes.ts:15` (`GET /users/:id` com `requireRole('ADMIN')`), encadeado por rota. Nos outros módulos o padrão é `router.use(authenticate)` no router inteiro.

### 1.6 Error middleware centralizado

- `src/middlewares/error.middleware.ts:14-65`, registrado por último em `src/app.ts:73`. Trata:
  - `AppError` → `{ error: { code, message, details? } }` com `statusCode`.
  - `ZodError` → 400 `VALIDATION_ERROR` com `details: [{ path, message }]`.
  - Prisma `P2002` → 409 `CONFLICT`; `P2025` → 404 `NOT_FOUND`.
  - Resto → `logger.error(...)` e 500 `INTERNAL_SERVER_ERROR`.
- Só loga o caso 500. `AppError` não é logado.
- O middleware `validate` (`src/middlewares/validate.middleware.ts:11-37`) converte `ZodError` em `ValidationError`. Toda falha de schema sai como `VALIDATION_ERROR`, nunca com código de domínio.
- 404 de rota inexistente: handler em `src/app.ts:69-71`.

### 1.7 Logger Pino

- `src/shared/logger/index.ts`: singleton `logger` criado por `createLogger()`.
  - `level: env.LOG_LEVEL` (`fatal|error|warn|info|debug|trace`, default `info`; `src/config/env.ts:6`).
  - `redact` com `[REDACTED]` em `req.headers.authorization`, `req.headers.cookie`, `*.password`, `*.passwordHash`, `*.token`, `*.accessToken` (l.4-11).
  - `base: { service: 'order-management-api', env }` (l.20). O nome do serviço está fixo.
  - Timestamp ISO; `pino-pretty` só em `development`.
- Pontos de chamada: `src/server.ts:10,14,15,25` (`server_started`, `shutdown_initiated`, `http_server_closed`, `bootstrap_failed`), `src/middlewares/request-logger.middleware.ts:14-24` (`http_request` com requestId, method, path, statusCode, durationMs, userId) e `src/middlewares/error.middleware.ts:57-60`.
- Convenção: objeto estruturado como primeiro argumento e mensagem como evento em `snake_case`.
- `pino-http` está em `package.json` mas não é usado (o request logger é próprio).
- O request logger gera ou propaga `X-Request-Id` com `uuid` v4 (`request-logger.middleware.ts:6-8`).

### 1.8 Outros padrões relevantes

- **ESM** (`"type": "module"`), imports com sufixo `.js`, TypeScript 5.6, Node ≥ 20 (`package.json`).
- **Controllers** são classes com handlers em arrow function, `try/catch` e `next(err)` (ex.: `order.controller.ts`).
- **Validação**: schemas Zod em `*.schemas.ts`, aplicados via `validate({ body, query, params })`. Os ids são validados com `z.string().uuid()`.
- **Paginação**: `page`/`pageSize` (máx. 100) no query schema (`order.schemas.ts:23-30`) e resposta `paginated(...)` (`src/shared/http/response.ts`).
- **Modelagem Prisma**: ids `String @id @default(uuid()) @db.Char(36)` em todas as tabelas de domínio. Tabelas em snake_case plural via `@@map`. `createdAt`/`updatedAt` padrão. Já existe campo `Json` (`Customer.address`, `schema.prisma:46`). Valores em centavos (`*Cents Int`).
- **Vínculo User ↔ Customer: não existe.** `User` (`schema.prisma:25-38`) não tem `customerId`, e as roles são só ADMIN/OPERATOR.
- **Scripts** (`package.json`): `dev`, `build`, `start` apontam para `server.ts`. Não existe script de worker.
- **Dependências úteis já presentes**: `uuid` 11, `zod`, `pino`. Não há cliente HTTP nem lib de HMAC. Node 20 oferece `fetch` e `node:crypto` nativos (isto é inferência, não está no código).
- **Body limit** da API: `express.json({ limit: '1mb' })` (`src/app.ts:59`).
- **Testes**: Vitest + Supertest contra banco real.
  - `vitest.config.ts`: `singleFork`, sem paralelismo de arquivos, `setupFiles: tests/setup.ts`.
  - `tests/setup.ts:7-15` limpa as tabelas com `deleteMany` em ordem de FK. Toda tabela nova precisa entrar aí.
  - Factories em `tests/helpers/factories.ts` (`getTestApp`, `bootstrapAuthenticatedUser`, `createTestCustomer`, `createTestProduct`).
  - Cobertura atual: `tests/auth.test.ts` (7 casos) e `tests/orders.test.ts` (8 casos, incluindo transição, 409 e 422).

---

## 2. TRANSCRIÇÃO

### a) Decisões fechadas

| # | Decisão | Origem |
|---|---|---|
| D1 | Webhook **outbound** apenas (só da plataforma para o cliente) | [09:02] Marcos; [09:03] Sofia |
| D2 | Disparo síncrono rejeitado; usar **padrão outbox no MySQL** existente, com insert na mesma transação da mudança de status | [09:06] Diego; fechado em [09:08] Larissa: "Tá decidido então: outbox em MySQL" |
| D3 | Worker lê a outbox por **polling a cada 2s** | [09:09] Diego; [09:10] Marcos; fechado em [09:10] Larissa ("Vamos registrar isso como uma decisão") |
| D4 | Worker como **processo separado** da API, mesmo banco e mesma stack | [09:11] Diego; [09:11] Larissa (entry `src/worker.ts` + `npm run worker`); [09:11] Diego ("Só não pode ser o mesmo processo") |
| D5 | **Single-worker** e ordenação garantida só por `order_id`, sem ordenação global (limitação conhecida) | [09:12] Diego; [09:13] Larissa ("Documentamos como limitação conhecida") |
| D6 | **Retry com backoff exponencial**, 5 tentativas, 1m/5m/30m/2h/12h | [09:15–09:17] Diego; fechado em [09:17] Larissa |
| D7 | **DLQ em tabela separada** `webhook_dead_letter` (payload, motivo, timestamp) | [09:18] Diego; [09:18] Bruno ("Faz sentido"). Sem fechamento explícito da Larissa, mas consta no resumo [09:48] Larissa ("DLQ persistida em tabela separada") |
| D8 | **Replay manual da DLQ** por endpoint admin `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente | [09:18] Diego; [09:19] Larissa ("Anota também") |
| D9 | **HMAC-SHA256 sobre o corpo**, **secret única por endpoint**, **rotação com grace period de 24h** | [09:20–09:21] Sofia; fechado em [09:22] Sofia ("Decidido: …") |
| D10 | **At-least-once** com **`X-Event-Id`** (UUID) para o cliente deduplicar | [09:24–09:25] Diego; fechado em [09:26] Larissa ("Decisão") |
| D11 | Módulo em `src/modules/webhooks` seguindo o padrão controller/service/repository/routes/schemas | [09:27] Bruno; [09:28] Diego |
| D12 | Códigos de erro com prefixo **`WEBHOOK_`** | [09:28] Bruno; [09:29] Larissa |
| D13 | **Reuso máximo**: AppError, Pino, error middleware, padrão de módulos, schemas Zod, códigos de erro | [09:30] Larissa ("Decisão: reuso máximo…") |
| D14 | Worker com **PrismaClient próprio** (mesma `DATABASE_URL`, outra instância) | [09:30] Bruno. Sem objeção, mas também sem fechamento formal: levantado e aceito tacitamente |
| D15 | `customer_id` vai **no body ou no path**, **não vem do JWT** (corrige a fala de Marcos em [09:31]) | [09:32] Larissa; [09:33] Bruno ("Anotado") |
| D16 | **Filtro de eventos aplicado na inserção** da outbox; se nenhum webhook quer aquele status, nada é inserido | [09:34] Bruno; [09:34] Diego ("Concordo") |
| D17 | Replay exige **role ADMIN** usando o **`requireRole`** existente | [09:36] Sofia; fechado em [09:36] Larissa |
| D18 | CRUD de configuração liberado para **qualquer role autenticada** (por enquanto) | [09:36] Marcos; [09:37] Sofia ("Por enquanto sim") |
| D19 | Insert na outbox **dentro da mesma transação** do `changeStatus`; se falhar, rollback | [09:40] Bruno; [09:41] Diego ("Essencial") |
| D20 | Integração por uma **função `publishWebhookEvent(tx, order, fromStatus, toStatus)`** que recebe o tx client, sem injetar o repository inteiro | [09:41] Bruno; [09:41] Diego |
| D21 | Ids da outbox em **UUID** | [09:51] Larissa |
| D22 | Payload **renderizado (snapshot) na inserção** | [09:52] Larissa; [09:52] Diego; [09:52] Bruno ("Decidido") |
| D23 | Prazo de **3 sprints**, com a revisão de segurança da Sofia incluída no fim | [09:46–09:47] Larissa |

### b) Requisitos funcionais

- **RF1**: Notificar o cliente B2B quando o status de um pedido dele muda. [09:00] Marcos
- **RF2**: Cadastro de webhook via `POST`, com `url` e a lista de status desejados. A secret é **gerada pela plataforma e devolvida na criação**. [09:31] Marcos
- **RF3**: `PATCH` para editar, `DELETE` para remover e `GET` para listar os webhooks de um customer. [09:33] Bruno
- **RF4**: Filtro de eventos por endpoint, como lista de status (ex.: só SHIPPED e DELIVERED). [09:33] Marcos
- **RF5**: Endpoint para o cliente **pedir nova secret** (rotação). [09:21] Sofia
- **RF6**: Histórico de entregas `GET /webhooks/:id/deliveries` com os "últimos 100" envios: sucesso/falha, payload, response e tempo de resposta. [09:34] Marcos
- **RF7**: Replay manual de item da DLQ: `POST /admin/webhooks/dead-letter/:id/replay`. [09:18] Diego; [09:35] Diego
- **RF8**: O replay precisa **registrar quem executou**, para auditoria. [09:36] Sofia
- **RF9**: Mover para a DLQ depois de esgotar as tentativas. [09:15] Diego
- **RF10**: Cada envio leva os headers `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` e `Content-Type: application/json`. [09:44] Diego; [09:44–09:45] Sofia/Diego

### c) Requisitos não funcionais / restrições

- **RNF1, latência**: abaixo de 10s conta como "tempo real" para os clientes. [09:02] Marcos. O polling de 2s implica latência mínima de 2s no pior caso, e isso foi aceito. [09:10] Larissa
- **RNF2, atomicidade**: não pode existir status alterado sem evento registrado. [09:40] Bruno
- **RNF3, isolamento de falha**: cliente lento ou fora do ar não pode travar `changeStatus`. [09:04] Bruno
- **RNF4, TLS obrigatório**: só URL `https`; `http` é recusado com erro de validação (schema Zod). [09:23] Sofia
- **RNF5, limite de payload de 64KB**; se passar, gera **erro** (não trunca). [09:23] Sofia; [09:24] Diego; [09:24] Larissa
- **RNF6, timeout HTTP de 10s** no worker; estourou, conta como falha e vai para retry. [09:42] Diego; [09:42] Sofia ("Bom")
- **RNF7, secret por endpoint**, para que o vazamento de uma não comprometa as outras. [09:21] Sofia
- **RNF8, ordering**: só por `order_id` e enquanto houver um único worker. [09:13] Larissa
- **RNF9, infra**: nada de subir infraestrutura nova (time pequeno). [09:07] Diego
- **RNF10, processo**: Sofia revisa a segurança por pelo menos 2 dias úteis antes do deploy (HMAC e geração de secret). [09:46] Sofia; reforçado em [09:49] Sofia
- **RNF11, negócio**: a Atlas quer a entrega até o fim de novembro, sob risco de churn para o concorrente. [09:00] Marcos; [09:45] Marcos

### d) Ganchos com o código existente

- **G1**: A transação de `changeStatus` já atualiza `orders`, insere em `order_status_history` e decrementa `stock_quantity`. [09:04] Bruno; detalhada em [09:40] Bruno
- **G2**: Nova entry-point no mesmo molde de `src/server.ts`: `src/worker.ts` + `npm run worker`. [09:11] Larissa
- **G3**: Mesmo banco e mesmo Prisma (client novo no outro processo). [09:11] Bruno; [09:30] Bruno
- **G4**: Padrão de módulo em `src/modules/*`. [09:27] Bruno
- **G5**: `AppError`, `InsufficientStockError`, `InvalidStatusTransitionError` e os códigos `INSUFFICIENT_STOCK`/`INVALID_STATUS_TRANSITION` como modelo. [09:28] Bruno
- **G6**: Logger Pino e error middleware que "já trata AppError, Zod e Prisma". [09:29] Bruno
- **G7**: Pool de conexão do Prisma. [09:29] Diego
- **G8**: JWT atual é de "usuário operador"; existem "usuários que representam o cliente". [09:32] Bruno; [09:32] Marcos
- **G9**: `requireRole` existente. [09:36] Larissa
- **G10**: Método `changeStatus` no service de orders, com integração via `tx`. [09:40–09:41] Bruno
- **G11**: `GET /orders/:id` como caminho para o cliente buscar detalhes (por isso o payload não leva items). [09:43] Diego
- **G12**: Ids UUID "como no resto do projeto". [09:51] Larissa
- **G13**: Validação de URL https "no schema Zod". [09:23] Sofia

### e) Descartado

| Ideia | Motivo | Origem |
|---|---|---|
| Disparo HTTP síncrono dentro do `changeStatus` | Transação já pesada; cliente lento trava as outras mudanças; não dá para fazer rollback se o cliente cair | [09:04] Bruno; [09:06] Diego |
| Redis Streams / Redis Cluster | Exige infra nova; overengineering para um time pequeno | [09:07] Larissa; [09:07] Diego |
| Trigger de banco para worker reativo | MySQL não tem NOTIFY/LISTEN; trigger não notifica processo externo | [09:09] Bruno (sugeriu); [09:09] Diego |
| Worker no mesmo processo da API | Restart da API derruba o worker | [09:11] Diego |
| Retry indefinido | O evento fica pendurado para sempre se o cliente sumir | [09:15] Diego |
| 3 tentativas | Insuficiente: cobriria só ~30 min, e já houve manutenção de 2h | [09:16] Bruno (sugeriu); [09:16] Diego |
| DLQ como status "failed" na própria outbox | Tabela separada deixa a leitura da outbox mais limpa e serve de evidência para debug | [09:17] Larissa (opção); [09:18] Diego |
| Secret global da plataforma | Se vaza uma, vaza tudo | [09:21] Sofia |
| Truncar payload acima do limite | Um payload desse tamanho indica erro; preferem falhar | [09:23] Sofia |
| Exactly-once | Exige coordenação dos dois lados e fica muito mais complexo | [09:25] Diego |
| `customer_id` vindo do JWT | O JWT é de usuário do sistema, não do cliente | [09:31] Marcos (propôs); [09:32] Larissa |
| Filtrar eventos na hora do envio | Filtrar na inserção economiza linhas | [09:34] Diego (perguntou); [09:34] Bruno |
| Injetar repository inteiro no `OrderService` | Basta uma função pura que recebe `tx` | [09:41] Bruno; [09:41] Diego |
| Enviar `items` no payload | Incha o payload; o cliente consulta `GET /orders/:id` | [09:43] Diego |
| Id auto-incremental na outbox | O projeto inteiro usa UUID | [09:51] Diego (perguntou); [09:51] Larissa |
| Renderizar o payload no envio (guardar só `order_id`) | O evento precisa refletir o estado do momento da mudança | [09:51] Bruno (opção); [09:52] Larissa |

### f) Adiado / fora de escopo nesta fase

- **Arquivamento** das linhas entregues "depois de 30 dias ou assim": fora do escopo. [09:08] Diego
- **Escalar para múltiplos workers** (particionar por `order_id` ou lock pessimista): "problema do futuro". [09:13] Diego
- **Endurecer a autorização do CRUD** de webhooks: "mais pra frente". [09:37] Sofia
- **Email de alerta** quando o webhook falha repetidamente: próxima fase, depois de medir o impacto. [09:37] Larissa; [09:38] Marcos
- **Rate limiting de saída**: ponto em aberto, "observar e decidir depois". [09:39] Diego; [09:39] Larissa
- **Dashboard visual** para o cliente: fora de escopo, é projeto do time de frontend. [09:40] Larissa
- **Documentação no portal do desenvolvedor** (at-least-once, integração via API): responsabilidade do Marcos, fora do código. [09:26] Marcos; [09:40] Marcos

### g) Detalhes técnicos secundários

- Status da outbox: `pendente, processando, falhou, entregue`, com índices em `status` e `created_at`. [09:08] Diego
- O worker lê os pendentes mais antigos **em batch pequeno**, sem tamanho definido. [09:08] Diego; [09:09] Diego
- Nomes de tabela: `webhook_outbox` [09:06] Diego; `webhook_dead_letter` [09:18] Diego.
- Tabela de configuração com `url + secret + customer_id + estado ativo`. [09:21] Bruno; [09:21] Sofia ("Sim")
- Arquivo do processador: `src/modules/webhooks/webhook.worker.ts` **ou** `webhook.processor.ts` (nome não decidido). [09:28] Bruno
- `event_type`: `"order.status_changed"`. [09:43] Diego
- Payload JSON: `event_id`, `event_type`, `timestamp` ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e "campos básicos" como `total_cents`, sem items. [09:43] Diego; [09:44] Bruno ("Bom"). A lista de "campos básicos" não está fechada.
- `X-Event-Id` é gerado **quando o evento entra na outbox**. [09:25] Diego
- `X-Timestamp` é o timestamp **do envio**, para permitir detectar replay attack. [09:44] Diego
- `X-Webhook-Id` é o id do endpoint cadastrado. [09:44] Sofia
- Janela total do backoff: "quase 15 horas". [09:17] Diego
- Estimativa: outbox+DLQ 1 sprint; worker+retry 1 sprint; CRUD+deliveries ½; integração+E2E ½; HMAC/schemas "mais um pouco". [09:46] Larissa
- Script `npm run worker`. [09:11] Larissa

### Ambiguidades e pontos a esclarecer (não decididos)

- **A1, contagem de tentativas**: "5 tentativas" convive com **5 intervalos** de backoff (1m/5m/30m/2h/12h ≈ 14h36). Se as 5 incluem o envio inicial, sobra um intervalo. Se são 5 *retries*, o total de envios é 6. [09:17] Diego / [09:17] Larissa
- **A2, `X-Signature`**: o HMAC é calculado só sobre o corpo [09:22] Sofia, e o `X-Timestamp` não entra na assinatura. Assim a proteção contra replay sugerida em [09:44] Diego não é garantida criptograficamente. Ninguém discutiu isso.
- **A3, formato do header** `X-Signature` (hex/base64, prefixo `sha256=`): sem localização, não foi discutido.
- **A4, "últimos 100" em deliveries**: não ficou claro se é limite fixo ou paginação, nem se existe tabela própria de deliveries (nenhuma foi nomeada). [09:34] Marcos
- **A5, rotação durante o grace period**: com duas secrets válidas, não se definiu se o envio assina com as duas ou só com a nova. [09:21] Sofia
- **A6, criação do pedido**: não se discutiu se a criação (`→ PENDING`) gera evento. As falas falam em "quando o status muda". [09:00] Marcos; [09:40] Bruno
- **A7, `timestamp` do payload**: "ISO 8601" [09:43] Diego, sem dizer se é o momento da mudança (snapshot) ou do envio.
- **A8, inconsistência de tempo**: em [09:27] Larissa diz "Estamos a uma hora de reunião", mas os timestamps marcam 27 min e o cabeçalho fala em ~55 min. Não afeta requisitos.

---

## 3. CRUZAMENTOS

- **D2/D19 (outbox na mesma transação) [09:06] Diego, [09:40] Bruno** se ligam a `src/modules/orders/order.service.ts:131-178`. O `prisma.$transaction(async (tx) => …)` interativo já existe, e o insert entra depois do `tx.orderStatusHistory.create` (l.159-167). A atomicidade vem de graça: qualquer throw dentro do callback faz rollback. *Atenção*: o filtro na inserção (D16) acrescenta uma leitura de webhooks do customer dentro da transação, que tem o timeout default de 5s do Prisma.
- **D20 (`publishWebhookEvent(tx, …)`) [09:41] Bruno** segue o padrão já existente de helpers que recebem `tx: TxClient` (`order.service.ts:24`, `debitStock` l.204, `reserveOrderNumber` l.245). Os dados do snapshot (D22) já estão no `order` carregado em l.132 (`orderNumber`, `customerId`, `totalCents`) e em `from`/`to` (l.138-139).
- **A6 (evento na criação?)**: `OrderService.create` (`order.service.ts:50-124`) grava `PENDING` direto, sem passar por `changeStatus`. Se a criação tiver que notificar, há um **segundo ponto de integração**. Senão, o primeiro evento possível é `PENDING → PAID|CANCELLED`.
- **D16 / RF4 (filtro por lista de status)** usam o enum `OrderStatus` (`prisma/schema.prisma:16-23`), com as transições válidas em `order.status.ts:3-10`. O schema Zod do filtro pode reaproveitar `z.nativeEnum(OrderStatus)` como em `order.schemas.ts:19`.
- **D22 (snapshot do payload)**: o tipo `Json` já é usado (`Customer.address`, `schema.prisma:46`). Ele serve para guardar o payload renderizado em `webhook_outbox`/`webhook_dead_letter`.
- **D21 (UUID) [09:51] Larissa** confere com o padrão `@id @default(uuid()) @db.Char(36)` de todas as tabelas de domínio (`schema.prisma`; exceção: `OrderNumberSequence`, Int). Nomes `webhook_outbox`/`webhook_dead_letter` seguem a convenção `@@map` snake_case.
- **D10 (`X-Event-Id` UUID) [09:25] Diego**: a lib `uuid` já é dependência e é usada em `src/middlewares/request-logger.middleware.ts:2,6`. O id do evento pode ser o próprio PK UUID da outbox. **Não há relação técnica com `order_status_history`** além de ambos nascerem na mesma transação. Um vínculo opcional (FK do evento para o registro de history) não foi discutido.
- **D4/G2 (worker separado, `src/worker.ts`) [09:11] Larissa** pode copiar o molde de `src/server.ts` (bootstrap, SIGINT/SIGTERM, `prisma.$disconnect`, `logger.fatal` em `bootstrap_failed`). Faltam scripts em `package.json`: hoje `dev`/`start` apontam só para `server.ts`.
- **D14/G3 (PrismaClient próprio) [09:30] Bruno**: `src/config/database.ts` exporta um singleton por módulo. Importá-lo no `worker.ts` já cria uma instância nova naquele processo, sem código adicional. `createPrismaClient()` também está disponível.
- **Parâmetros do worker (2s, 10s, backoff, batch)** podem entrar em `src/config/env.ts` (schema Zod do env) para manter o padrão de configuração validada. Isso não foi dito na reunião, é só onde o código acomodaria.
- **D12 (prefixo `WEBHOOK_`) [09:29] Larissa** contrasta com o código, que **não usa prefixo de módulo** (`INSUFFICIENT_STOCK`, `INACTIVE_PRODUCT`, `EMAIL_ALREADY_USED`). O módulo de webhooks inaugura essa convenção. Além disso, `NotFoundError` tem código fixo `NOT_FOUND` (`http-errors.ts:27-31`), então `WEBHOOK_NOT_FOUND` exige subclasse nova (no molde de `InvalidStatusTransitionError`) ou mudança na classe.
- **RNF4 (https via Zod) [09:23] Sofia + código `WEBHOOK_INVALID_URL` [09:28] Bruno**: o `validate` middleware (`validate.middleware.ts:26-31`) converte qualquer `ZodError` em `ValidationError` com código `VALIDATION_ERROR`. Uma URL http rejeitada no schema **sai como `VALIDATION_ERROR`, não como `WEBHOOK_INVALID_URL`**, a menos que a checagem fique no service ou que o middleware mude.
- **G6 / D13 ("error middleware pega sem mudar nada") [09:29] Bruno** vale para a API: `AppError` é serializado em `error.middleware.ts:15-24`. **Não vale para o worker**, que não é Express e precisa tratar e logar os próprios erros.
- **G6 / Pino** (`src/shared/logger/index.ts`):
  - `base.service` está fixo em `'order-management-api'` (l.20), então os logs do worker sairiam com o mesmo nome de serviço se nada mudar.
  - A lista de `redact` (l.4-11) **não cobre `secret`, `X-Signature` nem a secret do webhook**. É relevante para D9 e para o episódio de vazamento citado em [09:22] Diego.
- **D17 (replay só ADMIN com `requireRole`) [09:36] Larissa** usa `requireRole('ADMIN')` (`auth.middleware.ts:49-61`), aplicado por rota como em `user.routes.ts:12-18`. Hoje não existe prefixo `/admin`. O path `/admin/webhooks/...` ficaria sob `/api/v1` (`app.ts:67`) e precisaria de router próprio ou de montagem em `routes/index.ts`.
- **RF8 (auditar quem fez o replay) [09:36] Sofia**: `req.user.id` está disponível depois de `authenticate` (`auth.middleware.ts:42`), e o padrão de registrar autor já existe em `OrderStatusHistory.changedById` (`schema.prisma:122`). O log estruturado pode seguir o `request-logger` (que já loga `userId`).
- **D15/G8 (customer_id no body/path; "usuários que representam o cliente") [09:32] Marcos/Larissa**: no código **não existe vínculo User → Customer** (`schema.prisma:25-38`), e as roles são só `ADMIN|OPERATOR`. Com D18 (qualquer role autenticada), **qualquer usuário autenticado consegue gerenciar webhooks de qualquer customer**. Não há como checar posse hoje. Isso conversa com o adiamento "endurecer depois" [09:37] Sofia.
- **D18 (CRUD para qualquer autenticado)** segue o padrão `router.use(authenticate)` de `order.routes.ts:14`, `customer.routes.ts`, `product.routes.ts`.
- **RF6 (deliveries "últimos 100")**: há helper de paginação (`src/shared/http/response.ts`) e limite `pageSize ≤ 100` já usado em `order.schemas.ts:25`. Isso pode resolver A4 com paginação em vez de limite fixo.
- **RNF5 (64KB)** é independente do `express.json({ limit: '1mb' })` (`app.ts:59`), que limita a *entrada* da API. O limite de 64KB vale para o *payload de saída* e precisa ser checado na renderização ou no envio.
- **D9 (HMAC-SHA256) / RNF6 (timeout 10s)**: não há lib de HMAC nem cliente HTTP em `package.json`. O Node ≥ 20 (`engines`) traz `node:crypto` e `fetch` com `AbortSignal.timeout` (inferência, sem uso atual no código), o que encaixa em "não vamos botar nada novo" [09:29] Bruno.
- **D11 (módulo `src/modules/webhooks`)**: além da pasta, o padrão exige registrar o módulo em `src/app.ts` (`buildControllers`), em `src/routes/index.ts` (tipo `Controllers` + `router.use`) e no `tests/setup.ts` (limpeza das tabelas novas, respeitando as FKs com `customers`).
- **Docs-alvo**: os placeholders `docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md`, `docs/TRACKER.md` e `docs/adrs/README.md` já existem no repositório e são os destinos das próximas etapas.
