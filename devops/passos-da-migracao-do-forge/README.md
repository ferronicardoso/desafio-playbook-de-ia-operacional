---
nome: Passos da migração do Forge
descricao: Converte o diagnóstico de um pipeline em lote em uma sequência ordenada de etapas incrementais de migração para arquitetura orientada a eventos.
versao: 1.0.0
tags:
  - migracao-incremental
  - arquitetura-orientada-a-eventos
  - coexistencia
  - pipeline-de-dados
inputs:
  - nome: diagnostico
    descricao: Diagnóstico estruturado do estado atual do pipeline em lote, com fatos, componentes, dependentes, pontos frágeis, riscos, invariantes, lacunas e premissas identificados.
---

# Passos da migração do Forge

## Objetivo

Converter o diagnóstico de um pipeline em lote em uma sequência ordenada de etapas incrementais de migração para uma arquitetura orientada a eventos. É o segundo elo de uma cadeia de três prompts: recebe a saída de [`diagnostico-do-forge`](../diagnostico-do-forge/) e entrega a entrada de [`plano-de-migracao-do-forge`](../plano-de-migracao-do-forge/).

A partir do diagnóstico, o prompt herda o contexto necessário (fatos, componentes, dependentes, pontos frágeis, invariantes e riscos), registra lacunas e premissas, define a estratégia de coexistência entre lote e eventos e detalha as etapas. Cada etapa traz objetivo, pré-requisitos, o que muda e o que não muda, coexistência, situação de cada dependente, pontos frágeis e riscos tratados, invariantes protegidos, critério de avanço e ponto de retorno. A saída termina com uma matriz de cobertura de invariantes, pontos frágeis, riscos e dependentes por etapa.

O prompt não detalha ações operacionais, comandos nem responsáveis: isso é do elo 3. Ele proíbe virada única, exige que a desativação do lote seja uma etapa própria depois de uma etapa de execução em paralelo com reconciliação e não inventa limiares: o que falta vira lacuna.

## Quando usar

- Depois de obter um diagnóstico do estado atual do pipeline, para transformar o diagnóstico em etapas de migração que possam ser revistas e contestadas antes de virar plano de execução.
- Para decidir a ordem de migração dos dependentes e o que fica como fonte de verdade em cada fase, com ponto de retorno declarado por etapa.
- Como segunda etapa da cadeia de migração, para que o elo seguinte receba etapas com critérios de avanço e pontos de retorno.

## Exemplo de uso

1. Executar o elo 1 ([`diagnostico-do-forge`](../diagnostico-do-forge/)) e guardar a saída completa. Se houver barras de cópia (`\~`, `\-`), removê-las.
2. Copiar o conteúdo de `prompt.md` (sem o frontmatter) e substituir `{{diagnostico}}` pela saída do elo 1. É o único parâmetro e é obrigatório. O diagnóstico deve conter as seções 1 (fatos), 3 (dependentes) e 6 (invariantes).
3. Executar em uma conversa ou chamada nova.
4. Ler primeiro a seção 2 (lacunas e premissas) e a seção 3 (coexistência), porque as etapas dependem delas. Conferir os pré-requisitos e os pontos de retorno de cada etapa e a matriz de cobertura.
5. A saída completa é a entrada do elo 3.

