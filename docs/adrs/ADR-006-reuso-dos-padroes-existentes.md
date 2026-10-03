# ADR-006 — Reuso dos padrões existentes do projeto no módulo de webhooks

- **Status:** Aceito
- **Data da decisão:** Reunião técnica de quinta-feira, 09:00. A transcrição não registra a data de calendário.
- **Decisores:** Larissa (Tech Lead), Bruno (Eng. Pleno, Pedidos), Diego (Eng. Sênior, Plataforma)
- **Decisões do inventário cobertas:** D11, D12, D13, D20

## Contexto

O OMS já tem convenções consolidadas, e o time é pequeno ([09:07] Diego). Bruno: "A gente tem um padrão claro na codebase" ([09:27]). As convenções que a feature precisa respeitar, mapeadas no inventário (seção 1):

- **Módulos por domínio** em `src/modules/<dominio>/`, com `*.controller.ts`, `*.service.ts`, `*.repository.ts`, `*.routes.ts` e `*.schemas.ts`. A composição é manual em `src/app.ts` (`buildControllers`, l.26-53), e o registro de rotas fica em `src/routes/index.ts` (tipo `Controllers` + `buildApiRouter`).
- **Hierarquia de erros:** `AppError` em `src/shared/errors/app-error.ts`, mais as subclasses de `src/shared/errors/http-errors.ts`. Entre as de domínio estão `InvalidStatusTransitionError` (`INVALID_STATUS_TRANSITION`) e `InsufficientStockError` (`INSUFFICIENT_STOCK`), todas com códigos em `SCREAMING_SNAKE_CASE`.
- **Error middleware centralizado** em `src/middlewares/error.middleware.ts`. Serializa `AppError`, `ZodError` e os erros Prisma `P2002`/`P2025` no formato `{ error: { code, message, details? } }`.
- **Logger Pino** singleton em `src/shared/logger/index.ts`.
- **Validação com Zod** via `validate({ body, query, params })` (`src/middlewares/validate.middleware.ts`).
- **Funções auxiliares que recebem o client da transação:** `type TxClient = Prisma.TransactionClient` e métodos como `debitStock(tx, …)` e `reserveOrderNumber(tx)` em `src/modules/orders/order.service.ts:24, 204, 245`.

## Decisão

O sistema de webhooks é **mais um módulo igual aos outros**, em `src/modules/webhooks/`, com a mesma estrutura de arquivos. O módulo **reutiliza** o que já existe:

