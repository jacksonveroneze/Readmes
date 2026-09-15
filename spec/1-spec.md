# Spec: Interface Refit — Proventos Futuros (QK6)

## Problema / Porquê
O BFF consome proventos futuros do FD9; precisa passar a consumir do QK6.
Primeiro artefato: a interface Refit que representa esse endpoint do QK6.

## Comportamento esperado
Interface Refit tipada que espelha o contrato HTTP do endpoint de proventos
futuros do QK6.
- Entrada: <params da chamada — preencher>
- Saída: <DTO de resposta — preencher>
- Fluxo: <verbo> <rota> → desserializa no DTO de resposta

## Contrato (= inputs da skill)   ◄── coração desta spec
- Rota: <preencher>
- Verbo: <GET / POST / ...>
- Parâmetros: <path / query / header — preencher>
- Request body (se houver): <...>
- Retorno (response): <...>
- Headers / auth: <...>

## Critérios de aceite
- [ ] Interface compila e reflete exatamente rota, verbo, params e retorno do QK6
- [ ] DTOs de request/response batem com o contrato real do QK6
- [ ] Nomenclatura segue o padrão do projeto

## Regras e restrições
- Somente proventos FUTUROS (não recebidos)
- Não altera o consumo de proventos recebidos (permanece no FD9)

## Fora de escopo
- Repository que consome a interface (Spec B)
- Troca do fluxo FD9→QK6 dentro do BFF
- Proventos recebidos
- Remover/depreciar a interface FD9 atual

## Gate 1 — aprovação
[ ] Seção "Contrato" completa → pronta para alimentar a skill