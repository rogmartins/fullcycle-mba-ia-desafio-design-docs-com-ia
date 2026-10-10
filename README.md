# Da Reunião ao Documento: Design Docs Gerados por IA

Este README registra **como** produzi o pacote de design docs desta entrega: as ferramentas, a ordem de trabalho, os prompts e as correções que precisei fazer no caminho. O conteúdo técnico da feature está nos próprios documentos em [`docs/`](docs/).

## Sobre o desafio

O desafio pede um pacote completo de design docs (PRD, RFC, FDD, ADRs, Tracker de rastreabilidade e este README) para uma feature fictícia: um sistema de webhooks que notifica sistemas externos sobre mudanças em pedidos. O material de partida é a transcrição de uma reunião técnica ([`TRANSCRICAO.md`](TRANSCRICAO.md)) e o código real de um sistema de gestão de pedidos (OMS) em [`src/`](src/).

A regra central é a rastreabilidade: nada pode ser inventado sem origem identificável na transcrição ou no código. Por isso tratei o trabalho menos como "escrever documentos" e mais como "extrair, organizar e citar". Quando precisei completar uma lacuna que a reunião não resolveu, marquei o item como premissa minha, explicitamente, em vez de apresentá-lo como decisão da equipe.

## Ferramentas de IA utilizadas

- **Claude (via Claude Code, na linha de comando)**: executou os prompts dirigidos e gerou o conteúdo de cada documento (inventário inicial, ADRs, RFC, FDD, PRD, Tracker) a partir da transcrição e do código anexados.
- **Claude (Cowork, sessão de apoio)**: atuou como meu par de revisão durante todo o processo. Ajudou a interpretar o enunciado, montou comigo o roteiro de execução (`descricao/roteiro.md`), escreveu cada um dos prompts dirigidos que usei no Claude Code e revisou cada documento gerado, linha a linha, contra a transcrição e o código, procurando citações inventadas ou fora de contexto antes de eu aprovar.

Na prática, separei quem **gera** de quem **revisa**: o Claude Code produzia o documento com acesso ao repositório, e a sessão no Cowork servia de segundo olhar, sem o viés de quem acabou de escrever o texto.

## Workflow adotado

Produzi os documentos em uma ordem que segue as dependências entre eles: cada documento só usa como base o que já tinha sido produzido e revisado.

1. **Exploração inicial** ([`prompts/01-exploracao-inicial.md`](prompts/01-exploracao-inicial.md)). Antes de escrever qualquer documento, fiz uma varredura dirigida do código e da transcrição. O resultado foi um inventário ([`prompts/01-exploracao-inicial.resultado.md`](prompts/01-exploracao-inicial.resultado.md)) com decisões (D1-D23), requisitos funcionais (RF1-RF10), não funcionais (RNF1-RNF11) e pontos de integração com o código (G1-G13). Esse inventário virou a base factual de todos os documentos seguintes.
2. **ADRs** ([`prompts/02-adrs.md`](prompts/02-adrs.md)). Comecei pelas decisões pontuais porque RFC, FDD e PRD dependem delas.
3. **RFC** ([`prompts/03-rfc.md`](prompts/03-rfc.md)), consolidando a proposta técnica geral a partir dos ADRs.
4. **FDD** ([`prompts/04-fdd.md`](prompts/04-fdd.md)), detalhando fluxos, contratos de API e erros de implementação a partir do RFC e dos ADRs.
5. **PRD** ([`prompts/05-prd.md`](prompts/05-prd.md)), elevando o que já estava decidido ao nível de produto e negócio. Ficou por último entre os quatro documentos "de conteúdo" porque é o que mais se apoia nos outros três.
6. **Tracker** ([`prompts/06-tracker.md`](prompts/06-tracker.md)), como auditoria mecânica de todos os documentos anteriores: extrai cada item identificável e a sua origem (transcrição ou código).
7. **Este README** ([`prompts/07-readme.md`](prompts/07-readme.md)), escrito depois de todo o processo concluído.

