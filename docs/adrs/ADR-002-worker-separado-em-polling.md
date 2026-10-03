# ADR-002 — Worker em processo separado com polling de 2 segundos

- **Status:** Aceito
- **Data da decisão:** Reunião técnica de quinta-feira, 09:00. A transcrição não registra a data de calendário.
- **Decisores:** Larissa (Tech Lead), Diego (Eng. Sênior, Plataforma), Bruno (Eng. Pleno, Pedidos), Marcos (PM)
- **Decisões do inventário cobertas:** D3, D4, D5, D14

## Contexto

Com a outbox ([ADR-001](ADR-001-outbox-no-mysql.md)), alguém precisa ler os eventos pendentes e fazer as chamadas HTTP. O requisito de latência é ficar abaixo de 10s ([09:02] Marcos; RNF1).

Hoje o projeto tem um único entry-point: `src/server.ts`. Ele faz o bootstrap do Express, o graceful shutdown em SIGINT/SIGTERM e chama `prisma.$disconnect()`. Os scripts de `package.json` (`dev`, `start`) apontam só para ele. O `PrismaClient` é um singleton por processo, exportado por `src/config/database.ts`.

## Decisão

Um **worker que roda como processo Node separado da API**, com entry-point próprio `src/worker.ts` e script `npm run worker`. Ele faz **polling a cada 2 segundos**, lendo os eventos pendentes mais antigos (por `created_at`) em batch pequeno. Roda como **instância única (single-worker)**, o que garante ordem **por `order_id`**, mas não ordem global. Usa o **mesmo banco e a mesma stack**, com um **`PrismaClient` próprio** (mesma `DATABASE_URL`, outra instância).

Origens da decisão:

- D3: [09:09] Diego ("Polling em loop. A cada 2 segundos"); [09:10] Larissa ("Worker em polling, 2s. […] Aceitamos").
- D4: [09:11] Diego ("tem que rodar como processo separado, não dentro da mesma instância da API"); [09:11] Larissa ("criar um src/worker.ts e um script 'npm run worker'").
- D5: [09:12] Diego ("Por enquanto, single-worker e ordering implícita por order_id"); [09:13] Larissa ("Documentamos como limitação conhecida").
- D14: [09:30] Bruno ("PrismaClient é por processo. Mesmo banco, mesma DATABASE_URL, mas instância nova").

## Alternativas Consideradas

1. **Trigger do MySQL para acionar o worker de forma reativa.** Bruno sugeriu ([09:09]) e a ideia foi descartada. "MySQL não tem listener nativo tipo o NOTIFY/LISTEN do Postgres. Trigger […] não notifica processo externo […] fica esquisito" ([09:09] Diego).
2. **Worker dentro do mesmo processo da API.** Descartada. "Senão se a API reinicia, perde o worker" ([09:11] Diego).
3. **Vários workers em paralelo.** Não adotada nesta fase e adiada. Isso quebraria a ordenação por pedido. "Aí dá pra particionar por order_id, ou usar lock pessimista. Mas isso é problema do futuro" ([09:13] Diego).

## Consequências

### Positivas

- **Atende o requisito de latência com folga.** "Polling de 2 segundos atende o requisito de 'abaixo de 10 segundos' tranquilo" ([09:09] Diego; [09:10] Marcos: "2 segundos serve").
- **Ciclo de vida independente.** Reiniciar ou fazer deploy da API não derruba as entregas, e vice-versa (D4).
- **Nenhuma infraestrutura nova.** Usa o mesmo MySQL e o mesmo Prisma. O bootstrap e o shutdown podem seguir o molde de `src/server.ts`.
- **Ordenação por pedido sem lock**, enquanto houver uma instância só. Ela basta para o negócio: "Os clientes nunca pediram garantia de ordering global" ([09:14] Marcos).

### Negativas

- **Latência mínima de até 2s no pior caso**, aceita explicitamente ([09:10] Larissa).
- **Sem escala horizontal.** Subir um segundo worker quebra a ordenação por `order_id` (D5).
- **Ponto único de falha nas entregas.** Se o worker cair, os eventos se acumulam na outbox, sem perda graças ao [ADR-001](ADR-001-outbox-no-mysql.md), mas ninguém os entrega até ele voltar. Isto é inferência a partir de D4 e D5.
- **Mais um processo para operar.** Ele precisa de script, deploy e monitoramento próprios. Nada disso está em `package.json` hoje.
- **O worker fica fora da infraestrutura HTTP.** O error middleware (`src/middlewares/error.middleware.ts`) não cobre o worker, que precisa tratar e logar os próprios erros. Além disso, o logger tem `base.service` fixo em `'order-management-api'` (`src/shared/logger/index.ts:20`). Os dois pontos vêm dos cruzamentos do inventário.
- **Consultas mesmo sem eventos.** O polling consulta o banco a cada 2s mesmo quando não há nada pendente. O índice em `status` e `created_at` ([09:08] Diego) reduz o custo. Isto é inferência.
- **Eventos presos em "processando".** A outbox tem o estado `processando` ([09:08] Diego). Se o worker cair no meio de um batch, eventos podem ficar nesse estado. A reunião não discutiu como recuperá-los (inferência e ponto em aberto).

### Trade-off explícito

Ganha simplicidade operacional, porque é só mais um processo Node no mesmo banco. Perde escala horizontal e garantia de ordenação global, e aceita latência mínima de 2s.

### Pontos em aberto

- O tamanho do "batch pequeno" não foi definido ([09:08] Diego).
- O nome do arquivo de processamento ficou entre `src/modules/webhooks/webhook.worker.ts` e `webhook.processor.ts`, sem decisão ([09:28] Bruno).

## Referências

- `src/server.ts`: molde de entry-point e graceful shutdown.
- `src/config/database.ts`: `createPrismaClient()` e o singleton por processo.
- `package.json`: scripts atuais, sem script de worker.
- Inventário: D3, D4, D5, D14; RNF1, RNF8; seção "g".
