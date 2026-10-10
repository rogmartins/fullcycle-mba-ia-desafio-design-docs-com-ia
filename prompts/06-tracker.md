# Prompt — Geração do Tracker (Etapa 6 do roteiro)

> Usado na Etapa 6 do roteiro (`descricao/roteiro.md`). Depende de TODOS os documentos já produzidos: `entregavel/docs/PRD.md`, `entregavel/docs/RFC.md`, `entregavel/docs/FDD.md` e os 7 ADRs em `entregavel/docs/adrs/`. Anexe todos ao rodar este prompt — este é o único prompt que precisa dos 4 documentos + ADRs ao mesmo tempo.

```
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
