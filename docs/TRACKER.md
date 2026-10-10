# Tracker de Rastreabilidade — Sistema de Webhooks de Notificação de Pedidos

> Cada linha liga um item do [PRD](PRD.md), do [RFC](RFC.md), do [FDD](FDD.md) ou de um [ADR](adrs/README.md) à sua origem.
>
> - **Fonte `TRANSCRICAO`:** a Localização traz o timestamp e o falante de `TRANSCRICAO.md`, no formato `[hh:mm] Nome`.
> - **Fonte `CODIGO`:** a Localização traz o caminho real do arquivo no repositório (e a linha, quando o documento de origem a cita).
> - **Fonte `PREMISSA`:** escolha marcada 🔶 no documento de origem, sem respaldo na reunião nem no código. Precisa de validação com a equipe.
>
> Premissas que fecham uma questão levantada na reunião (ex.: FDD P-01 a P-06) usam a fonte da questão original, conforme a regra do tracker. Os IDs reaproveitam a numeração dos próprios documentos (PRD-FR-NN, FDD-CA-NN, FDD P-NN etc.).

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Avisar o cliente B2B automaticamente quando o status de um pedido dele muda | TRANSCRICAO | [09:00] Marcos |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Cadastro de webhook com URL e status desejados; secret gerada pela plataforma e mostrada no cadastro | TRANSCRICAO | [09:31] Marcos |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Editar, remover e listar webhooks de um customer, indicado explicitamente na requisição | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Filtro de eventos por webhook (lista de status, ex.: só SHIPPED e DELIVERED) | TRANSCRICAO | [09:33] Marcos |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Cliente pede nova secret; a anterior vale por mais 24h | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Histórico das últimas 100 entregas: sucesso/falha, payload, resposta e tempo de resposta | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Reenvio automático com intervalos de 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Larissa |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Guardar à parte (DLQ) a notificação que esgotou as tentativas, com payload, motivo e momento | TRANSCRICAO | [09:18] Diego |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | ADMIN reprocessa manualmente notificação guardada, que volta para a fila | TRANSCRICAO | [09:18] Diego |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Registrar quem executou cada reprocessamento (auditoria) | TRANSCRICAO | [09:36] Sofia |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Notificação traz id do evento, assinatura, id do webhook e momento do envio | TRANSCRICAO | [09:44] Diego |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | Notificação enxuta (pedido, status anterior e novo, total), sem itens, com snapshot do momento da mudança | TRANSCRICAO | [09:43] Diego |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Notificação sai em menos de 10s após a mudança de status | TRANSCRICAO | [09:02] Marcos |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Status mudou com webhook interessado implica notificação registrada; rollback descarta a notificação | TRANSCRICAO | [09:40] Bruno |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Cliente lento ou fora do ar não atrasa a mudança de status | TRANSCRICAO | [09:04] Bruno |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | Autenticidade e integridade verificáveis com HMAC-SHA256 | TRANSCRICAO | [09:22] Sofia |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | Secret própria por webhook, para conter vazamentos | TRANSCRICAO | [09:21] Sofia |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | Só endereços https; http recusado no cadastro | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-07 | docs/PRD.md | Requisito Não Funcional | Entrega "pelo menos uma vez"; cliente descarta repetições pelo id do evento | TRANSCRICAO | [09:24] Diego |
| PRD-NFR-08 | docs/PRD.md | Requisito Não Funcional | Ordem garantida só por pedido e com um único processo de envio | TRANSCRICAO | [09:13] Larissa |
| PRD-NFR-09 | docs/PRD.md | Requisito Não Funcional | Timeout de 10s por envio e limite de 64KB por notificação | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-10 | docs/PRD.md | Requisito Não Funcional | Reinício da API não interrompe o envio; parada do envio não perde notificações | TRANSCRICAO | [09:11] Diego |
| PRD-NFR-11 | docs/PRD.md | Requisito Não Funcional | Nenhuma infraestrutura nova | TRANSCRICAO | [09:07] Diego |
| PRD-NFR-12 | docs/PRD.md | Requisito Não Funcional | Revisão de segurança de pelo menos 2 dias úteis antes do deploy | TRANSCRICAO | [09:46] Sofia |
| PRD-OBJ-P1 | docs/PRD.md | Objetivo/Métrica | Latência da notificação abaixo de 10s em operação normal | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-P2 | docs/PRD.md | Objetivo/Métrica | Liberação até o fim de novembro, em 3 sprints, para reter a Atlas | TRANSCRICAO | [09:45] Marcos |
| PRD-OBJ-P3 | docs/PRD.md | Objetivo/Métrica | 0 mudança de status com webhook assinante sem evento registrado | TRANSCRICAO | [09:40] Bruno |
| PRD-OBJ-P4 | docs/PRD.md | Objetivo/Métrica | Reenvio automático cobre ≈ 14h36 de indisponibilidade do cliente | TRANSCRICAO | [09:17] Diego |
| PRD-OBJ-P5 | docs/PRD.md | Objetivo/Métrica | Os 3 clientes que pediram a feature recebendo notificações e deixando o polling | TRANSCRICAO | [09:00] Marcos |
| PRD-FORA-01 | docs/PRD.md | Restrição | Fora de escopo: e-mail de alerta ao cliente (próxima fase) | TRANSCRICAO | [09:37] Larissa |
| PRD-FORA-02 | docs/PRD.md | Restrição | Fora de escopo: rate limiting de saída ("observar e decidir depois") | TRANSCRICAO | [09:39] Larissa |
| PRD-FORA-03 | docs/PRD.md | Restrição | Fora de escopo: dashboard visual (time de frontend) | TRANSCRICAO | [09:40] Larissa |
| PRD-FORA-04 | docs/PRD.md | Restrição | Fora de escopo: arquivamento de notificações entregues | TRANSCRICAO | [09:08] Diego |
| PRD-FORA-05 | docs/PRD.md | Restrição | Fora de escopo: vários workers em paralelo | TRANSCRICAO | [09:13] Diego |
| PRD-FORA-06 | docs/PRD.md | Restrição | Fora de escopo: restringir quem configura webhooks | TRANSCRICAO | [09:37] Sofia |
| PRD-FORA-07 | docs/PRD.md | Restrição | Fora de escopo: webhooks de entrada (fluxo só de saída) | TRANSCRICAO | [09:02] Marcos |
| PRD-FORA-08 | docs/PRD.md | Restrição | Fora de escopo: entrega exatamente uma vez | TRANSCRICAO | [09:25] Diego |
| PRD-FORA-09 | docs/PRD.md | Restrição | Fora de escopo: ordem global entre pedidos | TRANSCRICAO | [09:13] Larissa |
| PRD-FORA-10 | docs/PRD.md | Restrição | Fora de escopo: notificação na criação do pedido (FDD P-05) | TRANSCRICAO | [09:40] Bruno |
| PRD-DEC-01 | docs/PRD.md | Trade-off | At-least-once simplifica a plataforma e passa a deduplicação ao cliente | TRANSCRICAO | [09:25] Diego |
| PRD-DEC-02 | docs/PRD.md | Trade-off | Reenvio automático por ~15h; depois só recuperação manual por ADMIN | TRANSCRICAO | [09:17] Marcos |
| PRD-DEC-03 | docs/PRD.md | Trade-off | Sem infraestrutura nova em troca de latência mínima de ~2s | TRANSCRICAO | [09:10] Larissa |
| PRD-DEC-04 | docs/PRD.md | Trade-off | Ordem só por pedido; basta para o negócio, limita escala futura | TRANSCRICAO | [09:14] Marcos |
| PRD-DEC-05 | docs/PRD.md | Trade-off | Configuração aberta a qualquer autenticado por enquanto; replay só ADMIN | TRANSCRICAO | [09:37] Sofia |
| PRD-DEC-06 | docs/PRD.md | Trade-off | Prazo de 3 sprints com a revisão de segurança no fim | TRANSCRICAO | [09:47] Larissa |
| PRD-DEP-01 | docs/PRD.md | Restrição | Dependência: agendar a revisão de segurança da Sofia antes do deploy | TRANSCRICAO | [09:49] Sofia |
| PRD-DEP-02 | docs/PRD.md | Restrição | Dependência: documentação no portal do desenvolvedor (Marcos) | TRANSCRICAO | [09:40] Marcos |
| PRD-DEP-03 | docs/PRD.md | Restrição | Dependência: Marcos confirma prazo e comunica os clientes | TRANSCRICAO | [09:49] Marcos |
| PRD-DEP-04 | docs/PRD.md | Restrição | Dependência: sessão de revisão do design com Bruno e Diego antes de codar | TRANSCRICAO | [09:50] Larissa |
| PRD-RISCO-01 | docs/PRD.md | Risco | R1: atraso na entrega para a Atlas (prazo apertado, revisão obrigatória, questões abertas) | TRANSCRICAO | [09:45] Marcos |
| PRD-RISCO-02 | docs/PRD.md | Risco | R2: cliente não deduplica e reclama ao suporte | TRANSCRICAO | [09:25] Sofia |
| PRD-RISCO-03 | docs/PRD.md | Risco | R3: notificações de um pedido fora de ordem após reenvio | TRANSCRICAO | [09:12] Diego |
| PRD-RISCO-04 | docs/PRD.md | Risco | R4: qualquer autenticado configura webhooks de qualquer customer | TRANSCRICAO | [09:37] Sofia |
| PRD-RISCO-05 | docs/PRD.md | Risco | R5: falha silenciosa após ~15h, sem e-mail de alerta | TRANSCRICAO | [09:37] Larissa |
| PRD-RISCO-06 | docs/PRD.md | Risco | R6: processo de envio parado interrompe as notificações | TRANSCRICAO | [09:11] Diego |
| PRD-AC-01 | docs/PRD.md | Critério de Aceite | Cliente recebe notificação assinada em menos de 10s após o despacho | TRANSCRICAO | [09:02] Marcos |
| PRD-AC-02 | docs/PRD.md | Critério de Aceite | Status não assinado não gera notificação | TRANSCRICAO | [09:34] Bruno |
| PRD-AC-03 | docs/PRD.md | Critério de Aceite | Secret aparece só no cadastro e na troca | TRANSCRICAO | [09:31] Marcos |
| PRD-AC-04 | docs/PRD.md | Critério de Aceite | Cadastro de endereço http é recusado | TRANSCRICAO | [09:23] Sofia |
| PRD-AC-05 | docs/PRD.md | Critério de Aceite | Cliente verifica a assinatura com biblioteca padrão de HMAC-SHA256 | TRANSCRICAO | [09:20] Sofia |
| PRD-AC-06 | docs/PRD.md | Critério de Aceite | Secret antiga aceita por 24h após a troca; depois só a nova | TRANSCRICAO | [09:21] Sofia |
| PRD-AC-07 | docs/PRD.md | Critério de Aceite | Endpoint fora por 2h recebe as notificações ao voltar, sem ação da operação | TRANSCRICAO | [09:16] Diego |
| PRD-AC-08 | docs/PRD.md | Critério de Aceite | ADMIN reprocessa com o mesmo id de evento; OPERATOR não consegue | TRANSCRICAO | [09:36] Sofia |
| PRD-AC-09 | docs/PRD.md | Critério de Aceite | Reprocessamento registra o usuário que o executou | TRANSCRICAO | [09:36] Sofia |
| PRD-AC-10 | docs/PRD.md | Critério de Aceite | Histórico mostra sucesso, conteúdo, resposta e tempo de cada envio | TRANSCRICAO | [09:34] Marcos |
| PRD-AC-11 | docs/PRD.md | Critério de Aceite | Mudança de status funciona com endpoint do cliente lento ou fora | TRANSCRICAO | [09:04] Bruno |
| PRD-AC-12 | docs/PRD.md | Critério de Aceite | Documentação publicada no portal, com destaque para duplicatas | TRANSCRICAO | [09:26] Marcos |
| PRD-AC-13 | docs/PRD.md | Critério de Aceite | Revisão de segurança da Sofia aprovada antes do deploy | TRANSCRICAO | [09:46] Sofia |
| PRD-P-01 | docs/PRD.md | Premissa | Meta de ≥ 95% dos eventos abaixo de 10s | PREMISSA | sem origem — ver PRD §13 |
| PRD-P-02 | docs/PRD.md | Premissa | Sucesso de adoção = 3 de 3 clientes-piloto em produção | PREMISSA | sem origem — ver PRD §13 |
| PRD-P-03 | docs/PRD.md | Premissa | Probabilidade e impacto dos riscos são avaliação qualitativa do PRD | PREMISSA | sem origem — ver PRD §13 |
| PRD-P-04 | docs/PRD.md | Premissa | Piloto com onboarding assistido e ensaios de falha | PREMISSA | sem origem — ver PRD §13 |
| RFC-ALT-01 | docs/RFC.md | Alternativa Descartada | Disparo HTTP síncrono no changeStatus: cliente lento travaria outros pedidos | TRANSCRICAO | [09:06] Diego |
| RFC-ALT-02 | docs/RFC.md | Alternativa Descartada | Redis Streams/Cluster: infraestrutura nova, overengineering | TRANSCRICAO | [09:07] Diego |
| RFC-ALT-03 | docs/RFC.md | Alternativa Descartada | Exactly-once: exigiria coordenação dos dois lados | TRANSCRICAO | [09:25] Diego |
| RFC-ALT-04 | docs/RFC.md | Alternativa Descartada | Retry indefinido ou só 3 tentativas | TRANSCRICAO | [09:16] Diego |
| RFC-ALT-05 | docs/RFC.md | Alternativa Descartada | Trigger de banco para acordar o worker: MySQL não tem NOTIFY/LISTEN | TRANSCRICAO | [09:09] Diego |
| RFC-ABERTO-01 | docs/RFC.md | Questão em Aberto | Contagem de tentativas: 5 ou 6 envios? | TRANSCRICAO | [09:17] Larissa |
| RFC-ABERTO-02 | docs/RFC.md | Questão em Aberto | X-Timestamp fora do HMAC enfraquece a defesa contra replay | TRANSCRICAO | [09:44] Diego |
| RFC-ABERTO-03 | docs/RFC.md | Questão em Aberto | Assinatura durante o grace period: uma ou duas secrets? | TRANSCRICAO | [09:21] Sofia |
| RFC-ABERTO-04 | docs/RFC.md | Questão em Aberto | X-Event-Id no replay da DLQ: mesmo id ou novo? | TRANSCRICAO | [09:18] Diego |
| RFC-ABERTO-05 | docs/RFC.md | Questão em Aberto | A criação do pedido gera evento? | TRANSCRICAO | [09:40] Bruno |
| RFC-ABERTO-06 | docs/RFC.md | Questão em Aberto | Estado "falhou" da outbox versus DLQ separada (nota de consistência §7) | TRANSCRICAO | [09:08] Diego |
| RFC-RISCO-01 | docs/RFC.md | Risco | changeStatus ganha leitura e insert; transação usa timeout default do Prisma | CODIGO | src/modules/orders/order.service.ts |
| RFC-RISCO-02 | docs/RFC.md | Risco | Worker é novo processo a implantar e monitorar | TRANSCRICAO | [09:11] Diego |
| RFC-RISCO-03 | docs/RFC.md | Risco | Lista de redact do logger não cobre a secret | CODIGO | src/shared/logger/index.ts |
| RFC-RISCO-04 | docs/RFC.md | Risco | 3 sprints contra o prazo de fim de novembro | TRANSCRICAO | [09:46] Larissa |
| FDD-OBJ-01 | docs/FDD.md | Objetivo/Métrica | 1º envio em menos de 10s após o commit do changeStatus | TRANSCRICAO | [09:02] Marcos |
| FDD-OBJ-02 | docs/FDD.md | Objetivo/Métrica | Falha no insert da outbox faz rollback da mudança de status | TRANSCRICAO | [09:40] Bruno |
| FDD-OBJ-03 | docs/FDD.md | Objetivo/Métrica | changeStatus sem I/O de rede | TRANSCRICAO | [09:04] Bruno |
| FDD-OBJ-04 | docs/FDD.md | Objetivo/Métrica | Nenhuma infraestrutura nova | TRANSCRICAO | [09:07] Diego |
| FDD-OBJ-05 | docs/FDD.md | Objetivo/Métrica | Timeout HTTP de 10s; estouro conta como falha | TRANSCRICAO | [09:42] Diego |
| FDD-OBJ-06 | docs/FDD.md | Objetivo/Métrica | Corpo enviado até 64KB; acima disso, erro | TRANSCRICAO | [09:24] Larissa |
| FDD-OBJ-07 | docs/FDD.md | Objetivo/Métrica | Só URLs https | TRANSCRICAO | [09:23] Sofia |
| FDD-OBJ-08 | docs/FDD.md | Objetivo/Métrica | Backoff 1m/5m/30m/2h/12h e DLQ ao esgotar | TRANSCRICAO | [09:17] Larissa |
| FDD-OBJ-09 | docs/FDD.md | Objetivo/Métrica | Toda requisição leva X-Signature e X-Event-Id | TRANSCRICAO | [09:22] Sofia |
| FDD-OBJ-10 | docs/FDD.md | Objetivo/Métrica | Testes existentes continuam passando sem alteração | CODIGO | tests/orders.test.ts |
| FDD-FLUXO-01 | docs/FDD.md | Decisão | publishWebhookEvent(tx, …) chamado após tx.orderStatusHistory.create no changeStatus | CODIGO | src/modules/orders/order.service.ts:159-167 |
| FDD-FLUXO-02 | docs/FDD.md | Decisão | Worker em polling de 2s reivindica batch e envia | TRANSCRICAO | [09:09] Diego |
| FDD-FLUXO-03 | docs/FDD.md | Decisão | Fluxo de retry com RETRY_BACKOFF_MS e status FAILED | TRANSCRICAO | [09:17] Diego |
| FDD-FLUXO-04 | docs/FDD.md | Decisão | Movimentação para a DLQ e replay manual devolvendo à outbox | TRANSCRICAO | [09:18] Diego |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato de API | POST /api/v1/webhooks: cadastra webhook e devolve a secret | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato de API | GET /api/v1/webhooks?customerId=: lista webhooks de um customer | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato de API | GET /api/v1/webhooks/:id: detalha um webhook (P-31) | PREMISSA | sem origem — ver FDD §13 (P-31) |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato de API | PATCH /api/v1/webhooks/:id: edita url, events, active | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato de API | DELETE /api/v1/webhooks/:id: remove webhook | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato de API | GET /api/v1/webhooks/:id/deliveries: histórico de entregas | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato de API | POST /api/v1/webhooks/:id/secret/rotate: rotaciona a secret | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-08 | docs/FDD.md | Contrato de API | POST /api/v1/admin/webhooks/dead-letter/:id/replay: replay de DLQ (ADMIN) | TRANSCRICAO | [09:35] Diego |
| FDD-CONTRATO-09 | docs/FDD.md | Contrato de API | GET /api/v1/admin/webhooks/dead-letter: lista a DLQ (P-27) | PREMISSA | sem origem — ver FDD §13 (P-27) |
| FDD-ERRO-01 | docs/FDD.md | Código de Erro | WEBHOOK_NOT_FOUND (404) | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-02 | docs/FDD.md | Código de Erro | WEBHOOK_CUSTOMER_NOT_FOUND (404): customerId informado não existe | TRANSCRICAO | [09:32] Larissa |
| FDD-ERRO-03 | docs/FDD.md | Código de Erro | WEBHOOK_DEAD_LETTER_NOT_FOUND (404): id de DLQ inexistente no replay | TRANSCRICAO | [09:18] Diego |
| FDD-ERRO-04 | docs/FDD.md | Código de Erro | WEBHOOK_INVALID_URL (400): URL não https | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-05 | docs/FDD.md | Código de Erro | VALIDATION_ERROR (400) reusado do middleware validate | CODIGO | src/middlewares/validate.middleware.ts:26-31 |
| FDD-ERRO-06 | docs/FDD.md | Código de Erro | WEBHOOK_PAYLOAD_TOO_LARGE (422 no replay; DLQ no worker) | TRANSCRICAO | [09:24] Larissa |
| FDD-ERRO-07 | docs/FDD.md | Código de Erro | WEBHOOK_SECRET_REQUIRED (interno do worker) | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-08 | docs/FDD.md | Código de Erro | WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED (409), decorrente do replay idempotente | PREMISSA | sem origem — ver FDD §13 (P-04) |
| FDD-ERRO-09 | docs/FDD.md | Código de Erro | WEBHOOK_INACTIVE (409): replay para endpoint removido ou desativado | PREMISSA | sem origem — ver FDD §13 (P-12) |
| FDD-ERRO-10 | docs/FDD.md | Código de Erro | UNAUTHORIZED (401) reusado de authenticate | CODIGO | src/middlewares/auth.middleware.ts:27-47 |
| FDD-ERRO-11 | docs/FDD.md | Código de Erro | FORBIDDEN (403) via requireRole para não ADMIN | TRANSCRICAO | [09:36] Sofia |
| FDD-CA-01 | docs/FDD.md | Critério de Aceite | Status assinado cria linha PENDING na outbox por endpoint, mesmo eventId | TRANSCRICAO | [09:06] Diego |
| FDD-CA-02 | docs/FDD.md | Critério de Aceite | Sem webhook assinante, nenhuma linha criada | TRANSCRICAO | [09:34] Bruno |
| FDD-CA-03 | docs/FDD.md | Critério de Aceite | Falha forçada no insert não altera status, history nem estoque | TRANSCRICAO | [09:40] Bruno |
| FDD-CA-04 | docs/FDD.md | Critério de Aceite | Payload é snapshot do momento da mudança | TRANSCRICAO | [09:52] Larissa |
| FDD-CA-05 | docs/FDD.md | Critério de Aceite | Headers enviados e X-Signature confere com HMAC-SHA256 | TRANSCRICAO | [09:44] Diego |
| FDD-CA-06 | docs/FDD.md | Critério de Aceite | DELIVERED só com 2xx; timeout, rede ou não 2xx viram FAILED | TRANSCRICAO | [09:42] Diego |
| FDD-CA-07 | docs/FDD.md | Critério de Aceite | Após o 6º envio falho, linha vai para webhook_dead_letter | TRANSCRICAO | [09:17] Larissa |
| FDD-CA-08 | docs/FDD.md | Critério de Aceite | Replay exige ADMIN; OPERATOR recebe 403; mesmo eventId | TRANSCRICAO | [09:36] Larissa |
| FDD-CA-09 | docs/FDD.md | Critério de Aceite | Segundo replay da mesma DLQ devolve 409 | PREMISSA | sem origem — ver FDD §13 (P-04) |
| FDD-CA-10 | docs/FDD.md | Critério de Aceite | Replay grava replayedById e loga userId | TRANSCRICAO | [09:36] Sofia |
| FDD-CA-11 | docs/FDD.md | Critério de Aceite | URL http devolve 400 WEBHOOK_INVALID_URL | TRANSCRICAO | [09:23] Sofia |
| FDD-CA-12 | docs/FDD.md | Critério de Aceite | Secret só nas respostas de criação e rotação; mascarada nos logs | TRANSCRICAO | [09:31] Marcos |
| FDD-CA-13 | docs/FDD.md | Critério de Aceite | Durante 24h após rotação, X-Signature traz as duas assinaturas | TRANSCRICAO | [09:21] Sofia |
| FDD-CA-14 | docs/FDD.md | Critério de Aceite | Corpo acima de 65.536 bytes não é enviado | TRANSCRICAO | [09:24] Larissa |
| FDD-CA-15 | docs/FDD.md | Critério de Aceite | Linhas PROCESSING voltam a PENDING no boot do worker | PREMISSA | sem origem — ver FDD §13 (P-17) |
| FDD-CA-16 | docs/FDD.md | Critério de Aceite | Eventos do mesmo pedido e endpoint em ordem de created_at sem falhas | TRANSCRICAO | [09:12] Diego |
| FDD-CA-17 | docs/FDD.md | Critério de Aceite | Testes existentes de orders e auth não quebram | CODIGO | tests/auth.test.ts |
| FDD-CA-18 | docs/FDD.md | Critério de Aceite | npm run worker sobe processo separado com shutdown gracioso | TRANSCRICAO | [09:11] Larissa |
| FDD-RISCO-01 | docs/FDD.md | Risco | Ordenação quebrada por retry dentro do mesmo pedido | TRANSCRICAO | [09:12] Diego |
| FDD-RISCO-02 | docs/FDD.md | Risco | Perda da WEBHOOK_SECRET_ENCRYPTION_KEY torna secrets irrecuperáveis | PREMISSA | sem origem — ver FDD §13 (P-11) |
| FDD-RISCO-03 | docs/FDD.md | Risco | Transação de changeStatus mais lenta | TRANSCRICAO | [09:04] Bruno |
| FDD-RISCO-04 | docs/FDD.md | Risco | Bytes assinados diferentes dos enviados invalidam a assinatura | TRANSCRICAO | [09:22] Sofia |
| FDD-RISCO-05 | docs/FDD.md | Risco | SSRF: usuário autenticado cadastra URL da rede interna (P-32) | PREMISSA | sem origem — ver FDD §13 (P-32) |
| FDD-RISCO-06 | docs/FDD.md | Risco | Endpoint lento (até 10s) atrasando o batch | TRANSCRICAO | [09:42] Diego |
| FDD-RISCO-07 | docs/FDD.md | Risco | Crescimento das tabelas sem arquivamento | TRANSCRICAO | [09:08] Diego |
| FDD-RISCO-08 | docs/FDD.md | Risco | Duplicata após crash do worker | TRANSCRICAO | [09:24] Diego |
| FDD-RISCO-09 | docs/FDD.md | Risco | tests/setup.ts sem limpar as tabelas novas quebra FKs | CODIGO | tests/setup.ts |
| FDD-INTEG-01 | docs/FDD.md | Decisão | Classes de erro WEBHOOK_* novas no molde das existentes | CODIGO | src/shared/errors/http-errors.ts |
| FDD-INTEG-02 | docs/FDD.md | Decisão | Logger ganha redact da secret e service configurável para o worker | CODIGO | src/shared/logger/index.ts |
| FDD-INTEG-03 | docs/FDD.md | Decisão | Registro dos routers /webhooks e /admin/webhooks | CODIGO | src/routes/index.ts |
| FDD-INTEG-04 | docs/FDD.md | Decisão | buildControllers instancia repository, service e controller de webhooks | CODIGO | src/app.ts |
| FDD-INTEG-05 | docs/FDD.md | Decisão | envSchema ganha as 4 variáveis WEBHOOK_* | CODIGO | src/config/env.ts |
| FDD-INTEG-06 | docs/FDD.md | Decisão | 4 modelos novos e enum WebhookOutboxStatus | CODIGO | prisma/schema.prisma |
| FDD-INTEG-07 | docs/FDD.md | Decisão | Scripts worker e worker:dev, sem dependência nova | CODIGO | package.json |
| FDD-INTEG-08 | docs/FDD.md | Decisão | Worker reusa o singleton prisma (instância própria por processo) | CODIGO | src/config/database.ts |
| FDD-INTEG-09 | docs/FDD.md | Decisão | src/worker.ts replica bootstrap e graceful shutdown do server | CODIGO | src/server.ts |
| FDD-INTEG-10 | docs/FDD.md | Decisão | Reuso de authenticate e requireRole('ADMIN') sem alteração | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-P-01 | docs/FDD.md | Premissa | "5 tentativas" = 5 retentativas, até 6 envios (14h36) | TRANSCRICAO | [09:17] Diego |
| FDD-P-02 | docs/FDD.md | Premissa | HMAC só sobre o corpo; X-Timestamp informativo, não assinado | TRANSCRICAO | [09:22] Sofia |
| FDD-P-03 | docs/FDD.md | Premissa | X-Signature sha256=<hex>; duas assinaturas no grace period, nova primeiro | TRANSCRICAO | [09:21] Sofia |
| FDD-P-04 | docs/FDD.md | Premissa | Replay mantém o eventId original e zera tentativas | TRANSCRICAO | [09:25] Diego |
| FDD-P-05 | docs/FDD.md | Premissa | Criação do pedido não gera evento; PENDING fora de events | TRANSCRICAO | [09:40] Bruno |
| FDD-P-06 | docs/FDD.md | Premissa | FAILED é estado transitório; falha permanente vai para a DLQ | TRANSCRICAO | [09:08] Diego |
| FDD-P-07 | docs/FDD.md | Premissa | timestamp do payload = mudança; X-Timestamp = envio | TRANSCRICAO | [09:43] Diego |
| FDD-P-08 | docs/FDD.md | Premissa | Limite de 64KB checado no worker e no replay, não no changeStatus | PREMISSA | sem origem — ver FDD §13 |
| FDD-P-09 | docs/FDD.md | Premissa | Fan-out: uma linha de outbox por endpoint, mesmo eventId | PREMISSA | sem origem — ver FDD §13 |
| FDD-P-10 | docs/FDD.md | Premissa | Tabelas webhook_endpoints e webhook_deliveries; histórico paginado até 100 | TRANSCRICAO | [09:34] Marcos |
| FDD-P-11 | docs/FDD.md | Premissa | Secret whsec_ + 32 bytes, cifrada com AES-256-GCM | PREMISSA | sem origem — ver FDD §13 |
| FDD-P-12 | docs/FDD.md | Premissa | DELETE é soft delete; eventos de endpoint inativo são descartados | PREMISSA | sem origem — ver FDD §13 |
| FDD-P-13 | docs/FDD.md | Premissa | order_id sem FK na outbox e na DLQ | PREMISSA | sem origem — ver FDD §13 |
| FDD-P-14 | docs/FDD.md | Premissa | "Campos básicos" do payload = só total_cents | TRANSCRICAO | [09:43] Diego |
| FDD-P-15 | docs/FDD.md | Premissa | Nenhuma dependência npm nova (node:crypto e fetch nativos) | PREMISSA | sem origem — ver FDD §13 |
| FDD-P-16 | docs/FDD.md | Premissa | Arquivo de processamento = webhook.processor.ts | TRANSCRICAO | [09:28] Bruno |
| FDD-P-17 | docs/FDD.md | Premissa | Boot do worker devolve PROCESSING para PENDING sem contar tentativa | PREMISSA | sem origem — ver FDD §13 |
| FDD-P-18 | docs/FDD.md | Premissa | Loop com setTimeout reagendado, sem sobreposição | PREMISSA | sem origem — ver FDD §13 |
| FDD-P-19 | docs/FDD.md | Premissa | WEBHOOK_BATCH_SIZE = 10 ("batch pequeno") | TRANSCRICAO | [09:08] Diego |
| FDD-P-20 | docs/FDD.md | Premissa | Paralelismo entre grupos de orderId, ordem dentro do grupo | PREMISSA | sem origem — ver FDD §13 |
| FDD-P-21 | docs/FDD.md | Premissa | User-Agent OMS-Webhooks/1.0 | PREMISSA | sem origem — ver FDD §13 |
| FDD-P-22 | docs/FDD.md | Premissa | redirect manual: 3xx conta como falha | PREMISSA | sem origem — ver FDD §13 |
| FDD-P-23 | docs/FDD.md | Premissa | Todo status fora de 2xx, inclusive 4xx, é retentado | PREMISSA | sem origem — ver FDD §13 |
| FDD-P-24 | docs/FDD.md | Premissa | Na listagem, customerId vai na query string | TRANSCRICAO | [09:32] Larissa |
| FDD-P-25 | docs/FDD.md | Premissa | Nova rotação no grace period substitui a secret anterior | PREMISSA | sem origem — ver FDD §13 |
| FDD-P-26 | docs/FDD.md | Premissa | Replay responde 202 Accepted | PREMISSA | sem origem — ver FDD §13 |
| FDD-P-27 | docs/FDD.md | Premissa | Endpoint GET /admin/webhooks/dead-letter para listar a DLQ | PREMISSA | sem origem — ver FDD §13 |
| FDD-P-28 | docs/FDD.md | Premissa | Checagem de https no service para emitir WEBHOOK_INVALID_URL | TRANSCRICAO | [09:23] Sofia |
| FDD-P-29 | docs/FDD.md | Premissa | Métricas derivadas de logs e SQL, sem biblioteca nova | PREMISSA | sem origem — ver FDD §13 |
| FDD-P-30 | docs/FDD.md | Premissa | requestId não persistido na outbox | PREMISSA | sem origem — ver FDD §13 |
| FDD-P-31 | docs/FDD.md | Premissa | Endpoint GET /webhooks/:id para detalhar webhook, não discutido na reunião | PREMISSA | sem origem — ver FDD §13 |
| FDD-P-32 | docs/FDD.md | Premissa | SSRF por URL interna é risco levantado pelo FDD; mitigação na revisão da Sofia | PREMISSA | sem origem — ver FDD §13 |
| ADR-001-DECISAO | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Outbox no MySQL inserida na mesma transação do changeStatus, snapshot e PK UUID | TRANSCRICAO | [09:08] Larissa |
| ADR-001-ALT-01 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa Descartada | Disparo HTTP síncrono no changeStatus | TRANSCRICAO | [09:04] Bruno |
| ADR-001-ALT-02 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa Descartada | Fila externa Redis Streams/Cluster | TRANSCRICAO | [09:07] Diego |
| ADR-001-ALT-03 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa Descartada | Guardar só order_id e renderizar no envio | TRANSCRICAO | [09:52] Larissa |
| ADR-001-ALT-04 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa Descartada | PK auto-incremental na outbox | TRANSCRICAO | [09:51] Larissa |
| ADR-001-ALT-05 | docs/adrs/ADR-001-outbox-no-mysql.md | Alternativa Descartada | Filtrar eventos no envio em vez da inserção | TRANSCRICAO | [09:34] Bruno |
| ADR-001-CONSEQ-01 | docs/adrs/ADR-001-outbox-no-mysql.md | Trade-off | Atomicidade sem coordenação distribuída | TRANSCRICAO | [09:06] Diego |
| ADR-001-CONSEQ-02 | docs/adrs/ADR-001-outbox-no-mysql.md | Trade-off | Transação de changeStatus fica maior (timeout default do Prisma) | CODIGO | src/modules/orders/order.service.ts:126-179 |
| ADR-001-CONSEQ-03 | docs/adrs/ADR-001-outbox-no-mysql.md | Trade-off | Tabela cresce sem arquivamento nesta fase | TRANSCRICAO | [09:08] Diego |
| ADR-001-CONSEQ-04 | docs/adrs/ADR-001-outbox-no-mysql.md | Trade-off | Nenhuma infraestrutura nova | TRANSCRICAO | [09:07] Diego |
| ADR-001-CONSEQ-05 | docs/adrs/ADR-001-outbox-no-mysql.md | Trade-off | Filtro na inserção não é retroativo para webhooks novos | TRANSCRICAO | [09:34] Bruno |
| ADR-001-CONSEQ-06 | docs/adrs/ADR-001-outbox-no-mysql.md | Trade-off | Snapshot congela o estado do pedido no momento da mudança | TRANSCRICAO | [09:52] Larissa |
| ADR-002-DECISAO | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | Worker em processo separado, polling de 2s, single-worker, PrismaClient próprio | TRANSCRICAO | [09:10] Larissa |
| ADR-002-ALT-01 | docs/adrs/ADR-002-worker-separado-em-polling.md | Alternativa Descartada | Trigger do MySQL para acionar o worker | TRANSCRICAO | [09:09] Diego |
| ADR-002-ALT-02 | docs/adrs/ADR-002-worker-separado-em-polling.md | Alternativa Descartada | Worker no mesmo processo da API | TRANSCRICAO | [09:11] Diego |
| ADR-002-ALT-03 | docs/adrs/ADR-002-worker-separado-em-polling.md | Alternativa Descartada | Vários workers em paralelo (adiado) | TRANSCRICAO | [09:13] Diego |
| ADR-002-CONSEQ-01 | docs/adrs/ADR-002-worker-separado-em-polling.md | Trade-off | Latência mínima de até 2s, aceita | TRANSCRICAO | [09:10] Larissa |
| ADR-002-CONSEQ-02 | docs/adrs/ADR-002-worker-separado-em-polling.md | Trade-off | Sem escala horizontal sem perder a ordenação por pedido | TRANSCRICAO | [09:12] Diego |
| ADR-002-CONSEQ-03 | docs/adrs/ADR-002-worker-separado-em-polling.md | Trade-off | Error middleware não cobre o worker | CODIGO | src/middlewares/error.middleware.ts |
| ADR-002-CONSEQ-04 | docs/adrs/ADR-002-worker-separado-em-polling.md | Trade-off | Ciclo de vida independente da API | TRANSCRICAO | [09:11] Diego |
| ADR-002-CONSEQ-05 | docs/adrs/ADR-002-worker-separado-em-polling.md | Trade-off | Ordenação por pedido sem lock basta para o negócio | TRANSCRICAO | [09:14] Marcos |
| ADR-002-CONSEQ-06 | docs/adrs/ADR-002-worker-separado-em-polling.md | Trade-off | Logger com base.service fixo em order-management-api | CODIGO | src/shared/logger/index.ts:20 |
| ADR-002-CONSEQ-07 | docs/adrs/ADR-002-worker-separado-em-polling.md | Trade-off | Eventos podem ficar presos em "processando" se o worker cair | TRANSCRICAO | [09:08] Diego |
| ADR-003-DECISAO | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Decisão | 5 tentativas com backoff 1m/5m/30m/2h/12h, DLQ em tabela separada, replay manual | TRANSCRICAO | [09:17] Larissa |
| ADR-003-ALT-01 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Alternativa Descartada | Retry indefinido com backoff | TRANSCRICAO | [09:15] Diego |
| ADR-003-ALT-02 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Alternativa Descartada | Só 3 tentativas | TRANSCRICAO | [09:16] Diego |
| ADR-003-ALT-03 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Alternativa Descartada | Marcar failed na própria outbox, sem DLQ | TRANSCRICAO | [09:18] Diego |
| ADR-003-CONSEQ-01 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Trade-off | Cobre indisponibilidades de até ~15h | TRANSCRICAO | [09:17] Marcos |
| ADR-003-CONSEQ-02 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Trade-off | Após ~15h, só intervenção humana; cliente não é avisado | TRANSCRICAO | [09:37] Larissa |
| ADR-003-CONSEQ-03 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Trade-off | Outbox fica enxuta para o polling | TRANSCRICAO | [09:18] Diego |
| ADR-003-CONSEQ-04 | docs/adrs/ADR-003-retry-com-backoff-e-dlq.md | Trade-off | Últimas tentativas entregam eventos bem defasados | TRANSCRICAO | [09:17] Diego |
| ADR-004-DECISAO | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Decisão | HMAC-SHA256 sobre o corpo, secret por endpoint, rotação com grace period de 24h | TRANSCRICAO | [09:22] Sofia |
| ADR-004-ALT-01 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Alternativa Descartada | Secret global da plataforma | TRANSCRICAO | [09:21] Sofia |
| ADR-004-CONSEQ-01 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Trade-off | Rotação sem parada, mas com duas secrets válidas a manter | TRANSCRICAO | [09:21] Sofia |
| ADR-004-CONSEQ-02 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Trade-off | Redact do Pino não mascara secret nem X-Signature | CODIGO | src/shared/logger/index.ts:4-11 |
| ADR-004-CONSEQ-03 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Trade-off | Vazamento contido a um endpoint | TRANSCRICAO | [09:21] Sofia |
| ADR-004-CONSEQ-04 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Trade-off | Padrão de mercado, fácil de verificar no cliente | TRANSCRICAO | [09:20] Sofia |
| ADR-004-CONSEQ-05 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Trade-off | Secret precisa ser recuperável, não pode ser hash como passwordHash | CODIGO | prisma/schema.prisma:28 |
| ADR-005-DECISAO | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Decisão | At-least-once com X-Event-Id (UUID gerado na entrada da outbox) | TRANSCRICAO | [09:26] Larissa |
| ADR-005-ALT-01 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Alternativa Descartada | Exactly-once | TRANSCRICAO | [09:25] Diego |
| ADR-005-CONSEQ-01 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Trade-off | Responsabilidade da deduplicação passa ao cliente | TRANSCRICAO | [09:25] Sofia |
| ADR-005-CONSEQ-02 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Trade-off | Exige documentação destacada no portal | TRANSCRICAO | [09:26] Marcos |
| ADR-005-CONSEQ-03 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Trade-off | Alinhado ao mercado (Stripe, GitHub) | TRANSCRICAO | [09:25] Diego |
| ADR-005-CONSEQ-04 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Trade-off | Barato: lib uuid já usada no projeto | CODIGO | src/middlewares/request-logger.middleware.ts:2,6 |
| ADR-006-DECISAO | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Decisão | Módulo src/modules/webhooks com reuso máximo e prefixo WEBHOOK_ | TRANSCRICAO | [09:30] Larissa |
| ADR-006-ALT-01 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Alternativa Descartada | Injetar repository de webhooks no OrderService | TRANSCRICAO | [09:41] Diego |
| ADR-006-ALT-02 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Alternativa Descartada | Introduzir bibliotecas ou estruturas novas | TRANSCRICAO | [09:29] Bruno |
| ADR-006-ALT-03 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Alternativa Descartada | Códigos de erro sem prefixo de módulo, como os atuais | CODIGO | src/shared/errors/http-errors.ts |
| ADR-006-CONSEQ-01 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Trade-off | Prefixo WEBHOOK_ inaugura convenção nova | TRANSCRICAO | [09:29] Larissa |
| ADR-006-CONSEQ-02 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Trade-off | Falha de Zod sai como VALIDATION_ERROR, não WEBHOOK_INVALID_URL | CODIGO | src/middlewares/validate.middleware.ts:26-31 |
| ADR-006-CONSEQ-03 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Trade-off | NotFoundError tem código fixo; exige subclasse nova | CODIGO | src/shared/errors/http-errors.ts:27-31 |
| ADR-006-CONSEQ-04 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Trade-off | Reuso do error middleware vale só para a API, não para o worker | TRANSCRICAO | [09:29] Bruno |
| ADR-007-DECISAO | docs/adrs/ADR-007-autorizacao-dos-endpoints-de-webhook.md | Decisão | Replay exige ADMIN via requireRole; CRUD para qualquer autenticado; customer_id fora do JWT | TRANSCRICAO | [09:36] Larissa |
| ADR-007-ALT-01 | docs/adrs/ADR-007-autorizacao-dos-endpoints-de-webhook.md | Alternativa Descartada | Derivar o customer_id do JWT | TRANSCRICAO | [09:32] Larissa |
| ADR-007-ALT-02 | docs/adrs/ADR-007-autorizacao-dos-endpoints-de-webhook.md | Alternativa Descartada | Replay para qualquer role autenticada | TRANSCRICAO | [09:36] Sofia |
| ADR-007-ALT-03 | docs/adrs/ADR-007-autorizacao-dos-endpoints-de-webhook.md | Alternativa Descartada | Restringir o CRUD por role desde já (adiado) | TRANSCRICAO | [09:37] Sofia |
| ADR-007-CONSEQ-01 | docs/adrs/ADR-007-autorizacao-dos-endpoints-de-webhook.md | Trade-off | Ninguém é dono de um webhook; qualquer autenticado gerencia qualquer customer | TRANSCRICAO | [09:37] Sofia |
| ADR-007-CONSEQ-02 | docs/adrs/ADR-007-autorizacao-dos-endpoints-de-webhook.md | Trade-off | Não existe prefixo /admin; exige router dedicado | CODIGO | src/routes/index.ts |
| ADR-007-CONSEQ-03 | docs/adrs/ADR-007-autorizacao-dos-endpoints-de-webhook.md | Trade-off | Rastro de auditoria via req.user.id | CODIGO | src/middlewares/auth.middleware.ts:42 |
| ADR-007-CONSEQ-04 | docs/adrs/ADR-007-autorizacao-dos-endpoints-de-webhook.md | Trade-off | customer_id informado pelo chamador, sem checagem de posse | TRANSCRICAO | [09:32] Larissa |
