---
nome: Plano de migração do Forge
descricao: Transforma a sequência de etapas de migração de um pipeline para arquitetura orientada a eventos em plano executável, verificável e reversível.
versao: 1.0.0
tags:
  - plano-de-migracao
  - rollback
  - go-no-go
  - arquitetura-orientada-a-eventos
inputs:
  - nome: passos
    descricao: Sequência ordenada de etapas incrementais de migração, com contexto herdado, lacunas, premissas, estratégia de coexistência, critérios de avanço, pontos de retorno e matriz de cobertura.
---

# Plano de migração do Forge

## Objetivo

Transformar a sequência de etapas de migração de um pipeline para arquitetura orientada a eventos em um plano executável, verificável e reversível. É o terceiro e último elo de uma cadeia de três prompts: recebe a saída de [`passos-da-migracao-do-forge`](../passos-da-migracao-do-forge/), que parte do diagnóstico produzido por [`diagnostico-do-forge`](../diagnostico-do-forge/).

A partir das etapas, o prompt consolida as referências (fatos, componentes, dependentes, pontos frágeis, invariantes, lacunas e premissas, cada lacuna e premissa descrita em uma linha), define papéis genéricos e, para cada etapa, produz verificações antes, ações em ordem com papel responsável, verificações depois, critério go/no-go com quem decide e rollback. Cada rollback distingue o que é revertido do que não é e indica a compensação. O plano termina com os pontos de não retorno e a verificação contínua dos invariantes.

Os responsáveis são definidos por papel, nunca por pessoa. O prompt não reordena a estratégia nem cria etapas sem justificativa, não inventa números, limiares nem tempos-alvo (o que falta fica como "a definir" com lacuna registrada) e não inclui comandos com segredos.

## Quando usar

- Depois de definir as etapas de migração, para obter um plano que possa ser revisado e aprovado: o que verificar antes e depois, quem decide cada go/no-go e como voltar atrás.
- Para explicitar, por etapa, o que um rollback não desfaz (efeitos já publicados, índices, relatórios emitidos) e como compensar.
- Como terceira etapa da cadeia de migração, para que a decisão de executar e aprovar se apoie em um único documento.

## Exemplo de uso

1. Executar os elos 1 e 2 ([`diagnostico-do-forge`](../diagnostico-do-forge/) e [`passos-da-migracao-do-forge`](../passos-da-migracao-do-forge/)) e guardar a saída completa do elo 2. Se houver barras de cópia (`\~`, `\-`, `\*`), removê-las.
2. Copiar o conteúdo de `prompt.md` (sem o frontmatter) e substituir `{{passos}}` pela saída do elo 2. É o único parâmetro e é obrigatório. A entrada deve conter as seções 1 (contexto herdado) e 4 (etapas).
3. Executar em uma conversa ou chamada nova.
4. Ler primeiro a seção 1 (lacunas, premissas e bloqueios) e a seção 4 (pontos de não retorno). Conferir, por etapa, as verificações, o critério go/no-go e o rollback.
5. Antes de executar o plano, resolver as lacunas marcadas como bloqueantes e definir os limiares marcados como "a definir".