Revisei cada documento manualmente, com apoio do Claude no Cowork, antes de passar para o próximo. Isso incluiu conferir timestamps e trechos de código citados contra as fontes originais. Um documento só virava insumo da etapa seguinte depois dessa revisão. Assim, um erro em um ADR não se propagava silenciosamente para RFC, FDD e PRD.

## Prompts customizados

Todos os prompts que usei estão em [`prompts/`](prompts/). Destaco dois abaixo, colados exatamente como foram usados.

### Prompt dos ADRs (decisão pontual)

O prompt dos ADRs exige o ID do inventário ou o timestamp da transcrição para cada decisão, alternativa e consequência. Assim a IA não generaliza uma decisão pontual sem origem rastreável nem fecha por conta própria uma ambiguidade que a reunião deixou em aberto.

```text
Você vai atuar como engenheiro de software sênior produzindo ADRs
(Architecture Decision Records) para a feature "Sistema de Webhooks de
Notificação de Pedidos". Você JÁ TEM em mãos um inventário validado
(arquivo prompts/01-exploracao-inicial.resultado.md) com as decisões
fechadas (D1 a D23), requisitos, descartes, adiamentos e cruzamentos
com o código. Use esse inventário como ÚNICA fonte de verdade — não
invente nenhuma decisão, alternativa ou consequência que não esteja
nele ou rastreável à TRANSCRICAO.md / ao código em src/.

OBJETIVO
Produzir entre 5 e 8 arquivos ADR em docs/adrs/, cada um nomeado
ADR-NNN-titulo-em-kebab-case.md (ex.: ADR-001-outbox-no-mysql.md),
numerados sequencialmente a partir de 001.

COBERTURA OBRIGATÓRIA
O conjunto de ADRs deve cobrir, no mínimo, 5 das 6 decisões abaixo
(o inventário já mapeia cada uma a um ID de decisão — use esse
mapeamento):
1. Padrão Outbox no MySQL                           → D2 (+ D19, D21, D22)
2. Política de retry com backoff e DLQ               → D6, D7, D8
3. Autenticação HMAC-SHA256 com secret por endpoint   → D9
4. Garantia at-least-once com X-Event-Id              → D10
5. Worker em processo separado em polling             → D3, D4, D5, D14
6. Reuso dos padrões existentes do projeto            → D13, D11, D12, D20

Trate essas 6 como ADRs obrigatórios (um ADR por decisão, não agrupe
duas decisões obrigatórias no mesmo arquivo). Decisões secundárias do
inventário (D15–D18, D23, ou detalhes da seção "g") podem virar ADRs
adicionais SE você julgar que têm peso arquitetural própria (ex.: D17
— replay exige role ADMIN via requireRole — é candidata plausível a
ADR nº 7 ou 8); senão, deixe-as de lado para o FDD.

FORMATO DE CADA ADR (MADR)
Cada arquivo deve conter, no mínimo, estas seções:
- **Status**: Aceito (todas as decisões do inventário já foram
  fechadas na reunião, salvo indicação contrária).
- **Contexto**: o problema que motivou a decisão, citando o
  requisito/restrição de origem (RF/RNF do inventário) e, se
  aplicável, o ponto do código envolvido (caminho de arquivo real).
- **Decisão**: o que foi decidido, em 1-2 frases diretas.
- **Alternativas Consideradas**: pelo menos 1 alternativa REAL
  discutida na reunião e descartada (olhe a seção "e) Descartado" do
  inventário) ou, na ausência de uma alternativa descartada
  específica para aquela decisão, uma alternativa plausível que você
  argumente tecnicamente (deixe claro quando for este o caso).
- **Consequências**: separadas em Positivas e Negativas, com pelo
  menos 1 trade-off explícito em cada ADR (ex.: "ganha simplicidade
  operacional, perde garantia de ordering global").

REGRAS DE RASTREABILIDADE
- Toda decisão, alternativa e consequência citada precisa ser
  rastreável: cite o ID da decisão do inventário (D2, D9, etc.) e,
  quando vier da transcrição, o timestamp + falante originais (ex.:
  [09:08] Larissa). Não repita só o ID — traga a citação original
  resumida.
- Pelo menos 1 ADR tem que referenciar explicitamente arquivos,
  módulos ou classes reais do código (ex.: o ADR de reuso de padrões
  deve citar src/shared/errors/http-errors.ts, src/shared/logger,
  o padrão de módulo em src/modules/<dominio>/). Use os caminhos já
  levantados na seção "1. CÓDIGO" do inventário.
- Se uma decisão do inventário tiver uma ambiguidade associada (seção
  "Ambiguidades e pontos a esclarecer" do inventário, A1-A8), mencione
  isso na seção Consequências (ou numa nota) do ADR correspondente,
  em vez de resolver a ambiguidade por conta própria. Ambiguidades não
  decididas na reunião não são decisão sua para fechar aqui.

O QUE NÃO FAZER
- Não gere ADR para algo que está na seção "Adiado / fora de escopo"
  do inventário (ex.: email de alerta, dashboard visual, rate
  limiting) — essas não são decisões fechadas desta fase.
- Não detalhe contrato de API, payload completo ou matriz de erros
  aqui — isso é FDD, não ADR. O ADR registra a decisão arquitetural
  pontual, não a especificação de implementação.
- Não invente trade-offs genéricos de livro-texto desconectados do
  que foi efetivamente discutido — cada trade-off deve vir do
  inventário ou ser uma inferência explícita e justificada sobre o
  código real do projeto.

SAÍDA
Gere o conteúdo completo de cada um dos arquivos ADR propostos,
indicando claramente o nome do arquivo antes de cada bloco de
conteúdo. Ao final, liste em uma tabela resumida: número do ADR,
título, decisões do inventário cobertas (IDs) e se é um dos 6
obrigatórios ou um ADR adicional.
```