Um exemplo de execução com saída real está na seção [Execução](#execução).

## Limitações conhecidas

- A qualidade da saída depende do diagnóstico. Um diagnóstico sem a seção de dependentes ou de invariantes faz o prompt responder com a frase de insuficiência.
- Todos os critérios de avanço saíram com "limiar a definir (L13)", porque o diagnóstico não traz números de aceitação e o prompt proíbe inventar valores. As janelas de observação usam eventos do cenário (um ciclo de 60 minutos, uma execução completa de madrugada), mas o plano não traz nenhum limiar mensurável.
- A lista de invariantes herdada tem um único item (`I1`), que reúne a continuidade de três dependentes. Todas as etapas protegem `I1`, e a linha da matriz de cobertura dele é trivial.
- A ordem de migração dos dependentes (Sentinel, Cerebro, billing) é justificada apenas por critério de risco, sem dados que a sustentem. Começar pelo Sentinel, que é o produto central, é uma escolha discutível.
- O ponto de retorno da etapa de desativação do lote (E7) é ambíguo: diz que a perda da capacidade de replay ou reativação impede o rollback garantido, mas a coexistência da etapa prevê a capacidade de retorno "até liberação específica", e o plano não define essa liberação. O horizonte de recuperação do lote não saiu como lacuna.
- Os pré-requisitos de algumas etapas não coincidem com a tabela de lacunas: a L3 afeta a E4 e a L14 afeta a E7, mas não constam como pré-requisito delas (a E1 e a E3 já as resolveriam). O elo 3 monta os bloqueios a partir de uma das duas fontes.
- As seções 2 e 3 passaram ligeiramente do limite (318 contra 300 e 202 contra 200 palavras). O limite total (2.400 palavras) foi respeitado, com cerca de 2.136 palavras, contadas por tokens com letra ou número.
- Quase todos os itens saem rotulados como (julgamento), o que é esperado, porque as etapas são decisões do modelo.
- O resultado vem de duas execuções com um único modelo. A primeira usou uma versão anterior do prompt; a saída registrada abaixo é a da versão final.

## Execução

| Item | Valor |
|---|---|
| Modelo | GPT 6 Sol |
| Identificador na API | `gpt-6-sol` |
| Esforço de raciocínio | médio |

O único parâmetro, `diagnostico`, foi preenchido com a saída do elo 1 (seção Execução de [`diagnostico-do-forge`](../diagnostico-do-forge/)), sem alteração além da remoção das barras de cópia. A execução foi feita em um chat novo e temporário, sem acesso à memória.

### Saída

A saída foi copiada sem edição, exceto a remoção das barras de cópia do ChatGPT (`\~`, `\-`, `\*`).

```markdown
# PASSOS DA MIGRAÇÃO

## 1. Contexto herdado

### Fatos essenciais

- F1: O Forge utiliza um job em cron chamado "forge-batch-ingest", que acorda a cada 60min para ingestão. (fato)
- F2: A transformação possui 14 etapas de processamento encadeadas, executadas em Spark, com duração total de ~40min. (fato)
- F3: O destino são tabelas no data warehouse, particionadas por hora. (fato)
- F4: Se um lote falha, o próximo acumula o dobro de volume. (fato)
- F5: Sentinel depende do Forge e lê tabelas agregadas. (fato)
- F6: Cerebro depende do Forge e indexa eventos transformados. (fato)
- F7: Os relatórios de billing da Pepper dependem do Forge e rodam de madrugada. (fato)
- F8: A migração precisa consumir do Relay continuamente, processando pequenos blocos em vez do lote de 1h. (fato)
- F9: Os dependentes devem continuar funcionando durante a transição. (fato)
- F10: A migração deve ser incremental, sem big-bang e reversível. (fato)

### Componentes

- C1: "forge-batch-ingest" — ingestão — cron a cada 60min — origem não informada → dados para processamento, lote de 1h. (julgamento)
- C2: Spark — 14 etapas encadeadas — após ingestão (premissa) — dados ingeridos → eventos transformados e dados para tabelas, ~40min. (julgamento)
- C3: Data warehouse — armazenamento — após processamento (premissa) — dados processados → tabelas particionadas por hora. (julgamento)
- C4: Relay — origem de consumo contínuo — gatilho não informado — entrada não informada → pequenos blocos de dados. (julgamento)
- C5: Forge — ingestão, transformação e publicação — cron C1 — origem atual não informada → tabelas e eventos transformados. (julgamento)

Fluxos: C1 → C2 → C3; C4 → processamento incremental → disponibilização aos dependentes, com interfaces intermediárias não especificadas. (julgamento)

### Dependentes

- D1: Sentinel — tabelas agregadas C3 — leitura — frequência não informada — consistência e continuidade assumidas. (premissa)
- D2: Cerebro — eventos transformados C5 — indexação, interface não informada — frequência não informada — integridade e ausência de duplicações indevidas assumidas. (premissa)
- D3: Relatórios de billing da Pepper — resultados C5 — relatórios — madrugada, horário exato não informado — completude no momento de execução assumida. (premissa)

### Pontos frágeis

- P1: C1/C2 — falha de lote → dobro de volume no próximo → atrasos — D1, D2, D3. (julgamento)
- P2: C1/C2 — periodicidade de 60min e processamento de ~40min → defasagem de disponibilização — D1, D2, D3. (julgamento)
- P3: C2 — falha entre 14 etapas encadeadas → interrupção subsequente → indisponibilidade — D1, D2, D3. (julgamento)
- P4: C3/C5 — coexistência inconsistente → divergências em agregação, indexação e billing — D1, D2, D3. (julgamento)

### Invariantes

- I1: Sentinel, Cerebro e relatórios de billing da Pepper permanecem funcionais durante transição, coexistência e rollback — verificar leitura de agregados, indexação íntegra e execução dos relatórios de madrugada; critérios quantitativos não informados. (julgamento)

### Riscos

- R1: Completude — períodos publicados prematuramente. (julgamento)
- R2: Ordenação — alteração da sequência de disponibilização. (julgamento)
- R3: Duplicidade e idempotência — contagem ou indexação duplicada. (julgamento)
- R4: Dados atrasados e reprocessamento — resultados temporais inconsistentes. (julgamento)
- R5: Divergência entre caminhos — versões incompatíveis. (julgamento)
- R6: Picos e contrapressão — acúmulo e atrasos. (julgamento)
- R7: Reconciliação — equivalência entre granularidades não demonstrada. (julgamento)
- R8: Reversibilidade — lacunas e duplicações após rollback. (julgamento)

## 2. Lacunas e premissas

### Lacunas

- L1: Origem, protocolo e limites dos lotes — E1, E2 — bloqueia avanço: sim. (julgamento)
- L2: Dependências e falhas das 14 etapas Spark — E1, E3 — bloqueia avanço: sim. (julgamento)
- L3: Schemas, chaves, agregações e publicação das tabelas — E1, E3, E4 — bloqueia avanço: sim. (julgamento)
- L4: Interface, identidade, ordenação, retenção e replay do Relay — E1, E2 — bloqueia avanço: sim. (julgamento)
- L5: Consultas, frequência e completude de Sentinel — E1, E4 — bloqueia avanço: sim. (julgamento)
- L6: Entrega, indexação e unicidade de Cerebro — E1, E5 — bloqueia avanço: sim. (julgamento)
- L7: Fechamento, origem e tolerância a correções de billing da Pepper — E1, E6 — bloqueia avanço: sim. (julgamento)
- L8: Volumes, capacidade, picos e atrasos admissíveis — E1, E3 — bloqueia avanço: sim. (julgamento)
- L9: Retry, checkpoints, deduplicação e recuperação — E1, E3, E7 — bloqueia avanço: sim. (julgamento)
- L10: Reconciliação, tolerâncias e leitura no rollback — E1, E3–E7 — bloqueia avanço: sim. (julgamento)
- L11: Evolução de schema e compatibilidade — E1, E4–E6 — bloqueia avanço: sim. (julgamento)
- L12: Custos, recursos, monitoramento e limites de carga — E1, E3, E7 — bloqueia avanço: sim. (julgamento)
- L13: Limiares quantitativos, janelas de observação adicionais e critérios formais de aceitação — E1–E7 — bloqueia avanço: sim, após E1. (julgamento)
- L14: Identidade comum, correspondência temporal e cobertura histórica entre C1 e C4 — E1–E3, E7 — bloqueia avanço: sim. (julgamento)

### Premissas

- S1: C1 alimenta C2 antes de C3 — invalidada por caminhos paralelos — E1–E3. (premissa)
- S2: Sentinel exige agregados consistentes — invalidada se aceitar parcialidade — E4. (premissa)
- S3: Cerebro exige integridade e ausência de duplicações — invalidada por deduplicação automática suficiente — E5. (premissa)
- S4: Billing exige completude de madrugada — invalidada por fechamento parcial permitido — E6. (premissa)
- S5: Continuidade inclui contratos preservados — invalidada por mudanças ou interrupções aceitas — E1–E7. (premissa)

## 3. Estratégia de coexistência

- Fases E1–E3: lote permanece fonte de verdade das tabelas agregadas, eventos indexáveis e dados de billing; eventos processados em paralelo não são publicados aos consumidores. (julgamento)
- E4: eventos tornam-se fonte de verdade somente dos agregados de Sentinel; lote permanece referência para Cerebro e Pepper. (julgamento)
- E5: eventos tornam-se fonte de verdade também para Cerebro; Pepper permanece no lote. (julgamento)
- E6: eventos tornam-se fonte de verdade para Pepper após validação do fechamento; lote permanece disponível como alternativa. (julgamento)
- E7: após aceitação de todos os dependentes, desativa-se o lote; o retorno exige reativação e reconciliação da cobertura não processada. (julgamento)
- Reconciliação por partição horária, identidade de evento e período de apuração, distinguindo dados finais de provisórios. (julgamento)

divergencia_percentual = abs(valor_lote - valor_eventos) / abs(valor_lote) * 100, quando valor_lote for diferente de zero; quando for zero, comparar diferenças absolutas. (julgamento)

duplicidades = total_eventos - total_identificadores_unicos; comparar também completude, atrasos, ordenação quando exigida e correções tardias. (julgamento)

- Critério de ordem: D1 → D2 → D3, priorizando validação dos agregados, depois integridade da indexação e por último fechamento financeiro; não há dependência direta informada entre consumidores, portanto a ordem é decisão por risco. (julgamento)

## 4. Etapas

### E1 — Contratos e critérios

- Objetivo: Estabelecer equivalência, garantias e reversibilidade. (julgamento)
- Pré-requisitos: Diagnóstico F1–F10, C1–C5, D1–D3 e I1. (fato)
- O que muda: Definições de contratos, critérios, limites e lacunas L1–L14. (julgamento)
- O que não muda: Sistema, dados publicados e consumidores. (julgamento)
- Coexistência lote/eventos: Somente lote, fonte de verdade integral. (julgamento)
- Dependentes: D1, D2, D3 inalterados. (julgamento)
- Pontos frágeis tratados: P1–P4 caracterizados. (julgamento)
- Riscos tratados: R1–R8 convertidos em critérios verificáveis. (julgamento)
- Invariantes protegidos: I1, sem alteração de consumo. (julgamento)
- Critério de avanço: cobertura de contratos e critérios — limiar a definir (L13) — janela: levantamento completo. (julgamento)
- Ponto de retorno: diagnóstico original — critérios insuficientes — nenhuma alteração irreversível. (julgamento)

### E2 — Consumo paralelo do Relay

- Objetivo: Introduzir ingestão contínua independente. (julgamento)
- Pré-requisitos: E1; L1, L4, L14 resolvidas. (julgamento)
- O que muda: C4 passa a alimentar caminho incremental isolado. (julgamento)
- O que não muda: C1, C2, C3 e publicações. (julgamento)
- Coexistência lote/eventos: Lote oficial; entrada de eventos sem publicação. (julgamento)
- Dependentes: D1, D2, D3 continuam no lote. (julgamento)
- Pontos frágeis tratados: P1, P2, pela independência da entrada. (julgamento)
- Riscos tratados: R2, R3, R4, R6, R8, por validação de identidade, sequência, atraso e recuperação. (julgamento)
- Invariantes protegidos: I1, isolamento integral. (julgamento)
- Critério de avanço: cobertura e recuperação de eventos — limiar a definir (L13) — janela: ao menos um ciclo de 60min. (julgamento)
- Ponto de retorno: interromper caminho incremental isolado — perda de cobertura — nenhuma mudança publicada irreversível. (julgamento)

### E3 — Processamento paralelo e reconciliação

- Objetivo: Demonstrar equivalência entre Spark em lote e processamento incremental. (julgamento)
- Pré-requisitos: E2; L2, L3, L8–L10, L12, L13 resolvidas. (julgamento)
- O que muda: Processamento dos pequenos blocos, com resultados segregados, completude e reconciliação horária. (julgamento)
- O que não muda: Publicações oficiais e contratos. (julgamento)
- Coexistência lote/eventos: Lote oficial; eventos em modo sombra. (julgamento)
- Dependentes: D1, D2, D3 permanecem no lote. (julgamento)
- Pontos frágeis tratados: P1–P4, por isolamento, recuperação e comparação. (julgamento)
- Riscos tratados: R1–R8, incluindo reprocessamento, contrapressão e equivalência. (julgamento)
- Invariantes protegidos: I1, isolamento e resultados oficiais preservados. (julgamento)
- Critério de avanço: divergência, duplicidade, completude, atraso e recuperação — limiar a definir (L13) — janela: ciclos horários, falha simulada e execução de billing de madrugada. (julgamento)
- Ponto de retorno: lote exclusivo — divergência ou falha de recuperação — resultados sombra descartáveis; nenhuma publicação irreversível. (julgamento)

### E4 — Migração de Sentinel

- Objetivo: Migrar os agregados consumidos por D1. (julgamento)
- Pré-requisitos: E3; L5, L11 resolvidas. (julgamento)
- O que muda: D1 passa a consumir agregados incrementais validados. (julgamento)
- O que não muda: Contrato de D1 e consumo de D2, D3. (julgamento)
- Coexistência lote/eventos: Eventos oficiais para D1; lote para D2, D3 e retorno de D1. (julgamento)
- Dependentes: D1 migrado; D2 e D3 inalterados. (julgamento)
- Pontos frágeis tratados: P2, P4, com publicação consistente. (julgamento)
- Riscos tratados: R1–R5, R7, R8, pela reconciliação de agregados. (julgamento)
- Invariantes protegidos: I1, continuidade das leituras e alternativa em lote. (julgamento)
- Critério de avanço: completude, divergência e disponibilidade dos agregados — limiar a definir (L13) — janela: ciclos horários completos. (julgamento)
- Ponto de retorno: D1 volta ao lote — inconsistência — leituras já efetuadas não são desfeitas. (julgamento)

### E5 — Migração de Cerebro

- Objetivo: Migrar a indexação de eventos transformados. (julgamento)
- Pré-requisitos: E4; L6 resolvida. (julgamento)
- O que muda: D2 passa a receber eventos do caminho incremental. (julgamento)
- O que não muda: D1 incremental; D3 em lote. (julgamento)
- Coexistência lote/eventos: Eventos oficiais para D1, D2; lote para D3 e retorno. (julgamento)
- Dependentes: D2 migrado, D1 preservado, D3 inalterado. (julgamento)
- Pontos frágeis tratados: P3, P4, pela recuperação e integridade. (julgamento)
- Riscos tratados: R2–R5, R8, por unicidade, ordenação exigida e replay controlado. (julgamento)
- Invariantes protegidos: I1, continuidade e integridade da indexação. (julgamento)
- Critério de avanço: eventos ausentes, duplicados e divergentes — limiar a definir (L13) — janela: ciclos completos com reprocessamento. (julgamento)
- Ponto de retorno: D2 retorna ao lote — falha de integridade — indexações já materializadas exigem reconciliação. (julgamento)

### E6 — Migração de billing da Pepper

- Objetivo: Migrar a fonte dos relatórios financeiros. (julgamento)
- Pré-requisitos: E5; L7 resolvida. (julgamento)
- O que muda: D3 passa a consumir resultados incrementais fechados. (julgamento)
- O que não muda: Contratos e fontes de D1, D2. (julgamento)
- Coexistência lote/eventos: Eventos oficiais para D1–D3; lote preservado para retorno. (julgamento)
- Dependentes: D3 migrado; D1, D2 preservados. (julgamento)
- Pontos frágeis tratados: P1, P4, pela completude financeira reconciliada. (julgamento)
- Riscos tratados: R1, R3–R5, R7, R8, por fechamento e correções controladas. (julgamento)
- Invariantes protegidos: I1, execução de madrugada com dados completos. (julgamento)
- Critério de avanço: divergência financeira e completude — limiar a definir (L13) — janela: ao menos uma execução completa de madrugada. (julgamento)
- Ponto de retorno: D3 retorna ao lote — divergência de fechamento — relatórios emitidos exigem reconciliação. (julgamento)

### E7 — Desativação do lote

- Objetivo: Encerrar o caminho antigo após aceitação dos três dependentes. (julgamento)
- Pré-requisitos: E6; critérios E3–E6 atendidos; L9, L10, L12 resolvidas. (julgamento)
- O que muda: C1 deixa de acionar produção em lote; caminho incremental assume integralmente. (julgamento)
- O que não muda: Contratos de D1–D3 e resultados aprovados. (julgamento)
- Coexistência lote/eventos: Eventos oficiais; capacidade de retorno ao lote preservada até liberação específica. (julgamento)
- Dependentes: D1, D2 e D3 seguem no incremental. (julgamento)
- Pontos frágeis tratados: P1–P4, eliminando dependência da execução horária antiga. (julgamento)
- Riscos tratados: R4–R8, pela cobertura histórica, reconciliação e recuperação. (julgamento)
- Invariantes protegidos: I1, inclusive durante reativação. (julgamento)
- Critério de avanço: continuidade, completude e recuperabilidade — limiar a definir (L13) — janela: ciclos horários e execução de billing de madrugada após desativação. (julgamento)
- Ponto de retorno: reativar lote e reconciliar períodos afetados — violação de I1 — após perda da capacidade de replay ou reativação, rollback deixa de ser garantido; essa perda não integra a presente etapa. (julgamento)

## 5. Matriz de cobertura

- I1 → E1, E2, E3, E4, E5, E6, E7. (julgamento)
- P1 → E1, E2, E3, E6, E7. (julgamento)
- P2 → E1, E2, E3, E4, E7. (julgamento)
- P3 → E1, E3, E5, E7. (julgamento)
- P4 → E1, E3, E4, E5, E6, E7. (julgamento)
- R1 → E1, E3, E4, E6. (julgamento)
- R2 → E1, E2, E3, E4, E5. (julgamento)
- R3 → E1, E2, E3, E4, E5, E6. (julgamento)
- R4 → E1, E2, E3, E4, E5, E6, E7. (julgamento)
- R5 → E1, E3, E4, E5, E6, E7. (julgamento)
- R6 → E1, E2, E3, E7. (julgamento)
- R7 → E1, E3, E4, E6, E7. (julgamento)
- R8 → E1, E2, E3, E4, E5, E6, E7. (julgamento)
- D1 → E4; D2 → E5; D3 → E6. (julgamento)
```

## Curadoria

**Técnica:** encadeamento de prompts (prompt chaining), sem few-shot. Este elo resolve uma etapa do raciocínio: a partir do diagnóstico, definir a sequência de passos da migração. Os blocos do prompt mapeiam assim: o passo 2 do `<metodo>` herda o contexto, para que o elo 3 não precise do diagnóstico; o passo 3 define a coexistência; o passo 4 ordena as etapas (levantamento, execução em paralelo, migração de um dependente por vez, desativação do lote); os passos 5 a 7 detalham etapas, critérios de avanço e pontos de retorno; `<estrutura_esperada_da_entrada>` declara o que o elo espera ler do elo 1; `<regras>` proíbe virada única e exige que o lote só seja desativado em etapa própria. Exemplos de plano completo ancorariam o modelo em uma ordem específica e prejudicariam o reuso.

**Cadeia:** este é o elo 2 de 3. A entrada é a saída de [`diagnostico-do-forge`](../diagnostico-do-forge/); a saída é a entrada de [`plano-de-migracao-do-forge`](../plano-de-migracao-do-forge/), que exige as seções 1 (contexto herdado) e 4 (etapas). A seção 1 carrega fatos, componentes, dependentes, pontos frágeis, invariantes e riscos com os identificadores e rótulos originais, porque o elo 3 não recebe o diagnóstico.

**Refinos:** o prompt foi criado por meta-prompting com o Opus 5.5 e refinado com revisão humana. O `nome` e as tags passaram ao padrão do repositório e a frase de insuficiência passou para caixa alta. Os pontos frágeis passaram a ser herdados e o limite de palavras subiu de 2.200 para 2.400, para dar margem ao contexto herdado que a seção 1 precisa carregar. A primeira execução mostrou três problemas, corrigidos antes da execução registrada: a primeira etapa, "contratos e critérios", listava como pré-requisito lacunas que ela mesma deveria resolver, e quase todas as lacunas eram bloqueantes; os pontos frágeis herdados não eram tratados em nenhuma etapa nem constavam da matriz; e todos os critérios de avanço saíram sem limiar e sem janela. O prompt passou a definir a primeira etapa como de levantamento (não altera o sistema) e a excluir dos pré-requisitos as lacunas que a própria etapa resolve, a exigir o campo "Pontos frágeis tratados" e a linha `P#` na matriz, a permitir janelas expressas em eventos do cenário (rotuladas como julgamento, sem valores numéricos) e a justificar a ordem de migração dos dependentes. Os limites por seção foram redistribuídos (700, 300, 200, 1.050 e 150 palavras).

**Modelo:** GPT 6 Sol com esforço de raciocínio médio. Na execução final produziu sete etapas, com desativação do lote em etapa própria e precedida de execução em paralelo e reconciliação, manteve identificadores e números herdados, cobriu todos os pontos frágeis e riscos na matriz (conferida linha a linha contra os campos das etapas) e justificou a ordem dos dependentes. Os pontos fracos foram os critérios de avanço sem limiar e o ponto de retorno da etapa final.
