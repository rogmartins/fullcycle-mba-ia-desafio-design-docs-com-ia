# Prompt — Geração do PRD (Etapa 5 do roteiro)

> Usado na Etapa 5 do roteiro (`descricao/roteiro.md`). Depende do RFC (`entregavel/docs/RFC.md`), do FDD (`entregavel/docs/FDD.md`), dos 7 ADRs (`entregavel/docs/adrs/`) e do inventário (`prompts/01-exploracao-inicial.resultado.md`). Anexe todos esses arquivos, mais a transcrição, ao rodar este prompt.

```
Você vai atuar como Product Manager sênior escrevendo o PRD (Product
Requirement Document) da feature "Sistema de Webhooks de Notificação
de Pedidos". Você JÁ TEM em mãos:
- o inventário validado (prompts/01-exploracao-inicial.resultado.md);
- os 7 ADRs aceitos (entregavel/docs/adrs/);
- o RFC (entregavel/docs/RFC.md);
- o FDD (entregavel/docs/FDD.md).

Diferente desses três, o PRD opera em nível de PRODUTO/NEGÓCIO, não de
arquitetura ou implementação. Ele responde "por que e o quê", não
"como". Use os documentos acima como fonte PRINCIPAL — a essa altura,
é praticamente uma consolidação do que já foi decidido, elevado ao
nível de negócio. Não repita contratos de API, matriz de erros ou
detalhe de fluxo do FDD; não repita o detalhamento arquitetural do
RFC. Traga só o que importa para quem decide produto: motivação,
público, escopo, requisitos, métricas, riscos de negócio.

Não invente nenhum requisito, métrica, risco ou item de escopo que não
seja rastreável ao inventário, aos ADRs, ao RFC/FDD ou à
TRANSCRICAO.md.

ESTRUTURA OBRIGATÓRIA DE docs/PRD.md
1. **Resumo e contexto da feature**: 1-2 parágrafos. Quem pediu
   (Atlas Comercial, MaxDistribuição, Nova Cargo), o problema atual
   (polling em GET /orders) e a solução em uma frase.
2. **Problema e motivação**: o custo do problema atual para o cliente
   e para o negócio (risco de churn da Atlas, [09:00] Marcos; RNF11).
3. **Público-alvo e cenários de uso**: quem usa cada parte da feature
   — o cliente B2B que recebe notificações, o usuário operador/admin
   que configura webhooks e faz replay de DLQ. Descreva 2-3 cenários
   de uso concretos (ex.: "cliente cadastra endpoint e recebe
   notificação quando pedido é despachado"; "admin reprocessa entrega
   que falhou por indisponibilidade do cliente").
4. **Objetivos e métricas de sucesso**: objetivos de negócio/produto
   (não confundir com os "objetivos técnicos" do FDD). PRECISA incluir
   PELO MENOS 1 objetivo com métrica E meta quantitativa — pode
   derivar de RNF1 (ex.: "95% dos eventos entregues em menos de 10s
   da mudança de status") ou de RNF11 (prazo de entrega).
5. **Escopo**:
   - **Incluso**: liste em linguagem de produto (não de código) as
     capacidades entregues — cadastro/gestão de webhooks, notificação
     automática, histórico de entregas, segurança da entrega, retry e
     recuperação de falhas.
   - **Fora de escopo**: a seção PRECISA listar EXPLICITAMENTE PELO
     MENOS 2 itens descartados ou adiados na reunião. Use a seção
     "f" do inventário e a seção 5 do RFC: email de alerta, rate
     limiting de saída, dashboard visual, arquivamento, múltiplos
     workers, endurecimento de autorização, webhooks inbound.
6. **Requisitos funcionais**: PRECISA listar PELO MENOS 8 requisitos
   funcionais discutidos na reunião. Traduza RF1-RF10 do inventário
   para linguagem de produto (o que o sistema faz, não como), mas
   mantenha a referência de origem (RF do inventário + timestamp).
7. **Requisitos não funcionais**: traduza as RNFs relevantes para
   linguagem de produto (latência percebida, segurança da secret,
   disponibilidade), referenciando a origem.
8. **Decisões e trade-offs principais**: resuma em 4-6 bullets as
   decisões de maior impacto para o produto (não repita as 7 ADRs
   inteiras) — ex.: at-least-once implica o cliente tratar
   duplicatas; prazo de 3 sprints; replay manual em vez de automático.
   Link para os ADRs correspondentes.
9. **Dependências**: times/pessoas envolvidos (revisão de segurança
   da Sofia antes do deploy, RNF10; documentação no portal do
   desenvolvedor, responsabilidade do Marcos) e dependências técnicas
   de alto nível (sem entrar em código).
10. **Riscos e mitigação**: a seção PRECISA incluir PELO MENOS 2
    riscos, cada um com **probabilidade**, **impacto** e
    **mitigação** explícitos (tabela ou bullets estruturados). Use
    como base os riscos de negócio do RFC (seção 6) e do FDD (seção
    11), elevados ao nível de produto — ex.: atraso na entrega da
    Atlas (prazo apertado + revisão de segurança obrigatória), ou
    responsabilidade do cliente na deduplicação gerando reclamação de
    suporte.
11. **Critérios de aceitação**: critérios de produto, verificáveis
    por quem não lê código (ex.: "cliente cadastrado recebe
    notificação assinada em até X segundos após mudança de status
    visível no painel"), distintos dos critérios técnicos do FDD.
12. **Estratégia de testes e validação**: em nível de produto — como
    validar com os clientes-piloto (Atlas, MaxDistribuição, Nova
    Cargo), não a estratégia de testes automatizados (isso é FDD).

REGRAS
- Toda afirmação remete a um ID do inventário (D/RF/RNF), a um ADR,
  ao RFC/FDD, ou a timestamp+falante da transcrição.
- Não desça a nível de contrato de API, payload ou matriz de erros.
- Se um requisito do inventário não tiver "dono" claro de onde vem,
  marque como premissa em vez de apresentá-lo como consenso, seguindo
  o mesmo princípio usado no FDD.

SAÍDA
Gere o conteúdo completo de docs/PRD.md pronto para salvar no
repositório.
```