Um exemplo de execução com saída real está na seção [Execução](#execução).

## Limitações conhecidas

- A qualidade do plano depende das etapas recebidas. Se a entrada não for reconhecível como sequência de etapas com critério de avanço e ponto de retorno, o prompt responde com a frase de insuficiência.
- Nenhum limiar numérico aparece no plano: todos vêm como "limiares a definir (L13)" ou como "limites aprovados" na primeira etapa. O plano diz como verificar, mas não fixa o valor que separa go de no-go.
- No rollback, o formato pede um papel por passo; a saída traz um conjunto de papéis ao final da lista de passos, sem atribuir cada passo. A responsabilidade por passo fica menos precisa do que nas ações.
- O plano concentra todas as decisões de go/no-go e de rollback em um único papel, o aprovador da mudança, com aceite técnico dos responsáveis por dependente. Em uma organização real, a alçada pode ser distinta.
- As lacunas e premissas foram descritas na seção 1, mas algumas descrições afirmam ausência no sistema ("não documentados", "não mapeadas") quando o diagnóstico só diz que a informação não foi informada.
- A seção 1 ficou com 789 palavras, contra o limite de 750, e acrescentou subtítulos e a estratégia de coexistência, que o formato não prevê. O limite total (3.650 palavras) foi respeitado, com cerca de 3.257 palavras, contadas por tokens com letra ou número.
- Os pontos frágeis aparecem nas referências, mas não são ligados a verificações ou gatilhos de rollback, embora a etapa de processamento em paralelo simule falha e recuperação.
- O plano termina na desativação reversível do lote. A retirada definitiva do caminho em lote depende de uma "decisão posterior específica" que o plano não define.
- O resultado vem de duas execuções com um único modelo. A primeira usou uma versão anterior do prompt; a saída registrada abaixo é a da versão final.

## Execução

| Item | Valor |
|---|---|
| Modelo | GPT 6 Sol |
| Identificador na API | `gpt-6-sol` |
| Esforço de raciocínio | médio |

O único parâmetro, `passos`, foi preenchido com a saída do elo 2 (seção Execução de [`passos-da-migracao-do-forge`](../passos-da-migracao-do-forge/)), sem alteração além da remoção das barras de cópia. A execução foi feita em um chat novo e temporário, sem acesso à memória.

### Saída

A saída foi copiada sem edição, exceto a remoção das barras de cópia do ChatGPT (`\~`, `\-`).

```markdown
# PLANO DE MIGRAÇÃO EXECUTÁVEL

## 1. Referências, premissas e bloqueios

### Referências

Fatos essenciais

- F1: O Forge executa o cron `forge-batch-ingest` a cada 60min para ingestão. (fato)
- F2: A transformação possui 14 etapas encadeadas em Spark, com duração total de ~40min. (fato)
- F3: O destino são tabelas no data warehouse, particionadas por hora. (fato)
- F4: A falha de um lote dobra o volume acumulado no próximo. (fato)
- F5: Sentinel depende do Forge e lê tabelas agregadas. (fato)
- F6: Cerebro depende do Forge e indexa eventos transformados. (fato)
- F7: Os relatórios de billing da Pepper dependem do Forge e executam de madrugada. (fato)
- F8: A migração deve consumir continuamente do Relay, processando pequenos blocos. (fato)
- F9: Os dependentes devem permanecer funcionais durante a transição. (fato)
- F10: A migração deve ser incremental, sem big-bang e reversível. (fato)

Componentes e fluxos

- C1: `forge-batch-ingest` — ingestão por cron a cada 60min; origem não informada. (julgamento)
- C2: Spark — 14 etapas encadeadas, duração ~40min, processamento após ingestão. (julgamento)
- C3: Data warehouse — tabelas particionadas por hora, publicadas após processamento. (julgamento)
- C4: Relay — origem do consumo contínuo, com protocolo e identidade não informados. (julgamento)
- C5: Forge — ingestão, transformação e publicação de tabelas e eventos transformados. (julgamento)
- Fluxos: C1 → C2 → C3; C4 → processamento incremental → dependentes, com interfaces intermediárias a definir. (julgamento)

Dependentes

- D1: Sentinel — leitura de agregados; frequência e tolerância à parcialidade não informadas. (premissa)
- D2: Cerebro — indexação de eventos transformados; interface, unicidade e frequência não informadas. (premissa)
- D3: Relatórios de billing da Pepper — execução de madrugada; fechamento e tolerância a correções não informados. (premissa)

Pontos frágeis e invariante

- P1: Falha em C1/C2 dobra o volume seguinte e pode atrasar D1–D3. (julgamento)
- P2: Periodicidade de 60min e processamento de ~40min causam defasagem. (julgamento)
- P3: Falhas nas 14 etapas encadeadas podem interromper processamento subsequente. (julgamento)
- P4: Coexistência inconsistente pode causar divergências de agregação, indexação e billing. (julgamento)
- I1: Sentinel, Cerebro e billing da Pepper devem permanecer funcionais durante transição, coexistência e rollback; critérios quantitativos não informados. (julgamento)

Riscos controlados

- R1: Completude e publicação prematura. (julgamento)
- R2: Ordenação de disponibilização. (julgamento)
- R3: Duplicidade e idempotência. (julgamento)
- R4: Atrasos e reprocessamento. (julgamento)
- R5: Divergência entre caminhos. (julgamento)
- R6: Picos e contrapressão. (julgamento)
- R7: Equivalência e reconciliação. (julgamento)
- R8: Lacunas e duplicações no rollback. (julgamento)

Lacunas

- L1: Origem, protocolo e limites dos lotes não documentados. (julgamento)
- L2: Dependências e falhas das 14 etapas Spark não mapeadas. (julgamento)
- L3: Schemas, chaves, agregações e publicação não especificados. (julgamento)
- L4: Interface, identidade, ordenação, retenção e replay do Relay desconhecidos. (julgamento)
- L5: Consultas, frequência e completude de Sentinel indefinidas. (julgamento)
- L6: Entrega, indexação e unicidade de Cerebro indefinidas. (julgamento)
- L7: Fechamento, origem e correções de billing indefinidos. (julgamento)
- L8: Volumes, capacidade, picos e atrasos admissíveis desconhecidos. (julgamento)
- L9: Retry, checkpoints, deduplicação e recuperação indefinidos. (julgamento)
- L10: Reconciliação, tolerâncias e leitura no rollback indefinidas. (julgamento)
- L11: Evolução de schema e compatibilidade não especificadas. (julgamento)
- L12: Custos, recursos, monitoramento e limites de carga não definidos. (julgamento)
- L13: Limiares quantitativos, janelas adicionais e critérios formais de aceitação a definir. (julgamento)
- L14: Identidade comum, correspondência temporal e cobertura histórica C1/C4 desconhecidas. (julgamento)

Premissas

- S1: C1 alimenta C2 antes de C3, sem caminhos paralelos desconhecidos. (premissa)
- S2: Sentinel exige agregados consistentes, sem parcialidade não autorizada. (premissa)
- S3: Cerebro exige integridade e ausência de duplicações indevidas. (premissa)
- S4: Billing exige completude na execução de madrugada. (premissa)
- S5: Continuidade significa preservar contratos dos consumidores. (premissa)

Estratégia de coexistência

- E1–E3: Lote permanece fonte de verdade integral; processamento incremental sem publicação oficial. (julgamento)
- E4: Incremental oficial para Sentinel; lote para Cerebro, Pepper e retorno de Sentinel. (julgamento)
- E5: Incremental oficial para Sentinel e Cerebro; lote para Pepper e retorno. (julgamento)
- E6: Incremental oficial para todos; lote disponível para rollback. (julgamento)
- E7: Lote desativado, mantendo reativação e replay recuperáveis. (julgamento)
- Reconciliação: comparar partições horárias, identidades, períodos de apuração e estados provisórios/finais. (julgamento)
- Fórmula: `divergencia_percentual = abs(valor_lote - valor_eventos) / abs(valor_lote) * 100`, quando valor_lote diferente de zero; caso contrário, comparar diferença absoluta. (julgamento)
- Fórmula: `duplicidades = total_eventos - total_identificadores_unicos`; avaliar também eventos ausentes, atrasos, ordenação e correções tardias. (julgamento)

### Bloqueios e validações prévias

- E1: Resolvidas por esta etapa: L1–L14; validar S1–S5 e formalizar critérios antes de E2. (julgamento)
- E2: Bloqueantes L1, L4, L14 e critérios aplicáveis de L13; validar S1, S5. (julgamento)
- E3: Bloqueantes L2, L3, L8–L10, L12–L14; validar S1, S5. (julgamento)
- E4: Bloqueantes L5, L10, L11, L13; validar S2, S5. (julgamento)
- E5: Bloqueantes L6, L10, L11, L13; validar S3, S5. (julgamento)
- E6: Bloqueantes L7, L10, L11, L13; validar S4, S5. (julgamento)
- E7: Bloqueantes L9, L10, L12–L14; validar S1, S5 e recuperabilidade de D1–D3. (julgamento)

### Problemas de ordenação detectados

- Nenhum. (julgamento)

## 2. Papéis

- Responsável pela plataforma de dados — coordena E1–E3 e E7, processamento, publicação, reconciliação e recuperação; propõe go/no-go e executa rollback técnico. (julgamento)
- Responsável por Sentinel — valida agregados, aprova tecnicamente E4 e executa retorno de D1. (julgamento)
- Responsável por Cerebro — valida integridade da indexação, aprova tecnicamente E5 e executa retorno de D2. (julgamento)
- Responsável por billing — valida fechamento financeiro, aprova tecnicamente E6 e reconcilia relatórios. (julgamento)
- Responsável pela operação — verifica monitoramento, capacidade, alertas e continuidade; pode solicitar interrupção e rollback emergencial. (julgamento)
- Dono do dado — aprova contratos, semântica, equivalência, tolerâncias e correções de dados. (julgamento)
- Aprovador da mudança — decide go/no-go definitivo em E1–E7, autoriza rollback e exige aprovação técnica dos responsáveis afetados. (julgamento)

## 3. Plano por etapa

### E1 — Contratos e critérios

- Responsável pela etapa: Responsável pela plataforma de dados. (julgamento)

Verificações antes

- Confirmar F1–F10, C1–C5, D1–D3, P1–P4, I1 e operação atual; inspecionar execução, fluxos, tabelas e consumidores — plataforma de dados e operação. (julgamento)
- Confirmar que o lote permanece fonte de verdade, sem mudança de publicação — operação. (julgamento)

Ações em ordem

1. Mapear origem, interfaces, dependências, schemas, chaves e cobertura C1–C4, resolvendo L1–L4 e L14 — plataforma de dados. (julgamento)
2. Documentar contratos de Sentinel, Cerebro e Pepper, resolvendo L5–L7 e L11 — responsáveis pelos dependentes e dono do dado. (julgamento)
3. Medir capacidade, atraso, picos, recuperação, custos e monitoramento, resolvendo L8, L9 e L12 — plataforma de dados e operação. (julgamento)
4. Definir reconciliação, tolerâncias, checkpoints e leitura no rollback, resolvendo L10 — dono do dado e plataforma de dados. (julgamento)
5. Formalizar limiares, janelas, evidências e critérios de aprovação L13; validar S1–S5 — aprovador da mudança e responsáveis técnicos. (julgamento)

Verificações depois

- Cobertura de contratos, lacunas e critérios — limiar a definir (L13) — janela: levantamento completo — dono do dado. (julgamento)
- Existência de procedimentos testáveis de reconciliação e retorno — aceite formal a definir (L13) — levantamento completo — operação. (julgamento)

Go/no-go: Go com critérios aprovados, contratos mapeados e lacunas bloqueantes resolvidas; no-go diante de qualquer requisito bloqueante pendente; decide: aprovador da mudança. (julgamento)

Rollback

- Gatilhos: diagnóstico contraditório, critérios insuficientes ou ausência de aprovação. (julgamento)
- Procedimento em ordem: 1. Suspender aprovação; 2. Invalidar definições não aceitas; 3. Retomar levantamento — aprovador da mudança e plataforma de dados. (julgamento)
- É revertido: aprovação operacional das definições contestadas. (julgamento)
- Não é revertido e compensação: registros e diagnósticos produzidos; corrigir documentação e revalidar. (julgamento)
- Estado após o rollback: lote exclusivo, consumidores inalterados, sem alteração irreversível. (julgamento)

### E2 — Consumo paralelo do Relay

- Responsável pela etapa: Responsável pela plataforma de dados. (julgamento)

Verificações antes

- E1 aprovada; L1, L4, L14 resolvidas; critérios L13 definidos; S1 e S5 verificadas — plataforma de dados e aprovador. (julgamento)
- Confirmar isolamento do caminho incremental e inexistência de publicação aos dependentes — operação. (julgamento)

Ações em ordem

1. Configurar consumo contínuo C4 para destino isolado, sem alterar C1–C3 — plataforma de dados. (julgamento)
2. Registrar identidades, posições de leitura, sequência e cobertura temporal — plataforma de dados. (julgamento)
3. Exercitar interrupção, retomada e replay conforme contrato do Relay — plataforma de dados e operação. (julgamento)
4. Comparar cobertura do consumo com a origem e verificar impacto operacional — plataforma de dados. (julgamento)

Verificações depois

- Cobertura, atraso, ordenação exigida e duplicidades — limiares a definir (L13) — ao menos um ciclo de 60min — plataforma de dados. (julgamento)
- Recuperação após interrupção e ausência de publicação oficial — critérios aprovados em E1 — ao menos um ciclo de 60min — operação. (julgamento)

Go/no-go: Go com consumo recuperável, cobertura aceita e isolamento comprovado; no-go com perda de eventos, recuperação insuficiente ou impacto nos consumidores; decide: aprovador da mudança. (julgamento)

Rollback

- Gatilhos: perda de cobertura, falha de replay, contrapressão ou quebra de isolamento. (julgamento)
- Procedimento em ordem: 1. Interromper consumo incremental; 2. Preservar posições e evidências; 3. Confirmar lote oficial; 4. Reconciliar lacunas antes de retomar testes — plataforma de dados e operação. (julgamento)
- É revertido: consumo paralelo ativo. (julgamento)
- Não é revertido e compensação: registros já capturados e posições consumidas; preservar ou reconstruir cobertura com replay validado. (julgamento)
- Estado após o rollback: lote exclusivo; nenhum dependente migrado. (julgamento)

### E3 — Processamento paralelo e reconciliação

- Responsável pela etapa: Responsável pela plataforma de dados. (julgamento)

Verificações antes

- E2 aprovada, L2, L3, L8–L10, L12–L14 resolvidas; S1, S5 validadas — plataforma de dados. (julgamento)
- Confirmar capacidade e segregação dos resultados sombra — operação. (julgamento)

Ações em ordem

1. Implementar transformação incremental equivalente às 14 etapas Spark, preservando semântica validada — plataforma de dados. (julgamento)
2. Processar pequenos blocos do Relay em saída segregada, sem publicação oficial — plataforma de dados. (julgamento)
3. Aplicar checkpoints, deduplicação, tratamento de atrasados e reprocessamento definidos — plataforma de dados. (julgamento)
4. Reconciliar partições horárias, identidades, agregações e períodos de billing, separando estados finais e provisórios — dono do dado. (julgamento)
5. Simular falha e recuperação, verificar contrapressão e observar execução de billing de madrugada — plataforma de dados, operação e responsável por billing. (julgamento)

Verificações depois

- Divergência percentual ou absoluta, duplicidades, completude e atraso — limiares a definir (L13) — ciclos horários — dono do dado. (julgamento)
- Recuperação, capacidade e consistência financeira — critérios E1 — falha simulada e execução de madrugada — operação e responsável por billing. (julgamento)

Go/no-go: Go com equivalência demonstrada, recuperação aceita e I1 preservado; no-go diante de divergência fora dos critérios ou falha operacional; decide: aprovador da mudança. (julgamento)

Rollback

- Gatilhos: divergência, inconsistência temporal, duplicação indevida ou recuperação falha. (julgamento)
- Procedimento em ordem: 1. Suspender processamento sombra; 2. Isolar resultados; 3. Preservar evidências e checkpoints; 4. Validar lote exclusivo — plataforma de dados e operação. (julgamento)
- É revertido: processamento incremental em modo sombra e resultados descartáveis. (julgamento)
- Não é revertido e compensação: registros de teste e consumo já realizado; refazer reconciliação e replay controlado antes de nova tentativa. (julgamento)
- Estado após o rollback: lote como única fonte oficial, D1–D3 inalterados. (julgamento)

### E4 — Migração de Sentinel

- Responsável pela etapa: Responsável por Sentinel. (julgamento)

Verificações antes

- E3 aceita; L5, L10, L11, L13 resolvidas; S2, S5 validadas — responsável por Sentinel e dono do dado. (julgamento)
- Confirmar disponibilidade de agregados finais e retorno ao lote — operação. (julgamento)

Ações em ordem

1. Reconciliar agregados incrementais por partição horária e contrato de D1 — dono do dado. (julgamento)
2. Validar consultas e consistência de Sentinel contra resultados oficiais — responsável por Sentinel. (julgamento)
3. Alterar exclusivamente a fonte de D1 para agregados incrementais aprovados — responsável por Sentinel e plataforma de dados. (julgamento)
4. Monitorar consultas, disponibilidade, completude e divergência; manter D2 e D3 no lote — operação. (julgamento)

Verificações depois

- Completude, divergência e disponibilidade de agregados — limiares a definir (L13) — ciclos horários completos — responsável por Sentinel. (julgamento)
- Continuidade D2–D3 e viabilidade de retorno de D1 — critérios E1 — ciclos horários completos — operação. (julgamento)

Go/no-go: Go com agregados aceitos e leituras preservadas; no-go com inconsistência, indisponibilidade ou contrato violado; decide: aprovador da mudança, mediante aceite de Sentinel. (julgamento)

Rollback

- Gatilhos: agregados inconsistentes, incompletos ou indisponíveis. (julgamento)
- Procedimento em ordem: 1. Suspender publicação incremental para D1; 2. Restabelecer leitura do lote; 3. Reconciliar períodos afetados; 4. Verificar consultas — responsável por Sentinel e plataforma de dados. (julgamento)
- É revertido: fonte oficial de agregados de D1. (julgamento)
- Não é revertido e compensação: leituras e decisões já realizadas; identificar resultados afetados e corrigir derivados quando aplicável. (julgamento)
- Estado após o rollback: D1–D3 no lote; incremental segregado para diagnóstico. (julgamento)

### E5 — Migração de Cerebro

- Responsável pela etapa: Responsável por Cerebro. (julgamento)

Verificações antes

- E4 aceita; L6, L10, L11, L13 resolvidas; S3, S5 validadas — responsável por Cerebro. (julgamento)
- Confirmar deduplicação, ordenação exigida, replay e retorno de indexação — plataforma de dados e operação. (julgamento)

Ações em ordem

1. Comparar identidades e integridade dos eventos transformados entre caminhos — responsável por Cerebro. (julgamento)
2. Validar mecanismo de indexação e reprocessamento sem duplicações indevidas — responsável por Cerebro. (julgamento)
3. Transferir entrega oficial de eventos de D2 para o incremental; preservar D1 incremental e D3 no lote — plataforma de dados. (julgamento)
4. Monitorar eventos ausentes, duplicados, divergentes e recuperação — responsável por Cerebro e operação. (julgamento)

Verificações depois

- Eventos ausentes, duplicados e divergentes — limiares a definir (L13) — ciclos completos com reprocessamento — responsável por Cerebro. (julgamento)
- Integridade de D1 e continuidade de D3 — contratos aprovados — mesmos ciclos — operação. (julgamento)

Go/no-go: Go com indexação íntegra, replay aceito e continuidade; no-go com falha de unicidade, perda ou divergência; decide: aprovador da mudança, mediante aceite de Cerebro. (julgamento)

Rollback

- Gatilhos: perda, duplicação indevida, divergência ou falha de indexação. (julgamento)
- Procedimento em ordem: 1. Suspender entrega incremental a D2; 2. Retomar fonte em lote; 3. Comparar identidades indexadas; 4. Corrigir ausências e duplicações conforme contrato; 5. Validar buscas e indexação — responsável por Cerebro. (julgamento)
- É revertido: origem oficial de novos eventos de D2. (julgamento)
- Não é revertido e compensação: índices e efeitos já materializados; reconciliar registros e corrigir indexações afetadas. (julgamento)
- Estado após o rollback: D1 incremental, D2–D3 no lote. (julgamento)

### E6 — Migração de billing da Pepper

- Responsável pela etapa: Responsável por billing. (julgamento)

Verificações antes

- E5 aceita; L7, L10, L11, L13 resolvidas; S4, S5 validadas — responsável por billing e dono do dado. (julgamento)
- Confirmar regras de fechamento, correções tardias e retorno à fonte em lote — responsável por billing. (julgamento)

Ações em ordem

1. Reconciliar resultados financeiros por período de apuração e estado de fechamento — dono do dado. (julgamento)
2. Confirmar completude dos períodos exigidos para relatórios de madrugada — responsável por billing. (julgamento)
3. Transferir fonte oficial de D3 para resultados incrementais fechados; preservar lote para retorno — plataforma de dados e responsável por billing. (julgamento)
4. Executar relatórios e confrontar resultados financeiros com referências aceitas — responsável por billing. (julgamento)

Verificações depois

- Divergência financeira e completude — limiares a definir (L13) — ao menos uma execução completa de madrugada — responsável por billing. (julgamento)
- Continuidade de D1–D2 e recuperabilidade do lote — critérios aprovados — mesma execução — operação. (julgamento)

Go/no-go: Go com fechamento reconciliado, relatórios aceitos e I1 preservado; no-go com divergência, incompletude ou correção não controlada; decide: aprovador da mudança, mediante aceite de billing. (julgamento)

Rollback

- Gatilhos: divergência financeira, fechamento incompleto ou relatório incorreto. (julgamento)
- Procedimento em ordem: 1. Suspender uso do resultado incremental por D3; 2. Restaurar fonte em lote; 3. Reconciliar períodos; 4. Corrigir ou reemitir relatórios afetados conforme contrato; 5. Validar fechamento — responsável por billing. (julgamento)
- É revertido: fonte oficial dos relatórios financeiros. (julgamento)
- Não é revertido e compensação: relatórios emitidos e seus efeitos externos; registrar divergências e executar correções aprovadas. (julgamento)
- Estado após o rollback: D1–D2 incrementais; D3 em lote. (julgamento)

### E7 — Desativação do lote

- Responsável pela etapa: Responsável pela plataforma de dados. (julgamento)

Verificações antes

- E6 aceita; critérios E3–E6 cumpridos; L9, L10, L12–L14 resolvidas; S1, S5 validadas — plataforma de dados. (julgamento)
- Confirmar capacidade de reativação, replay, cobertura histórica e retorno dos três dependentes — operação e respectivos responsáveis. (julgamento)

Ações em ordem

1. Registrar posições, cobertura e resultados reconciliados antes da desativação — plataforma de dados. (julgamento)
2. Confirmar autorização formal e prontidão do rollback — aprovador da mudança e operação. (julgamento)
3. Desativar acionamento produtivo de `forge-batch-ingest`, preservando configuração e capacidade de reativação — plataforma de dados. (julgamento)
4. Monitorar processamento incremental, D1–D3 e cobertura temporal — operação. (julgamento)
5. Validar ciclos horários e execução de billing de madrugada após desativação — responsáveis técnicos. (julgamento)

Verificações depois

- Continuidade, completude, atraso e recuperabilidade — limiares a definir (L13) — ciclos horários e execução de billing de madrugada após desativação — operação. (julgamento)
- Cobertura histórica e ausência de lacunas indevidas — critérios L10 e L14 — mesmos eventos de observação — dono do dado. (julgamento)

Go/no-go: Go com I1 preservado, consumidores aceitos e retorno comprovadamente recuperável; no-go com lacunas, falha de reativação ou violação de contratos; decide: aprovador da mudança. (julgamento)

Rollback

- Gatilhos: violação de I1, perda de completude ou impossibilidade de recuperação incremental. (julgamento)
- Procedimento em ordem: 1. Registrar último período incremental consistente; 2. Reativar lote; 3. Reconciliar cobertura e identidades entre caminhos; 4. Restaurar fontes oficiais dos dependentes conforme contratos; 5. Corrigir efeitos materializados e verificar I1 — plataforma de dados, operação e responsáveis pelos dependentes. (julgamento)
- É revertido: desativação do lote e fontes oficiais dos consumidores, conforme necessidade de retorno. (julgamento)
- Não é revertido e compensação: eventos publicados, índices, leituras e relatórios emitidos; reconciliar períodos e corrigir dados ou efeitos externos afetados. (julgamento)
- Estado após o rollback: lote reativado e oficial para consumidores restaurados; incremental isolado até nova aprovação. (julgamento)

## 4. Pontos de não retorno

- E1–E3: Não há publicação oficial incremental; antes de descartar resultados sombra ou checkpoints, comprovar cobertura recuperável — aprovador: aprovador da mudança. (julgamento)
- E4: Leituras já realizadas por Sentinel não podem ser desfeitas; antes da troca, comprovar consistência, rastreabilidade e retorno ao lote — aprovador: aprovador da mudança. (julgamento)
- E5: Indexações materializadas podem persistir após retorno; antes da troca, comprovar reconciliação e correção de índices — aprovador: aprovador da mudança. (julgamento)
- E6: Relatórios financeiros emitidos podem produzir efeitos externos; antes da troca, comprovar fechamento, rastreabilidade e mecanismo de correção — aprovador: aprovador da mudança. (julgamento)
- E7: Perda da capacidade de replay ou reativação inviabiliza rollback garantido; essa perda não está autorizada nesta etapa e exige decisão posterior específica — aprovador: aprovador da mudança. (julgamento)

## 5. Verificação contínua dos invariantes

- I1 — Sentinel: executar consultas de agregados, verificar completude, consistência e disponibilidade conforme contrato; antes e depois de cada mudança de fonte, nos ciclos horários de E2–E7 e durante rollback — responsável por Sentinel e operação. (julgamento)
- I1 — Cerebro: comparar identidades, eventos ausentes, duplicações indevidas e integridade da indexação; durante coexistência, reprocessamento, E2–E7 e rollback — responsável por Cerebro e operação. (julgamento)
- I1 — Billing da Pepper: verificar fechamento, completude, divergência financeira e execução dos relatórios de madrugada; em E3, E6, E7 e rollback que alcance períodos financeiros — responsável por billing e dono do dado. (julgamento)
- I1 — Publicação e coexistência: conferir por consumidor qual caminho é oficial, distinguindo dados finais e provisórios; antes e depois de cada alteração E1–E7 — plataforma de dados. (julgamento)
- I1 — Recuperação: verificar checkpoints, cobertura temporal, replay e capacidade de retorno sem lacunas ou duplicações indevidas; em testes de falha, antes de cada go e após rollback E2–E7 — plataforma de dados e operação. (julgamento)
- I1 — Decisão: suspender avanço diante de violação dos critérios aprovados; registrar evidências, decisão go/no-go, autorização de rollback e aceite dos responsáveis afetados em cada etapa E1–E7 — aprovador da mudança. (julgamento)
```

## Curadoria

**Técnica:** encadeamento de prompts (prompt chaining), sem few-shot. Este elo resolve a última etapa do raciocínio: transformar etapas em um plano que alguém possa executar, verificar e desfazer. Os blocos do prompt mapeiam assim: o passo 2 do `<metodo>` consolida referências e bloqueios, para que o plano seja entendido sem a entrada; o passo 3 define os papéis; o passo 4 detalha cada etapa (ações, verificações, go/no-go e rollback); o passo 5 lista os pontos de não retorno; o passo 6 define a verificação contínua dos invariantes; o passo 7 impede reordenar a estratégia sem registro; `<regras>` proíbe inventar valores, exige que todo rollback distinga o que é revertido do que não é e fixa responsáveis por papel. Exemplos de plano completo ancorariam o modelo em uma estrutura específica e prejudicariam o reuso.

**Cadeia:** este é o elo 3 de 3. A entrada é a saída de [`passos-da-migracao-do-forge`](../passos-da-migracao-do-forge/), que parte do diagnóstico de [`diagnostico-do-forge`](../diagnostico-do-forge/). Como cada elo só enxerga a saída do anterior, o plano reproduz, em uma linha, os fatos, as lacunas e as premissas de que depende.

**Refinos:** o prompt foi criado por meta-prompting com o Opus 5.5 e refinado com revisão humana. O `nome` e as tags passaram ao padrão do repositório e a frase de insuficiência passou para caixa alta. O prompt passou a receber e a listar os pontos frágeis e os fatos com números e nomes literais, a registrar problemas de ordenação em uma subseção própria e a ter limites maiores (de 2.800 para 3.350 palavras, com a seção 3 em 2.300). A primeira execução mostrou três problemas, corrigidos antes da execução registrada: o plano citava lacunas (`L1`, `L14`) e premissas (`S1`, `S5`) por código sem descrevê-las, o que impedia o entendimento sem outro documento; a primeira etapa listava como bloqueio prévio as lacunas que ela mesma resolve; e a verificação da etapa de captura trazia um valor ("tolerância zero") que não constava da entrada. O prompt passou a exigir a descrição de cada L# e S# na seção 1, a registrar a primeira etapa como "resolvidas por esta etapa" e a proibir limiares que não venham da entrada ou de eventos do cenário. O limite total subiu para 3.650 palavras, com a seção 1 em 750.

**Modelo:** GPT 6 Sol com esforço de raciocínio médio. Na execução final produziu as sete etapas na ordem recebida, descreveu todas as lacunas e premissas, deu a cada etapa verificações antes e depois, go/no-go e rollback com a separação entre o que é revertido e o que não é, usou papéis genéricos e declarou, na etapa final, que a perda da capacidade de replay ou reativação impede o rollback garantido e não está autorizada. Os pontos fracos foram a ausência de limiares e a atribuição de papéis por passo no rollback.
