# PRD — Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| **Feature** | Sistema de Webhooks de Notificação de Pedidos |
| **Responsável de produto** | Marcos (PM) |
| **Status** | Aprovado em reunião; implementação pendente da validação das premissas do [FDD §13](FDD.md#13-premissas-assumidas-pelo-fdd) |
| **Data** | 2026-10-10 |
| **Documentos relacionados** | [RFC](RFC.md) · [FDD](FDD.md) · [ADRs 001–007](adrs/README.md) |

> **Como ler este documento.** Toda afirmação remete a um ID do inventário (D/RF/RNF, em `prompts/01-exploracao-inicial.resultado.md`), a um ADR, ao RFC/FDD ou a um timestamp com o nome de quem falou em `TRANSCRICAO.md`. O que **não** foi decidido na reunião aparece como **🔶 Premissa PRD-P-NN** e está consolidado na [seção 13](#13-premissas-deste-prd). Este PRD responde "por que e o quê". O "como" fica no RFC e no FDD.

---

## 1. Resumo e contexto da feature

Três clientes B2B, **Atlas Comercial, MaxDistribuição e Nova Cargo**, pediram formalmente para ser avisados em tempo real quando o status dos pedidos deles muda na plataforma ([09:00] Marcos; RF1). Hoje eles descobrem mudanças consultando `GET /orders` de tempos em tempos (polling), o que deixa a integração "lenta e cara pra eles" ([09:00] Marcos).

**Solução em uma frase:** a plataforma passa a enviar automaticamente uma notificação assinada (webhook) para um endereço cadastrado pelo cliente sempre que o status de um pedido dele muda, com reenvio automático em caso de falha e recuperação manual pela operação ([RFC §1](RFC.md#1-resumo-executivo-tldr)). O fluxo é só de saída: os clientes "querem receber, não mandar" ([09:02] Marcos; D1).

## 2. Problema e motivação

**Custo para o cliente.**

- A integração por polling é lenta: o cliente só descobre a mudança na próxima consulta, e precisa "ficar atualizando manualmente" ([09:02] Marcos).
- É cara: cada cliente mantém consultas repetidas a `GET /orders`, mesmo quando nada mudou ([09:00] Marcos).

**Custo para o negócio.**

- **Risco de churn:** a Atlas Comercial sinalizou que pode migrar para o concorrente se a feature não for entregue no prazo ([09:00] Marcos; RNF11).
- **Prazo comercial:** a Atlas pediu a entrega até o **fim de novembro** ([09:45] Marcos). Na abertura, Marcos citou "fim do trimestre" ([09:00]); o prazo confirmado ao final da reunião foi fim de novembro, e é ele que vale para este PRD.
- O pedido veio de três clientes ao mesmo tempo, o que indica uma necessidade comum da base B2B, não um caso isolado ([09:00] Marcos).

## 3. Público-alvo e cenários de uso

| Público | O que usa | Origem |
|---|---|---|
| **Cliente B2B integrador** (sistemas da Atlas, MaxDistribuição, Nova Cargo) | Recebe as notificações no endpoint dele, verifica a assinatura e descarta duplicatas | RF1, D9, D10 |
| **Usuário autenticado que representa o cliente** | Cadastra, edita, remove e lista webhooks; escolhe os status de interesse; rotaciona a secret; consulta o histórico de entregas. Usa a API diretamente, com o JWT da plataforma | RF2–RF6; [09:32] Marcos; D18 |
| **Administrador da plataforma (role ADMIN)** | Reprocessa notificações que esgotaram as tentativas automáticas | RF7, RF8, D17 |

**Cenário A: cliente passa a ser avisado do despacho.** A Atlas cadastra um endpoint `https` e escolhe ouvir só `SHIPPED` e `DELIVERED` (exemplo de [09:33] Marcos). Recebe a secret na resposta do cadastro. Quando um pedido dela é despachado, o sistema da Atlas recebe em poucos segundos uma notificação assinada com o número do pedido e o status anterior e o novo. Para mais detalhes do pedido, consulta `GET /orders/:id` ([09:43] Diego). Ela deixa de fazer polling.

**Cenário B: cliente fora do ar em manutenção planejada.** O endpoint da Nova Cargo fica indisponível por 2 horas (situação real citada em [09:16] Diego). As notificações desse período falham e são reenviadas automaticamente em intervalos crescentes. Quando o endpoint volta, elas chegam sem que ninguém precise agir ([ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md)).

**Cenário C: administrador recupera entregas que falharam de vez.** O endpoint de um cliente ficou fora por mais de ~15 horas, e as notificações esgotaram as tentativas ([09:17] Diego). Depois que o cliente corrige o problema, um ADMIN reprocessa cada notificação pendente; o sistema registra quem fez o reprocessamento ([09:36] Sofia; RF7, RF8). Como a notificação reenviada tem o mesmo identificador, o cliente consegue reconhecer o que já tinha recebido ([FDD P-04](FDD.md#13-premissas-assumidas-pelo-fdd)).

**Cenário D: cliente troca a secret depois de um vazamento.** Um cliente vaza a secret no log da própria aplicação (episódio real, [09:22] Diego). Ele pede uma nova secret pela API; a antiga continua aceita por 24h para ele migrar os sistemas e depois deixa de valer ([09:21] Sofia; RF5).

## 4. Objetivos e métricas de sucesso

| ID | Objetivo de produto | Métrica | Meta | Origem |
|---|---|---|---|---|
| **OBJ-P1** | Notificação percebida como "tempo real" pelo cliente | Tempo entre a mudança de status e o 1º envio da notificação, em operação normal (endpoint saudável, sem fila acumulada). Medido pela latência de entrega do [FDD §8.1](FDD.md#81-métricas-sugeridas) | **Abaixo de 10 segundos.** 🔶 **PRD-P-01:** meta de **≥ 95% dos eventos** abaixo de 10s | RNF1, [09:02] Marcos; FDD OBJ-01 |
| **OBJ-P2** | Reter a Atlas Comercial cumprindo o prazo comercial | Data de liberação em produção | **Até o fim de novembro**, dentro de **3 sprints**, incluindo a revisão de segurança | RNF11, [09:45] Marcos; D23, [09:47] Larissa |
| **OBJ-P3** | Nenhuma mudança de status deixa de gerar notificação | Mudanças de status com webhook assinante e sem evento registrado | **0 ocorrências** | RNF2, [09:40] Bruno; D19 |
| **OBJ-P4** | Indisponibilidades comuns do cliente se resolvem sem intervenção humana | Janela coberta pelo reenvio automático | **≈ 14h36** entre a 1ª falha e a última tentativa (cobre com folga a manutenção de 2h citada) | D6, [09:17] Diego; [FDD P-01](FDD.md#13-premissas-assumidas-pelo-fdd) |
| **OBJ-P5** | Os clientes que pediram a feature deixam o polling | Clientes-piloto recebendo notificações em produção | 🔶 **PRD-P-02:** **3 de 3** (Atlas, MaxDistribuição, Nova Cargo) | [09:00] Marcos |

## 5. Escopo

### 5.1 Incluso

- **Cadastro e gestão de webhooks:** o cliente registra um ou mais endereços `https`, edita, desativa, remove e lista os webhooks de um customer (RF2, RF3; RNF4).
- **Escolha dos eventos de interesse:** cada endereço define em quais status de pedido quer ser avisado (RF4, D16).
- **Notificação automática** a cada mudança de status de pedido, com os dados essenciais do pedido e sem a lista de itens (RF1; [09:43] Diego).
- **Segurança da entrega:** cada notificação é assinada com uma secret exclusiva do endereço, gerada pela plataforma, que o cliente pode trocar com 24h de convivência entre a antiga e a nova (RF5, D9; [ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md)).
- **Identificação de cada notificação**, para o cliente reconhecer duplicatas e saber qual cadastro a recebeu (RF10, D10).
- **Histórico de entregas** por endereço: sucesso ou falha, conteúdo enviado, resposta e tempo de resposta (RF6).
- **Reenvio automático e recuperação de falhas:** retentativas com intervalos crescentes; o que esgotar as tentativas fica guardado e pode ser reprocessado manualmente por um administrador, com registro de quem o fez (RF7, RF8, RF9; [ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md)).

### 5.2 Fora de escopo

Itens descartados ou adiados explicitamente na reunião ([RFC §5](RFC.md#5-questões-em-aberto); inventário, seção f):

| Item | Situação | Origem |
|---|---|---|
| **E-mail de alerta** ao cliente quando o webhook falha repetidamente | Adiado para a próxima fase, "depois que a gente medir o impacto" | [09:37] Larissa; [09:38] Marcos ("anotado como futuro") |
| **Rate limiting de saída** (limitar a quantidade de notificações por cliente) | Ponto em aberto: "observar e decidir depois" | [09:39] Diego; [09:39] Larissa |
| **Dashboard visual** para o cliente ver os webhooks | Fora de escopo; projeto separado do time de frontend. Nesta fase, só API | [09:40] Larissa |
| **Arquivamento** de notificações já entregues | Fora de escopo desta feature | [09:08] Diego |
| **Vários workers em paralelo** (escala horizontal do envio) | "Problema do futuro" | [09:13] Diego |
| **Restrição de quem pode configurar webhooks** por perfil ou por cliente | Adiado: "Mais pra frente a gente pode endurecer" | [09:37] Sofia |
| **Webhooks de entrada** (cliente enviando para a plataforma) | Descartado: o fluxo é só de saída | D1; [09:02] Marcos; [09:03] Sofia |
| **Garantia de entrega exatamente uma vez** | Descartada em favor de "pelo menos uma vez" | [09:25] Diego; [ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md) |
| **Ordem global** das notificações entre pedidos diferentes | Não garantida; só por pedido | D5; [09:13] Larissa; [09:14] Marcos |
| **Notificação na criação do pedido** | 🔶 Não nesta fase ([FDD P-05](FDD.md#13-premissas-assumidas-pelo-fdd)); a reunião só tratou de mudanças de status | A6; [09:40] Bruno |

## 6. Requisitos funcionais

| ID | Requisito (linguagem de produto) | Origem |
|---|---|---|
| **PRD-FR-01** | O sistema avisa o cliente B2B automaticamente sempre que o status de um pedido dele muda. | RF1; [09:00] Marcos |
| **PRD-FR-02** | O cliente cadastra um webhook informando o endereço de destino e a lista de status que quer receber. A secret de assinatura é **gerada pela plataforma** e mostrada ao cliente no cadastro. | RF2; [09:31] Marcos |
| **PRD-FR-03** | O cliente edita, remove e lista os webhooks de um customer. O customer é indicado explicitamente na requisição, e não deduzido do login. | RF3; [09:33] Bruno; D15, [09:32] Larissa |
| **PRD-FR-04** | Cada webhook escolhe de quais status quer ser avisado (ex.: só `SHIPPED` e `DELIVERED`). Mudanças para status não assinados não geram notificação para aquele cliente. | RF4; [09:33] Marcos; D16, [09:34] Bruno |
| **PRD-FR-05** | O cliente pede uma nova secret pela API. A anterior continua aceita por 24 horas e depois expira. | RF5; D9; [09:21] Sofia |
| **PRD-FR-06** | O cliente consulta o histórico das últimas entregas de cada webhook (até 100 por consulta): sucesso ou falha, conteúdo enviado, resposta recebida e tempo de resposta. | RF6; [09:34] Marcos; [FDD P-10](FDD.md#13-premissas-assumidas-pelo-fdd) |
| **PRD-FR-07** | Se o endpoint do cliente falhar ou não responder, o sistema reenvia automaticamente em intervalos crescentes (1 min, 5 min, 30 min, 2 h, 12 h). | D6; [09:17] Larissa |
| **PRD-FR-08** | Esgotadas as tentativas, a notificação é guardada à parte, com o conteúdo, o motivo da falha e o momento, para análise e reprocessamento. | RF9; D7; [09:15] e [09:18] Diego |
| **PRD-FR-09** | Um administrador da plataforma reprocessa manualmente uma notificação guardada, que volta para a fila de envio. Usuários sem perfil ADMIN não podem fazer isso. | RF7; D8, [09:18] Diego; D17, [09:36] Sofia/Larissa |
| **PRD-FR-10** | Todo reprocessamento registra quem o executou, para auditoria. | RF8; [09:36] Sofia |
| **PRD-FR-11** | Cada notificação traz um identificador único do evento, a assinatura, o identificador do webhook que a originou e o momento do envio, para o cliente verificar a origem, descartar duplicatas e saber qual cadastro recebeu. | RF10; [09:25] e [09:44] Diego; [09:44] Sofia |
| **PRD-FR-12** | A notificação é enxuta: traz o tipo do evento, o pedido (id, número, customer, valor total), o status anterior e o novo, e reflete o estado do pedido no momento da mudança. Itens não são enviados; o cliente consulta o pedido se precisar de detalhes. | [09:43] Diego; D22, [09:52] Larissa; [FDD P-14](FDD.md#13-premissas-assumidas-pelo-fdd) |

## 7. Requisitos não funcionais

| ID | Requisito (linguagem de produto) | Origem |
|---|---|---|
| **PRD-NFR-01** | **Latência percebida:** em operação normal, a notificação sai em menos de 10 segundos após a mudança de status. O atraso mínimo de até ~2s é aceito. | RNF1, [09:02] Marcos; D3, [09:10] Larissa |
| **PRD-NFR-02** | **Confiabilidade:** se o status mudou e há webhook interessado, a notificação fica registrada; se a mudança for desfeita, nenhuma notificação é gerada. | RNF2, [09:40] Bruno; [ADR-001](adrs/ADR-001-outbox-no-mysql.md) |
| **PRD-NFR-03** | **Isolamento:** um cliente lento ou fora do ar não atrasa nem impede a mudança de status de nenhum pedido. | RNF3, [09:04] Bruno |
| **PRD-NFR-04** | **Autenticidade e integridade:** o cliente consegue verificar que a notificação veio da plataforma e não foi alterada, com um algoritmo padrão de mercado (HMAC-SHA256). | D9, [09:19–09:22] Sofia |
| **PRD-NFR-05** | **Segurança da secret:** cada webhook tem secret própria, para que o vazamento de uma não comprometa as outras. A secret só é exibida no cadastro e na troca. | RNF7, [09:21] Sofia; [FDD CA-12](FDD.md#10-critérios-de-aceite-técnicos) |
| **PRD-NFR-06** | **Canal seguro:** só endereços `https` são aceitos; `http` é recusado no cadastro. | RNF4, [09:23] Sofia |
| **PRD-NFR-07** | **Entrega "pelo menos uma vez":** a mesma notificação pode chegar mais de uma vez, e o cliente precisa descartar repetições pelo identificador do evento. | D10, [09:24–09:26] Diego/Larissa; [ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md) |
| **PRD-NFR-08** | **Ordem:** notificações de um mesmo pedido chegam na ordem das mudanças enquanto houver um único processo de envio e não houver falhas; não há ordem global entre pedidos. | RNF8, [09:13] Larissa; [FDD §11](FDD.md#11-riscos-e-mitigação) |
| **PRD-NFR-09** | **Limites de envio:** o cliente tem até 10 segundos para responder, senão conta como falha e entra no reenvio; notificações acima de 64KB não são enviadas e são tratadas como erro. | RNF6, [09:42] Diego; RNF5, [09:24] Larissa |
| **PRD-NFR-10** | **Disponibilidade:** reinícios ou deploys da API não interrompem o envio, e uma parada do envio não perde notificações, que ficam aguardando. | D4, [09:11] Diego; [RFC §6](RFC.md#6-impacto-e-riscos) |
| **PRD-NFR-11** | **Custo operacional:** nenhuma infraestrutura nova; a feature roda sobre o banco e a stack atuais. | RNF9, [09:07] Diego |
| **PRD-NFR-12** | **Processo:** revisão de segurança da Sofia de pelo menos 2 dias úteis antes do deploy. | RNF10, [09:46] e [09:49] Sofia |

## 8. Decisões e trade-offs principais

- **"Pelo menos uma vez" em vez de "exatamente uma vez".** Simplifica a plataforma e segue o padrão do mercado ("Stripe faz assim, GitHub faz assim", [09:25] Diego), mas **passa ao cliente a responsabilidade de descartar duplicatas** ([09:25] Sofia). A contrapartida de produto é documentação destacada no portal ([09:26] Marcos). → [ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md)
- **Reenvio automático por ~15h, depois recuperação manual.** Cinco retentativas cobrem indisponibilidades longas sem deixar notificações "penduradas pra sempre" ([09:15] Diego). Depois disso, o reprocessamento depende de um ADMIN; nesta fase não há aviso automático ao cliente (e-mail adiado). Marcos aceitou: "Se um cliente meu cair por 15 horas, ele já tá com problema sério dele" ([09:17]). → [ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md)
- **Sem infraestrutura nova, com latência de até ~2s.** O registro da notificação junto com a mudança de status e o envio por um processo separado garantem consistência e isolamento sem subir Redis ou similar ([09:07] Diego). O preço é um atraso mínimo de ~2s, aceito por estar bem abaixo dos 10s ([09:10] Larissa). → [ADR-001](adrs/ADR-001-outbox-no-mysql.md), [ADR-002](adrs/ADR-002-worker-separado-em-polling.md)
- **Ordem garantida só por pedido.** Um único processo de envio basta para a necessidade atual ("Os clientes nunca pediram garantia de ordering global", [09:14] Marcos), mas limita a escala futura. → [ADR-002](adrs/ADR-002-worker-separado-em-polling.md)
- **Configuração aberta a qualquer usuário autenticado, por enquanto; reprocessamento só para ADMIN.** Acelera a entrega, mas aceita que hoje qualquer usuário autenticado gerencie webhooks de qualquer customer ([RFC §6](RFC.md#6-impacto-e-riscos)). → [ADR-007](adrs/ADR-007-autorizacao-dos-endpoints-de-webhook.md)
- **Prazo de 3 sprints com a revisão de segurança no fim**, para caber no fim de novembro pedido pela Atlas ([09:46–09:47] Larissa). → D23

## 9. Dependências

**Pessoas e times**

| Dependência | Responsável | Origem |
|---|---|---|
| Revisão de segurança (assinatura e geração de secret) de **pelo menos 2 dias úteis antes do deploy**, agendada com antecedência | Sofia | RNF10; [09:46] e [09:49] Sofia |
| Documentação no **portal do desenvolvedor**: como integrar via API, verificação de assinatura e descarte de duplicatas | Marcos | [09:26] e [09:40] Marcos |
| Confirmação do prazo e comunicação com os três clientes | Marcos | [09:47] e [09:49] Marcos |
| Sessão de revisão do design com Bruno e Diego antes de começar a codar | Larissa | [09:50] Larissa |
| Validação, com a equipe, das premissas do FDD antes da implementação | Larissa, Bruno, Diego, Sofia | [FDD §13](FDD.md#13-premissas-assumidas-pelo-fdd) |
| Dashboard visual (futuro, não bloqueia esta fase) | Time de frontend | [09:40] Larissa |

**Dependências técnicas de alto nível**

- Banco de dados e stack atuais da plataforma, sem infraestrutura nova (RNF9).
- Um **novo processo de envio** para implantar e monitorar em produção, separado da API ([ADR-002](adrs/ADR-002-worker-separado-em-polling.md); [RFC §6](RFC.md#6-impacto-e-riscos)).
- Guarda segura da chave usada para proteger as secrets dos clientes ([FDD §11](FDD.md#11-riscos-e-mitigação), premissa P-11).
- Do lado do cliente: endpoint `https` público e capacidade de verificar HMAC-SHA256 e de descartar duplicatas (RNF4, D9, D10).

## 10. Riscos e mitigação

> 🔶 **PRD-P-03:** as classificações de **probabilidade** e **impacto** abaixo são avaliação deste PRD a partir dos riscos do [RFC §6](RFC.md#6-impacto-e-riscos) e do [FDD §11](FDD.md#11-riscos-e-mitigação); não foram quantificadas na reunião.

| # | Risco de negócio | Probabilidade | Impacto | Mitigação |
|---|---|---|---|---|
| **R1** | **Atraso na entrega para a Atlas.** O prazo de 3 sprints é apertado, a revisão de segurança obrigatória fica no fim, e questões em aberto (contagem de tentativas, assinatura do timestamp, identificador no reprocessamento) podem gerar retrabalho se fechadas tarde | Média | **Alto**: risco de churn da Atlas para o concorrente (RNF11) | Agendar a revisão da Sofia logo no início ([09:49] Sofia); validar as premissas do FDD na sessão de design antes de codar ([09:50] Larissa); Marcos alinha o prazo com a Atlas e comunica qualquer desvio cedo ([09:47] Marcos) |
| **R2** | **Cliente não descarta duplicatas** e processa a mesma notificação duas vezes, gerando reclamação de suporte | Média | Médio: erro operacional no sistema do cliente atribuído à plataforma ([09:25] Sofia) | Documentação destacada no portal ([09:26] Marcos); identificador do evento estável, inclusive no reprocessamento ([FDD P-04](FDD.md#13-premissas-assumidas-pelo-fdd)); validar o descarte com cada cliente-piloto (seção 12) |
| **R3** | **Notificações de um mesmo pedido fora de ordem** quando uma falha e é reenviada depois da seguinte (ex.: `SHIPPED` antes de `PROCESSING`) | Baixa | Médio: cliente interpreta estado errado do pedido | Documentar a limitação no portal; cada notificação traz status anterior, status novo e momento da mudança, o que permite reordenar ([FDD §11](FDD.md#11-riscos-e-mitigação), P-20) |
| **R4** | **Configuração aberta demais:** qualquer usuário autenticado pode cadastrar ou alterar webhooks de qualquer customer, inclusive redirecionando dados de pedidos | Baixa | Alto: exposição de dados de pedidos de um cliente | Risco aceito nesta fase ([ADR-007](adrs/ADR-007-autorizacao-dos-endpoints-de-webhook.md)); endurecimento já registrado como evolução ([09:37] Sofia); levar à revisão de segurança (RNF10) |
| **R5** | **Falha silenciosa para o cliente:** após ~15h fora do ar, as notificações esgotam as tentativas e o cliente não é avisado, porque o e-mail de alerta foi adiado | Baixa | Médio: cliente perde atualizações até perceber | Histórico de entregas consultável pelo cliente (RF6); monitoramento das notificações guardadas e reprocessamento por ADMIN ([FDD §8.1](FDD.md#81-métricas-sugeridas); RF7); reavaliar o e-mail na próxima fase com base nesse impacto ([09:37] Larissa) |
| **R6** | **Processo de envio parado** em produção: as notificações deixam de sair | Baixa | Alto: todos os clientes deixam de ser avisados até a retomada | Nenhuma notificação se perde, elas aguardam ([RFC §6](RFC.md#6-impacto-e-riscos)); alarme pela idade da notificação pendente mais antiga ([FDD §8.1](FDD.md#81-métricas-sugeridas)) |

## 11. Critérios de aceitação

Critérios de produto, verificáveis por quem não lê código (os critérios técnicos estão no [FDD §10](FDD.md#10-critérios-de-aceite-técnicos)):

- [ ] **AC-01:** um cliente com webhook cadastrado para `SHIPPED` recebe uma notificação assinada **em menos de 10 segundos** depois que um pedido dele é marcado como despachado (PRD-FR-01, PRD-NFR-01).
- [ ] **AC-02:** um cliente que **não** assinou `PAID` não recebe nada quando um pedido dele é pago (PRD-FR-04).
- [ ] **AC-03:** a secret aparece apenas na resposta do cadastro e da troca de secret; nenhuma consulta posterior a mostra (PRD-FR-02, PRD-NFR-05).
- [ ] **AC-04:** a tentativa de cadastrar um endereço `http://` é recusada com mensagem de erro de URL (PRD-NFR-06).
- [ ] **AC-05:** o cliente consegue verificar a assinatura de uma notificação recebida usando a secret e uma biblioteca padrão de HMAC-SHA256 (PRD-NFR-04).
- [ ] **AC-06:** depois de trocar a secret, notificações assinadas continuam sendo aceitas pelo cliente que ainda usa a antiga durante 24h; depois desse prazo, só a nova funciona (PRD-FR-05).
- [ ] **AC-07:** com o endpoint do cliente fora do ar por 2 horas, as notificações do período chegam automaticamente quando ele volta, sem ação da operação (PRD-FR-07; Cenário B).
- [ ] **AC-08:** uma notificação que esgota as tentativas pode ser reprocessada por um ADMIN e chega ao cliente com o **mesmo identificador de evento**; um usuário OPERATOR não consegue reprocessar (PRD-FR-08, PRD-FR-09).
- [ ] **AC-09:** cada reprocessamento fica registrado com o usuário que o executou (PRD-FR-10).
- [ ] **AC-10:** o histórico de entregas de um webhook mostra, para cada envio, se deu certo, o conteúdo enviado, a resposta e o tempo de resposta (PRD-FR-06).
- [ ] **AC-11:** a mudança de status de um pedido continua funcionando normalmente mesmo com o endpoint do cliente lento ou fora do ar (PRD-NFR-03).
- [ ] **AC-12:** a documentação de integração está publicada no portal do desenvolvedor, com destaque para o descarte de duplicatas (PRD-NFR-07; [09:26] Marcos).
- [ ] **AC-13:** a revisão de segurança da Sofia foi concluída e aprovada antes do deploy (PRD-NFR-12).

## 12. Estratégia de testes e validação

A validação de produto é feita com os três clientes que pediram a feature. A estratégia de testes automatizados está no FDD.

> 🔶 **PRD-P-04:** a reunião não definiu um processo de piloto. Os passos abaixo são proposta deste PRD, montada a partir dos cenários e decisões discutidos, e precisam ser combinados por Marcos com os clientes ([09:49] Marcos: "Eu atualizo os clientes hoje à tarde").

1. **Pré-requisitos de liberação:** revisão de segurança aprovada (RNF10) e documentação do portal publicada ([09:26] Marcos).
2. **Onboarding assistido dos pilotos** (Atlas primeiro, por causa do risco de churn; depois MaxDistribuição e Nova Cargo): cada cliente cadastra o endpoint, escolhe os status de interesse e confirma que verifica a assinatura (AC-01 a AC-05).
3. **Validação de latência:** acompanhar pedidos reais dos pilotos e conferir que as notificações chegam em menos de 10s ([FDD §8.1](FDD.md#81-métricas-sugeridas); OBJ-P1).
4. **Ensaio de indisponibilidade:** combinar com um piloto uma janela em que o endpoint dele fica fora, e confirmar a recuperação automática e, depois de esgotadas as tentativas, o reprocessamento por ADMIN (Cenários B e C; AC-07, AC-08).
5. **Ensaio de troca de secret:** o piloto troca a secret e confirma que não perde notificações durante as 24h de convivência (Cenário D; AC-06).
6. **Validação de duplicatas:** confirmar com cada piloto que o sistema dele descarta uma notificação repetida pelo identificador do evento (R2; AC-08).
7. **Critério de sucesso do piloto:** os três clientes recebendo notificações em produção e deixando o polling (OBJ-P5). Os dados de falha e reprocessamento observados alimentam a decisão sobre e-mail de alerta e rate limiting na próxima fase ([09:37] e [09:39] Larissa).

## 13. Premissas deste PRD

> Escolhas deste documento, **não** decisões da reunião. Precisam ser validadas por Marcos (PM) com a equipe e os clientes.

| ID | Premissa | Justificativa | Onde é usada |
|---|---|---|---|
| **PRD-P-01** | A meta de latência é **≥ 95% dos eventos** em menos de 10s, em operação normal | A reunião fixou o limite de 10s ([09:02] Marcos), mas não o percentual. Falhas de endpoint e reenvios ficam fora da medição por natureza | OBJ-P1 |
| **PRD-P-02** | Sucesso de adoção = os 3 clientes que pediram a feature recebendo notificações em produção | Deriva do pedido formal dos três clientes ([09:00] Marcos); não foi definida métrica de adoção | OBJ-P5, seção 12 |
| **PRD-P-03** | Probabilidade e impacto dos riscos são avaliação qualitativa deste PRD | A reunião levantou os riscos, mas não os classificou | Seção 10 |
| **PRD-P-04** | Processo de piloto com onboarding assistido e ensaios de falha | Nenhum processo de validação com clientes foi discutido | Seção 12 |