- a hierarquia `AppError`;
- o Pino;
- o error middleware;
- os schemas Zod e o `validate`;
- o padrão de códigos de erro, **com prefixo `WEBHOOK_`** (ex.: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`).

A integração com pedidos é feita por uma **função pura `publishWebhookEvent(tx, order, fromStatus, toStatus)`**, que recebe o client da transação corrente e é chamada por `OrderService.changeStatus`.

Origens da decisão:

- D11: [09:27] Bruno ("Vou propor uma pasta src/modules/webhooks com toda a estrutura"); [09:28] Diego ("Faz").
- D12: [09:28] Bruno ("Códigos tipo WEBHOOK_NOT_FOUND, WEBHOOK_INVALID_URL, WEBHOOK_SECRET_REQUIRED"); [09:29] Larissa ("Prefixo WEBHOOK_ pra tudo do módulo").
- D13: [09:29] Bruno ("o logger, que é Pino […] Não vamos botar nada novo. O middleware de erro centralizado já trata AppError, Zod e Prisma"); [09:30] Larissa ("Decisão: reuso máximo do que já existe").
- D20: [09:41] Bruno ("uma função publishWebhookEvent(tx, order, fromStatus, toStatus) que aceita o tx client da transação atual"); [09:41] Diego ("função pura recebendo o tx. Não precisa injetar repository inteiro").

## Alternativas Consideradas

1. **Injetar um repository de webhooks no `OrderService`.** Descartada. Bruno levantou a opção ("Vai me obrigar a passar um repository do webhook pro OrderService ou uma função") e o grupo escolheu a função ([09:41] Diego: "Não precisa injetar repository inteiro"). Com a função, o construtor `new OrderService(orderRepository, prisma)` em `src/app.ts:43` não muda.
2. **Introduzir bibliotecas ou estruturas novas** (outro logger, outro formato de erro). Descartada de forma genérica: "Não vamos botar nada novo" ([09:29] Bruno).
3. **Códigos de erro sem prefixo de módulo**, como os atuais (`INSUFFICIENT_STOCK`, `INACTIVE_PRODUCT`, `EMAIL_ALREADY_USED`). *Alternativa plausível, inferida do código; não foi discutida como opção.* O grupo fechou o prefixo `WEBHOOK_` ([09:29] Larissa) sem debater o padrão sem prefixo.

## Consequências

### Positivas

- **Consistência e curva de aprendizado baixa.** O módulo segue o mesmo desenho dos outros 5 domínios.
- **Formato de erro uniforme na API, sem mudar o middleware.** O `AppError` com código `WEBHOOK_*` é serializado por `src/middlewares/error.middleware.ts:15-24`.
- **Acoplamento mínimo com pedidos.** A função pura `publishWebhookEvent(tx, …)` segue o padrão `TxClient` já usado em `order.service.ts`.
- **Nenhuma dependência nova para logs e erros.**

### Negativas

- **O prefixo `WEBHOOK_` cria uma convenção nova.** Hoje nenhum código de erro tem prefixo de módulo. Os códigos de pedidos, por exemplo, não começam com `ORDER_`.
- **`NotFoundError` tem código fixo `NOT_FOUND`** (`src/shared/errors/http-errors.ts:27-31`). Para ter `WEBHOOK_NOT_FOUND` é preciso uma subclasse nova, no molde de `InvalidStatusTransitionError`, ou mudar a classe.
- **`WEBHOOK_INVALID_URL` não sai pelo caminho padrão.** O `validate` (`src/middlewares/validate.middleware.ts:26-31`) converte qualquer `ZodError` em `ValidationError` com código `VALIDATION_ERROR`. Uma URL `http` rejeitada no schema sairia como `VALIDATION_ERROR`, a menos que a checagem fique no service.
- **O reuso de "error middleware sem mudar nada" ([09:29] Bruno) vale só para a API.** O worker ([ADR-002](ADR-002-worker-separado-em-polling.md)) não roda no Express e precisa tratar os próprios erros. O logger tem `base.service: 'order-management-api'` fixo (`src/shared/logger/index.ts:20`) e `redact` sem `secret` (l.4-11).
- **Dependência direta de `orders` para `webhooks`.** `order.service.ts` passa a importar uma função do módulo de webhooks. Isto é inferência.
- **Mudanças de registro fora do módulo.** O módulo novo exige entradas em `src/app.ts`, em `src/routes/index.ts` e em `tests/setup.ts`, onde as tabelas novas precisam entrar na limpeza `deleteMany` em ordem de FK.

### Trade-off explícito

Ganha consistência e velocidade de entrega por reaproveitar o que existe. Aceita pequenas fricções onde o padrão atual não cobre o caso novo: códigos `WEBHOOK_*` com classes de código fixo, validação Zod que vira `VALIDATION_ERROR` e um worker fora do Express.

## Referências

- `src/modules/<dominio>/`: padrão de módulo (ex.: `src/modules/orders/`).
- `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts`, `src/shared/errors/index.ts`
- `src/middlewares/error.middleware.ts`, `src/middlewares/validate.middleware.ts`
- `src/shared/logger/index.ts`
- `src/app.ts:26-53`, `src/routes/index.ts`
- `src/modules/orders/order.service.ts:24, 126-179, 204, 245`
- `tests/setup.ts`
- Inventário: D11, D12, D13, D20; seção 3 (cruzamentos).
