# Prompt — Geração dos ADRs (Etapa 2 do roteiro)

> Usado na Etapa 2 do roteiro (`descricao/roteiro.md`). Depende do inventário gerado em `prompts/01-exploracao-inicial.resultado.md` — não rode este prompt sem anexar aquele arquivo (ou colar seu conteúdo) junto com o código e a transcrição.

```
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
