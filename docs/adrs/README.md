# Architectural Decision Records

Este diretório armazena os ADRs (Architectural Decision Records) do projeto.
Cada decisão arquitetural relevante deve ser registrada aqui em arquivos individuais,
no formato MADR, nomeados sequencialmente como `ADR-NNN-titulo-em-kebab-case.md`.

## Índice — Sistema de Webhooks de Notificação de Pedidos

| ADR | Título | Status |
|---|---|---|
| [ADR-001](ADR-001-outbox-no-mysql.md) | Padrão Outbox no MySQL para eventos de webhook | Aceito |
| [ADR-002](ADR-002-worker-separado-em-polling.md) | Worker em processo separado com polling de 2 segundos | Aceito |
| [ADR-003](ADR-003-retry-com-backoff-e-dlq.md) | Retry com backoff exponencial, DLQ em tabela separada e replay manual | Aceito |
| [ADR-004](ADR-004-hmac-sha256-com-secret-por-endpoint.md) | Autenticação de origem com HMAC-SHA256 e secret por endpoint | Aceito |
| [ADR-005](ADR-005-at-least-once-com-x-event-id.md) | Garantia de entrega at-least-once com X-Event-Id | Aceito |
| [ADR-006](ADR-006-reuso-dos-padroes-existentes.md) | Reuso dos padrões existentes do projeto no módulo de webhooks | Aceito |
| [ADR-007](ADR-007-autorizacao-dos-endpoints-de-webhook.md) | Autorização dos endpoints: replay ADMIN via `requireRole`; CRUD autenticado | Aceito |
