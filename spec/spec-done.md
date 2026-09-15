# Spec: Interface Refit — Proventos Futuros (QK6)

## Problema / Porquê
O BFF consome proventos futuros do FD9; precisa passar a consumir do QK6.
Primeiro artefato: a interface Refit que espelha o endpoint de proventos do QK6.

## Comportamento esperado
Interface Refit tipada que representa o contrato HTTP do endpoint de proventos
do QK6, retornando a lista de proventos futuros de um cliente.
- Entrada: client_id (identificador do cliente)
- Saída: lista de proventos futuros (ticker, empresa, valor, quantidade)
- Fluxo: GET /api/v1/proventos?client_id=<id> → desserializa em List<DTO>

## Contrato (= inputs da skill)   ◄── coração desta spec
- Base URL: https://10.0.0.195:7000   (vai na DI, NÃO na interface)
- Rota (relativa): /api/v1/proventos
- Verbo: GET   (assumido — confirmar)
- Parâmetros:
    - client_id : int  (query, obrigatório)
- Request body: nenhum
- Retorno: array de objetos —
    | Campo JSON  | Tipo    | Propriedade C# sugerida | Observação            |
    |-------------|---------|-------------------------|-----------------------|
    | client_id   | int     | ClientId                | corrigir chave (typo) |
    | ticker      | string  | Ticker                  |                       |
    | Empresa     | string  | Empresa                 | PascalCase na origem  |
    | Valor       | decimal | Valor                   | dinheiro → decimal    |
    | Quantidade  | int     | Quantidade              |                       |
- Headers / auth: nenhum informado (confirmar se há token/cert)

## Critérios de aceite
- [ ] Interface compila e reflete rota, verbo, query param e retorno do QK6
- [ ] DTO usa JsonPropertyName para casar o casing misto da origem
- [ ] Retorno tipado como List<ProventoFuturoResponse>
- [ ] Nomenclatura segue o padrão do projeto

## Regras e restrições
- Somente proventos FUTUROS (recebidos permanecem no FD9)
- Interface carrega apenas rota relativa; base URL fica na config

## Fora de escopo
- Repository que consome a interface (Spec B)
- Registro no DI / config do HttpClient (base URL, cert self-signed)
- Troca do fluxo FD9→QK6 no BFF
- Proventos recebidos e a interface FD9 atual

## Gate 1 — aprovação
[ ] Seção "Contrato" completa e confirmada → pronta para alimentar a skill