### Prompt do Tracker (rastreabilidade transversal)

O prompt do Tracker trata a tarefa como extração mecânica, não como criação. Ele traz as metas de cobertura do desafio como regras explícitas e uma regra anti-alucinação: na falta de origem, a IA deve omitir o item e me devolver a lista, nunca aproximar um timestamp ou caminho de arquivo.

```text
Você vai atuar como engenheiro de software auditando a própria
documentação para produzir o Tracker de Rastreabilidade da feature
"Sistema de Webhooks de Notificação de Pedidos". Este é um trabalho
MECÂNICO de extração, não de criação: todo item rastreável já está
nos documentos que você vai receber (PRD, RFC, FDD, 7 ADRs), porque
cada um deles já cita sua origem (ID do inventário, ADR, timestamp ou
caminho de arquivo). Sua tarefa é varrer cada documento e transformar
cada item identificável numa linha de tabela.

OBJETIVO
Produzir o arquivo docs/TRACKER.md, uma tabela markdown única com
EXATAMENTE este cabeçalho:

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |

ONDE:
- **ID**: identificador único. Use o prefixo do documento + tipo +
  número sequencial. Exemplos: PRD-FR-01 (requisito funcional do
  PRD), PRD-NFR-03, PRD-RISCO-02, RFC-ALT-01 (alternativa do RFC),
  RFC-ABERTO-02 (questão em aberto), FDD-CONTRATO-03 (endpoint),
  FDD-ERRO-05 (código de erro), FDD-FLUXO-01, FDD-PREMISSA-07,
  ADR-001-DECISAO, ADR-001-ALT-01, ADR-001-CONSEQ-02. Reaproveite os
  IDs que os próprios documentos já usam sempre que existirem (ex.:
  o PRD já numera PRD-FR-01 a PRD-FR-12 — use esses IDs, não invente
  novos).
- **Documento**: caminho relativo ao arquivo onde o item aparece
  (docs/PRD.md, docs/RFC.md, docs/FDD.md, docs/adrs/ADR-00N-*.md).
- **Tipo**: um de: Requisito Funcional, Requisito Não Funcional,
  Decisão, Restrição, Trade-off, Alternativa Descartada, Questão em
  Aberto, Risco, Premissa, Contrato de API, Código de Erro, Objetivo/
  Métrica, Critério de Aceite.
- **Conteúdo (resumo)**: descrição de UMA linha, curta, do item.
- **Fonte**: `TRANSCRICAO` ou `CODIGO`. Se o item for uma premissa
  sem origem na transcrição nem no código (marcada 🔶 nos
  documentos), use `PREMISSA` como fonte e deixe a Localização vazia
  ou indique "sem origem — ver seção de premissas do documento".
- **Localização**: para `TRANSCRICAO`, o timestamp + nome EXATAMENTE
  como aparece no documento de origem (formato `[hh:mm] Nome`). Para
  `CODIGO`, o caminho de arquivo real (e linha, se o documento já
  tiver). NUNCA aproxime ou invente um timestamp/caminho que não
  esteja explicitamente no documento de origem — se o documento cita
  só o ID da decisão sem timestamp, vá ao inventário
  (prompts/01-exploracao-inicial.resultado.md) buscar o timestamp
  correspondente a esse ID.

COMO VARRER CADA DOCUMENTO (gere linhas para cada um destes):

**docs/PRD.md**: cada requisito funcional (PRD-FR-*) e não funcional
(PRD-NFR-*) listado — use o ID já existente; cada objetivo/métrica
(OBJ-P*); cada item de "Fora de escopo"; cada risco (R1-R6, tipo
Risco); cada premissa (PRD-P-*, fonte PREMISSA).

**docs/RFC.md**: cada alternativa descartada da seção 4 (tipo
Alternativa Descartada); cada questão em aberto da seção 5 (tipo
Questão em Aberto).

**docs/FDD.md**: cada objetivo técnico (OBJ-*); cada endpoint da
seção 5 (tipo Contrato de API, ID FDD-CONTRATO-N); cada código de erro
da matriz da seção 6 (tipo Código de Erro); cada critério de aceite
técnico da seção 10 (CA-*); cada risco técnico da seção 11; cada
premissa da seção 13 (FDD-P-*, fonte PREMISSA, exceto quando a
premissa resolve uma questão que TEM origem na transcrição — nesse
caso use a fonte original, não PREMISSA).

**Cada ADR (001 a 007)**: a Decisão principal (tipo Decisão); cada
alternativa da seção "Alternativas Consideradas" (tipo Alternativa
Descartada); pelo menos um item de "Consequências" por ADR que seja
um trade-off relevante (tipo Trade-off).

REGRAS DE COBERTURA (critérios de aceite do desafio — não deixe de
atingir nenhum destes):
- Pelo menos 80% dos itens identificáveis nos documentos têm linha
  correspondente no tracker. Isso significa: seja exaustivo. Se um
  documento lista 12 requisitos funcionais numerados, o tracker
  precisa ter (pelo menos) 10 deles.
- Pelo menos 70% das linhas totais têm Fonte = TRANSCRICAO com
  timestamp válido no formato [hh:mm] Nome.
- Pelo menos 5 linhas têm Fonte = CODIGO com caminho de arquivo real.
  Use os 18 arquivos já listados na seção 12 do FDD como fonte
  principal para essas linhas.

REGRA ANTI-ALUCINAÇÃO
Se, ao tentar preencher a Localização de uma linha, você não
encontrar nenhum timestamp ou caminho de arquivo que sustente aquele
item em nenhum dos documentos-fonte, NÃO invente um. Classifique como
PREMISSA (se for uma escolha marcada 🔶 em algum documento) ou omita
a linha e liste-a separadamente ao final da sua resposta como "Itens
sem origem localizável — não incluídos no tracker", para que eu
decida o que fazer.

SAÍDA
Gere o conteúdo completo de docs/TRACKER.md pronto para salvar no
repositório. Ao final, informe: quantas linhas totais, quantas com
Fonte TRANSCRICAO (e o %), quantas com Fonte CODIGO, e uma estimativa
de quantos itens identificáveis existem nos documentos-fonte (para eu
poder confirmar que a cobertura de 80% foi atingida).
```

