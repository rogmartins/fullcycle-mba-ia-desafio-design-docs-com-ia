# ADR-007 — Autorização dos endpoints de webhook: replay exige ADMIN via `requireRole`; CRUD para qualquer usuário autenticado

- **Status:** Aceito
- **Data da decisão:** Reunião técnica de quinta-feira, 09:00. A transcrição não registra a data de calendário.
- **Decisores:** Larissa (Tech Lead), Sofia (Eng. de Segurança), Marcos (PM), Bruno
- **Decisões do inventário cobertas:** D15, D17, D18

## Contexto

A feature expõe dois tipos de endpoint:

- o CRUD de configuração de webhooks, mais o histórico de entregas e a rotação de secret (RF2–RF6);
- o replay da DLQ (RF7, [ADR-003](ADR-003-retry-com-backoff-e-dlq.md)), que mexe na fila de entrega.

No código atual:

- `authenticate` (`src/middlewares/auth.middleware.ts:27-47`) valida o JWT e popula `req.user = { id, email, role }`.
- `requireRole(...roles)` (`auth.middleware.ts:49-61`) devolve `ForbiddenError` quando a role não está na lista. Hoje é usado em **um único ponto**: `src/modules/users/user.routes.ts:15`.
- Os demais módulos aplicam `router.use(authenticate)` no router inteiro (ex.: `src/modules/orders/order.routes.ts:14`).
- As roles possíveis são só `ADMIN | OPERATOR` (`prisma/schema.prisma:11-14`), e **não existe vínculo entre `User` e `Customer`** (`prisma/schema.prisma:25-38`).

## Decisão

1. **Replay da DLQ** (`POST /admin/webhooks/dead-letter/:id/replay`): exige **role ADMIN**, aplicada com o **`requireRole('ADMIN')` existente**. O endpoint também **loga quem executou o replay**, para auditoria (RF8).
2. **CRUD de configuração de webhooks:** aberto a **qualquer usuário autenticado** (`authenticate`), **por enquanto**.
3. **`customer_id`** vai **no body ou no path** da requisição. Ele **não é derivado do JWT**.

Origens da decisão:

- D17: [09:36] Sofia ("Tem que ser ADMIN sim. Mexer em fila de entrega de notificação não é coisa de operador. E o endpoint de admin tem que logar quem fez o replay"); [09:36] Larissa ("Decidido, role ADMIN obrigatório no replay e a gente reaproveita o requireRole que já existe").
- D18: [09:36] Marcos ("O resto do CRUD […] pode ser qualquer role autenticada?"); [09:37] Sofia ("Por enquanto sim").
- D15: [09:32] Larissa ("o customer_id é passado no body ou no path. Não vem do JWT"); [09:33] Bruno ("Anotado").

## Alternativas Consideradas

1. **Derivar o `customer_id` do JWT.** Marcos propôs ([09:31]: "Customer_id implícito do JWT") e a ideia foi descartada. O JWT atual "é do usuário operador, não do cliente" ([09:32] Bruno), e por isso "Não vem do JWT" ([09:32] Larissa).
2. **Permitir o replay para qualquer role autenticada (inclusive OPERATOR).** Descartada. "Mexer em fila de entrega de notificação não é coisa de operador" ([09:36] Sofia).
3. **Restringir o CRUD de configuração por role desde já.** Não adotada nesta fase. "Por enquanto sim. Mais pra frente a gente pode endurecer" ([09:37] Sofia). O endurecimento está **adiado** e não é decidido aqui.

## Consequências

### Positivas

- **Nenhum mecanismo de autorização novo.** O replay usa `requireRole` e o CRUD segue o padrão `router.use(authenticate)` dos outros módulos (consistente com o [ADR-006](ADR-006-reuso-dos-padroes-existentes.md)).
- **A operação sensível fica restrita ao menor grupo possível.** O replay, que reenvia dados a sistemas externos, só pode ser feito por ADMIN.
- **Rastro de auditoria.** `req.user.id` fica disponível depois de `authenticate` (`auth.middleware.ts:42`). O projeto já tem o precedente de registrar autoria em `OrderStatusHistory.changedById` (`prisma/schema.prisma:122`), e o `request-logger` já loga `userId`.

### Negativas

- **Ninguém é dono de um webhook.** Como não há vínculo `User → Customer` e qualquer autenticado pode fazer o CRUD (D18), **qualquer usuário autenticado consegue criar, editar ou remover webhooks de qualquer customer** e obter a secret na criação ([ADR-004](ADR-004-hmac-sha256-com-secret-por-endpoint.md)). Isto é inferência a partir do código e do inventário. Aceito nesta fase com o endurecimento adiado ([09:37] Sofia).
- **O `customer_id` é informado pelo chamador**, o que reforça o risco acima. A plataforma não confere se o usuário representa aquele cliente.
- **Não existe prefixo `/admin`** na API hoje. Os routers são montados sob `/api/v1` em `src/routes/index.ts`, então o path de replay exige um router novo ou uma montagem dedicada.

### Trade-off explícito

Ganha simplicidade e prazo, porque reutiliza `authenticate` e `requireRole` sem um modelo novo de permissões. Abre mão, nesta fase, do princípio do menor privilégio no CRUD de configuração, já que não há como verificar se o usuário pertence ao customer.

## Referências

- `src/middlewares/auth.middleware.ts:27-61`: `authenticate` e `requireRole`.
- `src/modules/users/user.routes.ts:12-18`: único uso atual de `requireRole('ADMIN')`.
- `src/modules/orders/order.routes.ts:14`: padrão `router.use(authenticate)`.
- `prisma/schema.prisma:11-14, 25-38, 122`
- Inventário: D15, D17, D18; RF2–RF8; seção "f" (endurecimento do CRUD adiado).
