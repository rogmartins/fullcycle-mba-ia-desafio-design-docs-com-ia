# ADR-001 — Padrão Outbox no MySQL para eventos de webhook

- **Status:** Aceito
- **Data da decisão:** Reunião técnica de quinta-feira, 09:00. A transcrição não registra a data de calendário.
- **Decisores:** Larissa (Tech Lead), Diego (Eng. Sênior, Plataforma), Bruno (Eng. Pleno, Pedidos)
- **Decisões do inventário cobertas:** D2, D19, D21, D22, mais D16 (relacionada)

## Contexto

Três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) pediram para ser notificados quando o status dos pedidos deles muda. Hoje eles fazem polling em `GET /orders` ([09:00] Marcos; RF1). Para eles, "abaixo de 10 segundos já é tempo real" ([09:02] Marcos; RNF1).

O ponto do código onde o status muda é `OrderService.changeStatus`, em `src/modules/orders/order.service.ts:126-179`. Ele roda numa transação interativa `this.prisma.$transaction(async (tx) => …)` que:

- valida a transição pela máquina de estados (`src/modules/orders/order.status.ts`);
- debita ou repõe estoque;
- atualiza `orders`;
- grava a auditoria em `order_status_history`.

Bruno resumiu: "a transação de mudança de status hoje já é pesada" ([09:04] Bruno).

Restrições que vêm do inventário:

- **RNF2 (atomicidade):** "Não pode ter caso de status mudar e evento não sair" ([09:40] Bruno).
- **RNF3 (isolamento de falha):** um cliente lento ou fora do ar não pode travar `changeStatus` ([09:04] Bruno).
- **RNF9 (sem infra nova):** "a gente é um time pequeno" ([09:07] Diego).

## Decisão

Quando o status de um pedido muda, uma linha é inserida na tabela `webhook_outbox` **dentro da mesma transação** do `changeStatus`. Essa linha guarda o **payload já renderizado (snapshot)** e tem **PK UUID**. Se esse insert falhar, toda a mudança de status sofre rollback. O envio HTTP fica com um worker separado, descrito no [ADR-002](ADR-002-worker-separado-em-polling.md).

Origens da decisão:

- [09:06] Diego propôs o padrão: "dentro da mesma transação SQL que atualiza orders e order_status_history, a gente também insere uma linha numa tabela tipo webhook_outbox".
- [09:08] Larissa fechou: "Tá decidido então: outbox em MySQL".
- D19: [09:40] Bruno ("Se a outbox falhar de inserir, rollback"); [09:41] Diego ("Essencial").
- D21: [09:51] Larissa ("UUID, segue o padrão do resto do projeto").
- D22: [09:52] Larissa ("renderizado já, na hora da inserção"); Diego e Bruno concordaram.
- D16 (relacionada): a linha só é inserida se algum webhook ativo do customer assina aquele status ([09:34] Bruno: "Se nenhum webhook do customer quer aquele status, nem insere"; [09:34] Diego concordou).

## Alternativas Consideradas

1. **Disparar o HTTP de forma síncrona dentro do `changeStatus`.** Descartada. "Se a gente acrescentar um HTTP call no meio disso, qualquer cliente lento vai travar mudança de status pra outros pedidos" ([09:04] Bruno). "Se o cliente tiver fora do ar […] dá rollback na mudança de status? Não dá" ([09:04] Bruno). "Síncrono está fora de questão" ([09:06] Diego).
2. **Fila externa (Redis Streams / Redis Cluster).** Descartada. "A gente acabaria precisando subir mais infra" ([09:07] Larissa). "Subir Redis Cluster pra isso é overengineering. Outbox no MySQL existente resolve" ([09:07] Diego).
3. **Guardar só `order_id` e renderizar o payload no envio.** Descartada. "Se o pedido mudar depois, o evento ainda reflete o estado de quando o status mudou. Senão tem caso esquisito" ([09:52] Larissa).
4. **PK auto-incremental na outbox.** Descartada a favor de UUID, por consistência com o projeto ([09:51] Larissa). Todas as tabelas de domínio em `prisma/schema.prisma` usam `@id @default(uuid()) @db.Char(36)`.
5. **Filtrar os eventos na hora do envio, e não na inserção.** Descartada. Filtrar na inserção "economiza linha na tabela" ([09:34] Bruno).

## Consequências

### Positivas

- **Atomicidade sem coordenação distribuída.** "Se a transação principal commitou, o evento foi registrado, e se ela deu rollback, o evento some junto. Não tem inconsistência possível" ([09:06] Diego). Isso sai de graça porque `changeStatus` já usa uma transação interativa, e qualquer exceção dentro do callback desfaz tudo.
- **Nenhuma infraestrutura nova.** Usa o MySQL e o Prisma que já existem (RNF9).
- **Falha do cliente isolada.** A transação de pedido não faz I/O de rede externo (RNF3).
- **O snapshot congela o estado do pedido no momento da mudança** (D22), mesmo que o pedido seja alterado depois.
- O filtro na inserção (D16) evita linhas inúteis na outbox.

### Negativas

- **A transação de `changeStatus` fica maior.** Ganha uma leitura dos webhooks do customer (para o filtro D16) e um insert. Inferência sobre o código: a transação interativa não define `timeout` e herda o default de 5s do Prisma 5.22.
- **A disponibilidade do `changeStatus` fica acoplada ao insert na outbox** (é intencional, D19). Um defeito no módulo de webhooks, como um snapshot que estoura o limite de 64KB (RNF5), passa a impedir mudanças de status. Isto é inferência a partir de D19 combinado com RNF5.
- **A tabela cresce continuamente.** O arquivamento das linhas entregues "depois de 30 dias ou assim" ficou **fora do escopo** ([09:08] Diego).
- **A latência mínima de entrega fica atrelada ao polling do worker** ([ADR-002](ADR-002-worker-separado-em-polling.md)).
- **O filtro na inserção não é retroativo.** Um webhook cadastrado ou editado depois da mudança de status não recebe eventos passados. Isto é inferência a partir de D16 e não foi discutido.

### Trade-off explícito

Ganha atomicidade e zero infraestrutura nova. Em troca, a transação de pedido fica mais pesada, a mudança de status passa a depender do insert na outbox e a latência fica limitada pelo polling.

### Pontos em aberto (não decididos na reunião)

- **A6:** ninguém discutiu se a **criação** do pedido (`→ PENDING`, em `order.service.ts:50-124`) gera evento. Ela não passa por `changeStatus`. Se tiver que gerar, há um segundo ponto de integração.
- **A7:** o campo `timestamp` do payload é "ISO 8601" ([09:43] Diego), mas não se definiu se é o momento da mudança (o snapshot) ou o do envio.

## Referências

- `src/modules/orders/order.service.ts:126-179`: `changeStatus` e a transação.
- `src/modules/orders/order.status.ts`: máquina de estados.
- `prisma/schema.prisma`: convenção de PK UUID e `@@map` em snake_case.
- Inventário: D2, D16, D19, D21, D22; RNF2, RNF3, RNF5, RNF9; A6, A7.
