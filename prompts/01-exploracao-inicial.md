# Prompt — Exploração inicial (código + transcrição)

> Usado na Etapa 1 do roteiro (`descricao/roteiro.md`), item "Contextualização com IA", para mapear requisitos, decisões, descartes e ganchos com o código antes de produzir qualquer documento final.

```
Você vai atuar como engenheiro de software sênior fazendo o levantamento
inicial de uma nova feature em um projeto existente. NÃO gere nenhum
documento final ainda (nada de PRD, RFC, FDD ou ADR) — o objetivo desta
etapa é só produzir um INVENTÁRIO ESTRUTURADO de achados, que servirá de
matéria-prima rastreável para os documentos que virão depois.

CONTEXTO DO PROJETO
Este é um Order Management System (OMS) em Node.js + TypeScript, com
banco MySQL via Prisma, contendo módulos de autenticação, usuários,
clientes, produtos e pedidos. O ciclo de vida do pedido tem máquina de
estados controlada, controle transacional de estoque e auditoria de
mudanças de status. A aplicação atualmente NÃO tem nenhum mecanismo de
notificação externa, eventos, filas ou webhooks.

Uma reunião técnica (arquivo TRANSCRICAO.md, ~55 min, formato
[hh:mm] Nome: fala) decidiu construir um Sistema de Webhooks de
Notificação de Pedidos para preencher essa lacuna. Você tem acesso ao
código-fonte completo e a essa transcrição, que estão, respectivamente, em: `../entregavel/src`  e `../entregavel/TRANSCRICAO.md`

PARTE 1 — EXPLORAÇÃO DO CÓDIGO
Mapeie e descreva, com caminho de arquivo exato para cada item:
1. Estrutura modular da aplicação (como os módulos de auth, usuários,
   clientes, produtos e pedidos estão organizados).
2. A máquina de estados do ciclo de vida do pedido: onde está definida,
   quais são os estados e transições possíveis.
3. O método/fluxo `changeStatus` (ou equivalente): onde vive, como
   funciona a transação, o que ele já faz hoje, e quais seriam os
   pontos naturais de extensão para disparar eventos de webhook.
4. As classes de erro existentes e o padrão de códigos de erro usado
   hoje no projeto (formato, convenção de nomenclatura).
5. O middleware `requireRole`: como funciona, onde é usado.
6. O error middleware centralizado: como trata e formata erros.
7. O logger Pino: como é configurado e usado hoje (níveis, formato,
   onde é chamado).
8. Qualquer outro padrão arquitetural relevante (camadas, injeção de
   dependência, convenções de nomenclatura, testes existentes) que uma
   feature nova precisaria respeitar.

Para cada item, cite o(s) caminho(s) de arquivo real(is) — nunca
generalize sem apontar onde no código isso está.

PARTE 2 — LEITURA DIRIGIDA DA TRANSCRIÇÃO
Leia TRANSCRICAO.md por completo e extraia, cada item com timestamp e
falante (formato [hh:mm] Nome), organizados nas seguintes categorias:

a) DECISÕES FECHADAS — algo que o grupo decidiu explicitamente fazer
   de determinada forma (ex.: padrão de arquitetura, protocolo,
   mecanismo de garantia).
b) REQUISITOS FUNCIONAIS — o que a feature precisa fazer, do ponto de
   vista de comportamento observável.
c) REQUISITOS NÃO FUNCIONAIS / RESTRIÇÕES — desempenho, segurança,
   compatibilidade, limites técnicos ou de negócio mencionados.
d) GANCHOS COM O CÓDIGO EXISTENTE — menções diretas ou indiretas a
   componentes do código atual que a feature vai tocar ou reusar.
e) DESCARTADO — ideias ou abordagens explicitamente rejeitadas pelo
   grupo, com o motivo do descarte, se mencionado.
f) ADIADO / FORA DE ESCOPO NESTA FASE — algo cogitado mas explicitamente
   empurrado para uma fase futura.
g) DETALHES TÉCNICOS SECUNDÁRIOS — menções técnicas específicas
   (formatos, nomes de headers, valores de timeout, nomes de tabelas
   etc.) que não são decisão arquitetural em si, mas são detalhe de
   implementação.

Regras importantes:
- Cada item das categorias (a) a (g) PRECISA ter o timestamp e o nome
  de quem falou. Se você não consegue localizar isso com precisão na
  transcrição, não invente — sinalize como "sem localização precisa"
  em vez de aproximar.
- Não infira requisitos que não foram ditos. Se um participante
  "sugeriu" algo mas não houve fechamento do grupo, classifique como
  "levantado, não decidido" e não como decisão fechada.
- Separe claramente o que foi DESCARTADO/ADIADO do que foi DECIDIDO —
  essa é a distinção mais importante deste levantamento.

PARTE 3 — SAÍDA ESPERADA
Organize a resposta em duas seções (Código e Transcrição), usando os
subitens acima como subtítulos, em formato de lista. Ao final, monte
uma terceira seção "Cruzamentos" apontando explicitamente onde um
requisito ou decisão da transcrição se conecta a um componente do
código já mapeado (ex.: "a decisão de garantir at-least-once com
X-Event-Id [xx:xx] Fulano se conecta à tabela de auditoria já existente
em src/...").

Este inventário será a base para depois produzirmos ADRs, RFC, FDD, PRD
e o Tracker de rastreabilidade — por isso a exigência de citar sempre
a origem exata (timestamp+nome ou caminho de arquivo) em cada item.
```
