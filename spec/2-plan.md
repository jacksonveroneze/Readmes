# Plano: Interface Refit — Proventos Futuros (QK6)
> ref: 1-spec.md

## Abordagem
Gerar a interface pela skill de Refit, alimentando rota + params + retorno do
contrato QK6. Sem código manual além de ajuste de nome/local.

## Componentes afetados
- Criar: interface Refit `I<...>` + DTOs de request/response
- Alterar: nada nesta etapa (DI e repository ficam para depois)

## Dependências
- Skill de geração de Refit
- Documentação / contrato real do endpoint QK6
- Refit já configurado no projeto

## Decisões e riscos (ADR-lite)
- Decisão: gerar via skill — Porque: padroniza e acelera — Descartado: escrever à mão
- Decisão: nome da interface `I<...>` — <definir>
- Risco: doc do QK6 divergir da resposta real — Mitigação: validar contra resposta viva

## Estratégia de teste
- Compilação + conferência do contrato gerado vs. QK6
- (Integração real fica na Spec B, com o repository)

## Gate 2 — aprovação
[ ] Inputs da skill revisados; nome e local definidos