## Iterações e ajustes

### 1. Setup do repositório

O fork criado pelo botão "Fork" do GitHub não era reconhecido como fork de fato pela API do GitHub. Pior: ele deixaria o meu trabalho visível na rede pública de forks do repositório base, exposto a outros alunos. Abandonei essa abordagem, excluí o repositório e criei um repositório independente e privado, para onde migrei o conteúdo que eu já tinha clonado.

### 2. Contradição entre RFC e FDD sobre o estado da outbox

A transcrição menciona um estado "falhou" dentro da própria outbox e também uma tabela de DLQ separada, mas a reunião não explica a diferença entre os dois. Em vez de escolher um lado em silêncio, o RFC registrou a inconsistência na sua seção de notas de consistência entre as fontes. O FDD resolveu a ambiguidade com uma premissa própria, a **P-06**: `FAILED` é um estado transitório dentro da outbox, e só a falha permanente, depois de esgotadas as tentativas, vai para a DLQ. A P-06 está marcada como escolha do FDD, não como decisão da reunião.

### 3. Lacunas encontradas pelo Tracker

Quando gerei o Tracker como auditoria final, dois itens do FDD apareceram sem origem rastreável na transcrição: um endpoint de detalhe de webhook (`GET /webhooks/:id`) e um risco de SSRF por cadastro de URL interna. Em vez de forçar uma citação inexistente, formalizei os dois como novas premissas do FDD, a **P-31** e a **P-32**. Ambas estão marcadas como propostas do documento, não como consenso da reunião. Atualizei o FDD antes do commit final. Foi a regra anti-alucinação do prompt do Tracker que expôs essas lacunas: a auditoria pegou o que a revisão documento a documento tinha deixado passar.

