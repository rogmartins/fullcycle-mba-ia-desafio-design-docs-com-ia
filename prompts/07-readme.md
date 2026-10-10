# Prompt — README do processo (Etapa 7 do roteiro)

> Usado na Etapa 7 do roteiro (`descricao/roteiro.md`). Esta etapa é diferente das anteriores: não é para a IA "inventar" um processo genérico, é para documentar o processo REAL que já aconteceu. Por isso o prompt abaixo já traz os fatos concretos (ferramentas, ordem, iterações) para a IA só organizar e redigir — não repita a lista de ADRs, RFC, FDD, PRD, Tracker: isso já está nos próprios documentos.
>
> Rode este prompt sem anexar a transcrição nem o código — não são necessários aqui. Anexe só os arquivos de prompts (`prompts/01-exploracao-inicial.md` a `prompts/06-tracker.md`) se quiser que a IA copie os blocos de código literalmente.

```
Você vai escrever o entregavel/README.md, substituindo o conteúdo atual
(que hoje é o enunciado do desafio) pela documentação do processo real
de produção deste pacote de design docs. NÃO é para gerar conteúdo
genérico sobre "como usar IA para design docs" — é para registrar o que
de fato aconteceu nesta produção, com os fatos que vou te dar abaixo.

ESTRUTURA OBRIGATÓRIA

1. **Sobre o desafio** (1-2 parágrafos, com minhas próprias palavras):
   o desafio pede um pacote de design docs (PRD, RFC, FDD, ADRs,
   Tracker e este README) para uma feature fictícia — um sistema de
   webhooks de notificação de pedidos — a partir de uma transcrição de
   reunião e do código real de um sistema de gestão de pedidos (OMS).
   A regra central é rastreabilidade: nada pode ser inventado sem
   origem identificável na transcrição ou no código.

2. **Ferramentas de IA utilizadas**: liste e descreva o papel de cada
   uma:
   - **Claude (via Claude Code, na linha de comando)**: executou os
     prompts dirigidos e gerou o conteúdo de cada documento (inventário
     inicial, ADRs, RFC, FDD, PRD, Tracker) a partir da transcrição e
     do código anexados.
   - **Claude (Cowork, sessão de apoio)**: atuou como par de revisão
     durante todo o processo — ajudou a interpretar o enunciado, montou
     o roteiro de execução (descricao/roteiro.md), escreveu cada um dos
     prompts dirigidos usados no Claude Code, e revisou cada documento
     gerado linha a linha contra a transcrição e o código, procurando
     citações inventadas ou fora de contexto antes da aprovação.

3. **Workflow adotado**: descreva a ordem de produção e por quê:
   - Primeiro, uma exploração inicial dirigida do código e da
     transcrição (prompts/01-exploracao-inicial.md), gerando um
     inventário de decisões (D1-D23), requisitos funcionais (RF1-RF10),
     não funcionais (RNF1-RNF11) e pontos de integração com o código
     (G1-G13), usado como base factual para todos os documentos
     seguintes.
   - Depois, os ADRs (prompts/02-adrs.md) — decisões pontuais primeiro,
     porque RFC/FDD/PRD dependem delas.
   - Em seguida o RFC (prompts/03-rfc.md), consolidando a proposta
     técnica geral a partir dos ADRs.
   - Depois o FDD (prompts/04-fdd.md), detalhando fluxos, contratos de
     API e erros de implementação a partir do RFC e dos ADRs.
   - Depois o PRD (prompts/05-prd.md), elevando o que já estava
     decidido ao nível de produto/negócio — por último entre os quatro
     documentos "de conteúdo", porque é o que mais se apoia nos outros
     três.
   - Por fim, o Tracker (prompts/06-tracker.md), como auditoria
     mecânica de todos os documentos anteriores, extraindo cada item
     identificável e sua origem (transcrição ou código).
   - Este README é escrito por último, depois de todo o processo
     concluído.
   - Cada documento foi revisado manualmente (com apoio do Claude no
     Cowork) antes de seguir para o próximo passo, incluindo
     verificação pontual de timestamps e trechos de código citados
     contra as fontes originais.

4. **Prompts customizados**: inclua, em blocos de código markdown,
   PELO MENOS estes dois prompts (cole o conteúdo entre os marcadores
   ``` exatamente como estão nos arquivos-fonte, sem resumir):
   - O prompt dos ADRs (prompts/02-adrs.md) — exemplo de prompt para
     decisão pontual.
   - O prompt do Tracker (prompts/06-tracker.md) — exemplo de prompt
     para o documento transversal de rastreabilidade, com as regras de
     cobertura e a regra anti-alucinação.
   Antes de cada bloco, uma frase curta dizendo por que esse prompt foi
   escrito daquele jeito (ex.: "o prompt dos ADRs exige o ID do
   inventário ou timestamp para cada decisão, para impedir que a IA
   generalize uma decisão pontual sem origem rastreável").

5. **Iterações e ajustes**: descreva estes momentos REAIS de correção
   (não invente outros; pode resumir, mas não troque os fatos):
   - **Setup do repositório**: o fork criado pelo botão "Fork" do
     GitHub não era reconhecido como fork de fato pela API do GitHub
     e, mais importante, deixaria o trabalho visível na rede pública
     de forks do repositório base, exposto a outros alunos. Foi preciso
     abandonar essa abordagem, excluir o repositório, e recriar um
     repositório independente e privado, migrando o conteúdo já
     clonado para ele.
   - **Contradição entre RFC e FDD sobre o estado da outbox**: a
     transcrição menciona tanto um estado "falhou" dentro da própria
     outbox quanto uma tabela de DLQ separada, sem a reunião resolver
     explicitamente a diferença entre os dois. O RFC sinalizou essa
     inconsistência explicitamente (seção de notas de consistência) em
     vez de escolher um lado silenciosamente; o FDD resolveu a
     ambiguidade com uma premissa própria (P-06: FAILED é estado
     transitório dentro da outbox, e só a falha permanente depois de
     esgotadas as tentativas vai para a DLQ), marcada como escolha do
     FDD e não como decisão da reunião.
   - **Lacunas encontradas pelo Tracker**: ao gerar o Tracker como
     auditoria final, dois itens do FDD (um endpoint de detalhe de
     webhook e um risco de SSRF por URL interna) apareceram como itens
     sem origem rastreável na transcrição. Em vez de forçar uma citação
     inexistente, esses itens foram formalizados como novas premissas
     do FDD (P-31 e P-32), marcadas explicitamente como propostas deste
     documento e não como consenso da reunião — e o FDD foi atualizado
     para refletir isso antes do commit final.
   - Informe quantas iterações principais (gerar → revisar → ajustar
     prompt ou documento → regerar) foram necessárias no total, pela
     sua lembrança do processo.

6. **Como navegar a entrega**: liste o caminho de cada arquivo entregue
   e uma ordem sugerida de leitura para o avaliador, nesta ordem (é a
   ordem de "altura" dos documentos, do mais estratégico ao mais
   operacional, com o Tracker como verificação transversal):
   1. docs/PRD.md
   2. docs/RFC.md
   3. docs/adrs/ (os 7 ADRs)
   4. docs/FDD.md
   5. docs/TRACKER.md
   Mencione também que a pasta prompts/ (fora de entregavel/) guarda
   todos os prompts usados, para quem quiser ver o processo completo.

REGRAS
- Não repita o conteúdo técnico do PRD/RFC/FDD/ADRs/Tracker aqui — este
  README é sobre o PROCESSO de produção, não sobre a feature em si.
- Pode manter uma seção curta ou um link no final referenciando o
  enunciado original em descricao/enunciado.md, mas o foco é a sua
  jornada de produção.
- Tom narrativo em primeira pessoa (sou eu, o autor do trabalho,
  descrevendo o que fiz), não um manual genérico de boas práticas.

SAÍDA
Gere o conteúdo completo de entregavel/README.md pronto para salvar no
repositório, substituindo o conteúdo atual (o enunciado).
```
