# ADR-004 — Autenticação de origem com HMAC-SHA256 e secret por endpoint

- **Status:** Aceito
- **Data da decisão:** Reunião técnica de quinta-feira, 09:00. A transcrição não registra a data de calendário.
- **Decisores:** Sofia (Eng. de Segurança), Larissa (Tech Lead), Diego, Bruno
- **Decisões do inventário cobertas:** D9

## Contexto

Os webhooks levam dados de pedidos para endpoints fora da infraestrutura da plataforma. "O cliente tem que conseguir validar que a requisição veio realmente da gente, e que ninguém adulterou o payload no meio" ([09:19] Sofia).

Já aconteceu de um cliente vazar a secret no log da própria aplicação ([09:22] Diego). Restrições relacionadas:

- secret única por endpoint (RNF7);
- TLS obrigatório (RNF4);
- revisão de segurança de pelo menos 2 dias úteis antes do deploy, com foco em "HMAC e geração de secret" ([09:46] Sofia; RNF10).

No código atual não há biblioteca de HMAC em `package.json`. O logger tem uma lista de `redact` (`src/shared/logger/index.ts:4-11`) que cobre `password`, `token` e `authorization`, mas **não cobre `secret`**.

## Decisão

- Cada requisição é assinada com **HMAC-SHA256 sobre o corpo**, e a assinatura vai no header `X-Signature`.
- A secret é **única por endpoint de webhook**, gerada pela plataforma e devolvida na criação (RF2, [09:31] Marcos).
- A secret pode ser **rotacionada via API**. Depois da rotação, a secret antiga **continua válida em paralelo por 24 horas** e então expira.

Origens da decisão:

- [09:20] Sofia ("Padrão é HMAC […] manda a assinatura num header tipo X-Signature"; "SHA-256").
- [09:21] Sofia ("cada endpoint de webhook do cliente tem que ter uma secret única"; rotação com 24h em paralelo).
- [09:22] Sofia: "Decidido: HMAC-SHA256 sobre o corpo do request, secret por endpoint, suporte a rotação com grace period de 24h."

URL obrigatoriamente `https` ([09:23] Sofia) é um requisito complementar. A própria Sofia disse que "nem é decisão arquitetural, é só uma validação no schema Zod", então o detalhe fica para o FDD.

## Alternativas Consideradas

1. **Uma secret global da plataforma para todos os clientes.** Descartada. "Não é uma secret global da nossa plataforma. Senão se vaza uma, vaza tudo" ([09:21] Sofia).
2. **Assinatura assimétrica (par de chaves, com o cliente validando por chave pública).** *Alternativa plausível proposta neste ADR; não foi discutida na reunião.* Ela eliminaria o segredo compartilhado, mas exigiria distribuir chaves e usar bibliotecas menos comuns nos clientes. O critério de Sofia aponta para HMAC: "HMAC-SHA256 é o padrão de mercado, todo cliente sério tem biblioteca pra isso" ([09:20]).

## Consequências

### Positivas

- **Padrão de mercado**, fácil de verificar do lado do cliente ([09:20] Sofia).
- **Vazamento contido.** A secret vazada de um endpoint compromete só aquele endpoint (RNF7).
- **Rotação sem parada.** O grace period de 24h dá tempo ao cliente "de migrar os sistemas dele" ([09:21] Sofia).

### Negativas

- **A secret precisa ser recuperável pelo worker** para calcular o HMAC. Ela não pode ser guardada como hash de mão única, como `passwordHash` em `User`. Isto é inferência técnica. A forma de armazenar não foi discutida e deve entrar na revisão de segurança (RNF10).
- **A rotação exige guardar duas secrets válidas com expiração** durante as 24h, o que aumenta o modelo de dados e a lógica de assinatura.
- **Risco de vazamento nos logs da própria plataforma.** A lista `redact` do Pino (`src/shared/logger/index.ts:4-11`) não mascara `secret` nem `X-Signature` e precisará ser estendida. O inventário aponta isso nos cruzamentos.
- **O cliente precisa implementar a verificação.** A segurança depende de ele de fato validar a assinatura.

### Trade-off explícito

Ganha isolamento de vazamento por endpoint e rotação sem parada. Paga com armazenamento de segredo recuperável e com a complexidade de manter duas secrets simultâneas.

### Pontos em aberto (não decididos na reunião)

- **A2:** o HMAC é calculado **só sobre o corpo** ([09:22] Sofia), e o `X-Timestamp` ([09:44] Diego) **não entra na assinatura**. A proteção contra replay sugerida para o cliente, portanto, não é garantida criptograficamente.
- **A3:** o formato do `X-Signature` (hex ou base64, prefixo do tipo `sha256=`) não foi discutido.
- **A5:** durante o grace period, não se decidiu se o envio é assinado com as duas secrets ou só com a nova.

Esses pontos devem ser levados à revisão de segurança da Sofia, e não decididos neste ADR.

## Referências

- `src/shared/logger/index.ts:4-11`: lista de `redact` atual.
- `prisma/schema.prisma:28`: `passwordHash` como contraste (hash de mão única).
- Inventário: D9; RF2, RF5; RNF4, RNF7, RNF10; A2, A3, A5.