### Número de iterações

No total, foram **cerca de 7 iterações principais** (gerar → revisar → ajustar o prompt ou o documento → regerar), aproximadamente uma por etapa do workflow, contando a correção do FDD disparada pelo Tracker.

## Como navegar a entrega

Sugiro esta ordem de leitura. Ela vai do documento mais estratégico ao mais operacional e termina no Tracker, que é a verificação transversal de todos os outros:

1. [`docs/PRD.md`](docs/PRD.md): problema, público, escopo e métricas de sucesso.
2. [`docs/RFC.md`](docs/RFC.md): proposta técnica, alternativas e questões em aberto.
3. [`docs/adrs/`](docs/adrs/): os 7 ADRs, um por decisão arquitetural.
   - [`ADR-001-outbox-no-mysql.md`](docs/adrs/ADR-001-outbox-no-mysql.md)
   - [`ADR-002-worker-separado-em-polling.md`](docs/adrs/ADR-002-worker-separado-em-polling.md)
   - [`ADR-003-retry-com-backoff-e-dlq.md`](docs/adrs/ADR-003-retry-com-backoff-e-dlq.md)
   - [`ADR-004-hmac-sha256-com-secret-por-endpoint.md`](docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md)
   - [`ADR-005-at-least-once-com-x-event-id.md`](docs/adrs/ADR-005-at-least-once-com-x-event-id.md)
   - [`ADR-006-reuso-dos-padroes-existentes.md`](docs/adrs/ADR-006-reuso-dos-padroes-existentes.md)
   - [`ADR-007-autorizacao-dos-endpoints-de-webhook.md`](docs/adrs/ADR-007-autorizacao-dos-endpoints-de-webhook.md)
4. [`docs/FDD.md`](docs/FDD.md): fluxos, contratos de API, matriz de erros e integração com o código.
5. [`docs/TRACKER.md`](docs/TRACKER.md): origem de cada item (transcrição ou código).

A pasta [`prompts/`](prompts/) guarda todos os prompts que usei, de `01` a `07`, e também o inventário gerado na exploração inicial. É o caminho para quem quiser ver o processo completo.

## Enunciado original

O enunciado completo do desafio está no repositório base: [devfullcycle/mba-ia-desafio-design-docs-com-ia](https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia).
