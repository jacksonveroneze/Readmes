# Roteiro Inicial de Entrevista sobre IA

Este roteiro tem como objetivo avaliar conhecimentos introdutórios e práticos sobre Inteligência Artificial aplicada ao desenvolvimento de software.

A proposta não é exigir conhecimento profundo em Machine Learning, mas validar se o candidato entende os principais conceitos, sabe aplicar IA com responsabilidade e possui senso crítico para uso no dia a dia técnico.

---

## 1. Fundamentos de LLM

### Objetivo

Validar se o candidato entende o conceito base por trás de ferramentas como ChatGPT, Claude, Gemini, GitHub Copilot e outros assistentes baseados em IA.

### Assuntos

- O que é um LLM
- Prompt
- Tokens
- Janela de contexto
- Alucinação
- Limitações de um LLM
- Diferença entre modelo, família de modelo e fornecedor

### Profundidade esperada

O candidato não precisa saber detalhes matemáticos ou de treinamento de modelos, mas deve saber explicar que um LLM recebe uma entrada, considera o contexto disponível e gera uma resposta provável.

Também é importante que entenda que o modelo pode errar, inventar informações ou responder com segurança mesmo quando está incorreto.

---

## 2. Prompt Engineering

### Objetivo

Avaliar se o candidato sabe usar IA de forma intencional, estruturando bons comandos em vez de apenas fazer perguntas genéricas.

### Assuntos

- Como estruturar um bom prompt
- Papel da IA
- Contexto
- Objetivo
- Restrições
- Formato esperado
- Uso de exemplos no prompt
- Técnicas básicas para melhorar respostas
- Validação das respostas geradas pela IA

### Profundidade esperada

O candidato deve entender que um prompt bem estruturado reduz ambiguidade e aumenta a qualidade da resposta.

Também deve demonstrar que sabe validar a resposta gerada, principalmente quando envolve código, arquitetura, segurança ou decisões técnicas.

---

## 3. Agents

### Objetivo

Avaliar se o candidato entende a diferença entre um chatbot simples e um sistema baseado em IA capaz de executar tarefas com apoio de ferramentas.

### Assuntos

- O que é um agent
- Diferença entre chatbot simples e agent
- Uso de ferramentas por agents
- Execução de tarefas em etapas
- Quando usar agents
- Quando evitar agents
- Riscos de autonomia excessiva

### Profundidade esperada

O candidato deve entender que um agent não apenas responde texto.

Um agent pode usar ferramentas, consultar APIs, executar etapas, tomar decisões intermediárias e produzir uma saída final com base em um objetivo.

Também é importante que o candidato entenda que agents adicionam complexidade e precisam de controle, rastreabilidade e validação.

---

## 4. Skills

### Objetivo

Avaliar se o candidato entende como padronizar capacidades reutilizáveis para tarefas com IA.

### Assuntos

- O que são skills
- Como skills ajudam a padronizar tarefas
- Diferença entre prompt, skill e agent
- Uso de skills para revisão de código
- Uso de skills para geração de documentação
- Uso de skills para criação de projetos
- Reaproveitamento de instruções, scripts e templates

### Profundidade esperada

O candidato deve entender skill como uma capacidade reutilizável, normalmente composta por instruções, contexto, arquivos, scripts ou procedimentos para executar bem uma tarefa específica.

Exemplos esperados:

- Skill para revisar código C#
- Skill para criar ADRs
- Skill para gerar documentação
- Skill para criar estrutura inicial de um projeto
- Skill para padronizar análise de logs

---

## 5. MCP

### Objetivo

Verificar se o candidato conhece o conceito de integração padronizada entre aplicações de IA e sistemas externos.

### Assuntos

- O que é MCP
- Para que serve
- Como conecta IA a sistemas externos
- Diferença entre MCP, API comum e tool/function calling
- Exemplos de uso: GitHub, banco de dados, arquivos e APIs internas
- Quando MCP faz sentido

### Profundidade esperada

O candidato deve entender MCP como um protocolo para expor ferramentas, dados e sistemas externos para aplicações de IA de forma padronizada.

Não é necessário conhecer implementação detalhada, mas é importante entender o papel do MCP como uma camada de integração entre modelos, agents e ferramentas externas.

---

## 6. Guardrails, Segurança e Privacidade

### Objetivo

Avaliar maturidade do candidato em relação a controle, segurança, privacidade e uso responsável de IA.

### Assuntos

- O que são guardrails
- Por que guardrails são importantes
- Validação de entrada
- Validação de saída
- Segurança, privacidade e conformidade
- Prevenção de vazamento de dados sensíveis
- Riscos ao enviar dados sensíveis para LLMs
- Dados internos e confidenciais
- Código proprietário
- Logs com informações sensíveis
- Prompt injection
- Data leakage
- Controle de acesso
- Sanitização de dados
- Uso responsável de ferramentas públicas de IA

### Profundidade esperada

O candidato deve entender que guardrails são regras, validações e controles usados para reduzir riscos no uso de IA.

Também deve demonstrar consciência de que nem todo dado pode ser enviado para ferramentas externas, especialmente dados sensíveis, confidenciais, estratégicos ou protegidos por regras de compliance.

---

## 7. RAG

### Objetivo

Verificar se o candidato entende como conectar LLMs a dados externos para gerar respostas mais contextualizadas e confiáveis.

### Assuntos

- O que é RAG
- Diferença entre LLM puro e LLM com RAG
- Componentes básicos de uma arquitetura RAG
- Base de conhecimento
- Chunking em nível introdutório
- Embeddings
- Busca vetorial
- Recuperação de contexto
- Geração da resposta com base nos documentos recuperados
- Riscos de RAG
- Dados desatualizados
- Documentos irrelevantes
- Contexto ruim
- Falta de controle de acesso sobre os documentos

### Profundidade esperada

O candidato deve entender que RAG não é um modelo, mas uma arquitetura que combina busca, dados externos e LLM.

Fluxo esperado em alto nível:

> Pergunta do usuário  
> → busca de informações relevantes  
> → envio do contexto recuperado para o LLM  
> → geração da resposta com base no contexto

O candidato também deve entender que RAG ajuda a reduzir alucinações, mas não elimina completamente o risco de respostas incorretas.

---

## 8. Embeddings e Busca Vetorial

### Objetivo

Validar entendimento básico sobre como textos podem ser representados e comparados semanticamente.

### Assuntos

- O que são embeddings
- Diferença entre LLM e embedding model
- Transformação de texto em vetor
- Similaridade semântica
- Busca vetorial
- Uso de embeddings em RAG
- Uso de embeddings em bases de conhecimento

### Profundidade esperada

O candidato não precisa conhecer álgebra linear ou detalhes matemáticos.

O esperado é que consiga explicar que embeddings transformam textos em representações numéricas, permitindo encontrar conteúdos semanticamente parecidos com uma pergunta ou documento.

---

## 9. IA Aplicada ao Desenvolvimento de Software

### Objetivo

Entender se o candidato sabe aplicar IA no dia a dia técnico de forma prática, produtiva e responsável.

### Assuntos

- Geração de código
- Revisão de código
- Criação de testes unitários
- Geração de documentação
- Apoio em troubleshooting
- Análise de logs
- Apoio em arquitetura
- Refatoração assistida
- Criação de ADRs
- Uso em IDEs como VS Code, Cursor, GitHub Copilot ou Claude Code

### Profundidade esperada

O candidato deve saber usar IA como apoio, e não como substituto completo do julgamento técnico.

É importante que demonstre senso crítico, revisão humana, validação com testes e cuidado com código gerado automaticamente.

---

# Organização Final da Entrevista

| Bloco | Tema | Objetivo |
|---|---|---|
| 1 | LLM e Prompt Engineering | Validar fundamentos |
| 2 | Agents, Skills e MCP | Validar automação e integração |
| 3 | Guardrails, Segurança e Privacidade | Validar maturidade e responsabilidade |
| 4 | RAG, Embeddings e Busca Vetorial | Validar arquitetura com dados externos |
| 5 | IA aplicada ao desenvolvimento de software | Validar uso prático no dia a dia |

---

# Critério Geral de Avaliação

O candidato não precisa ser especialista em IA, mas deve demonstrar:

- Clareza conceitual
- Capacidade de explicar com simplicidade
- Uso prático no contexto de desenvolvimento de software
- Consciência sobre limitações dos modelos
- Preocupação com segurança e privacidade
- Capacidade de validar respostas geradas por IA
- Senso crítico para não usar IA de forma ingênua
- Entendimento de que IA deve apoiar, não substituir completamente a engenharia

---

# Níveis de Maturidade Esperados

| Nível | Característica |
|---|---|
| Básico | Conhece LLM, prompt, alucinação e usa IA no dia a dia |
| Intermediário | Entende RAG, embeddings, agents, MCP e guardrails em nível conceitual |
| Avançado | Consegue discutir arquitetura, riscos, integração, segurança e uso em produção |
