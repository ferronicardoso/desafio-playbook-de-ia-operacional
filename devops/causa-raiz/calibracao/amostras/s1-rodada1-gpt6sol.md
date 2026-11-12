## 1. Resumo

Hipótese principal (inferido): a degradação resulta de contenção de recursos entre indexação e busca, agravada por um `reindex task` anormalmente prolongado, com pressão sobre heap, throttling de indexação, filas de escrita saturadas e perda de eficiência do `query cache` [configuração] [métricas] [logs].

Confiança: média. Há convergência entre métricas e logs quanto à saturação e às consequências para buscas. Entretanto, a taxa de progresso do reindex é incompatível com a duração configurada, e não há evidência suficiente para determinar se o reindex causou a contenção ou se já estava sendo prejudicado por outro gargalo.

## 2. Linha do tempo

Referência temporal: UTC. Métricas amostradas a cada 30 minutos; logs com precisão de milissegundos. Não há evidência de defasagem de relógios, mas a granularidade impede ordenar com precisão eventos entre amostras [métricas] [logs].

- 02:00: horário programado do reindex, com duração média configurada de 90 minutos. Trata-se de configuração, não de comprovação de início da execução (observado) [configuração].
- 08:00: `search_p99_ms=850`, `indexed_docs_per_s=4200`, `heap_used_pct=61`, `cache_hit_pct=74`. Primeira amostra disponível; não constitui baseline histórico (observado) [métricas].
- 08:02:11: `reindex task [88123]` registra 38% de progresso, portanto já estava ativo na abertura da janela (observado) [logs].
- 08:14:33: primeiro evento GC registrado: `young`, duração de 620ms (observado) [logs].
- 08:30: p99 sobe para 1100ms e heap para 68% (observado) [métricas].
- 08:41:07: primeiro throttling registrado em `logs-2026.05`, shard `[7]`, por `segment writing can't keep up` (observado) [logs].
- 09:00–09:12: indexação atinge 9800 docs/s; p99 chega a 2300ms; fila `write` alcança `150/200`; ocorre GC `old` de 1.1s (observado) [métricas] [logs].
- 09:20:02: reindex alcança 40%, após registrar 38% às 08:02:11 (observado) [logs].
- 09:30–09:51: p99 chega a 4100ms, heap a 88%, cache hit cai para 41%; surgem alertas de `parent` circuit breaker e fila `188/200` (observado) [métricas] [logs].
- 09:58–10:05: rejeição de bulk, GC `old` de 1.8s, reindex em 41%, timeout de busca, resultados parciais, circuit breaker acionado e falhas de shards (observado) [logs]. Às 10:00, p99 atinge 6700ms e heap 94% (observado) [métricas].

Divergência quantitativa relevante: o reindex avançou de 3,8M para 4,0M documentos entre 08:02:11 e 09:20:02, aproximadamente 43 docs/s. Entre 09:20:02 e 09:58:44, avançou aproximadamente 43 docs/s novamente (inferido) [logs].

Essas taxas são muito inferiores aos 4100–12400 `indexed_docs_per_s` registrados no dashboard e não sustentam uma conclusão de que o reindex seja o principal gerador dessa carga [métricas] [logs].

Além disso, 41% após o horário esperado de conclusão contrasta com `avg_duration_min: 90` (inferido) [configuração] [logs]. A duração média configurada não demonstra qual deveria ser a taxa efetiva desse trabalho.

## 3. Cadeia causal

1. Origem — hipótese: carga elevada de escrita, possivelmente associada ao reindex prolongado e/ou a outros produtores de indexação. A origem do aumento de carga não foi identificada [configuração] [métricas] [logs].
2. Mecanismo intermediário (observado): `segment writing can't keep up` provoca throttling; filas `write` crescem enquanto heap e atividade de GC aumentam [logs] [métricas].
3. Amplificação (inferido): pressão de memória e redução da eficiência do `query cache` aumentam o custo e a latência potencial das buscas. A relação causal específica entre esses fenômenos não está comprovada [métricas] [logs].
4. Consequências (observado): rejeições de bulk, circuit breaker acionado, buscas lentas, timeout, resultados parciais e falhas de shards [logs].

O mecanismo de degradação está sustentado; a causa-raiz inicial permanece hipótese. A primeira evidência de degradação não demonstra que o reindex seja sua origem.

## 4. Causa versus consequência

| Fenômeno                            | Papel                   | Evidência                                                                                        |
| ----------------------------------- | ----------------------- | ------------------------------------------------------------------------------------------------ |
| Origem da carga de escrita          | indeterminado           | Aumento de `indexed_docs_per_s`, sem identificação do produtor (observado) [métricas]            |
| Reindex prolongado                  | indeterminado           | Progresso anormalmente lento frente à duração média configurada (inferido) [configuração] [logs] |
| `segment writing can't keep up`     | mecanismo intermediário | Throttling registrado (observado) [logs]                                                         |
| Pressão de heap e GC                | mecanismo intermediário | Heap crescente e eventos GC (observado) [métricas] [logs]                                        |
| Queda de `cache_hit_pct`            | mecanismo intermediário | Queda de 74% para 29%; contribuição causal não confirmada (inferido) [métricas]                  |
| Fila `write` saturada               | mecanismo intermediário | `200/200` e rejeição de bulk (observado) [logs]                                                  |
| Circuit breaker                     | consequência            | `parent` acionado por pressão de memória (observado) [logs]                                      |
| Latência, timeout e falhas de busca | consequência            | p99 crescente, resultados parciais e falhas (observado) [métricas] [logs]                        |

## 5. Hipóteses alternativas consideradas

Hipótese A — gargalo primário de escrita de segmentos ou armazenamento.

- A favor (observado): mensagem explícita `segment writing can't keep up`, seguida de throttling e crescimento das filas [logs].
- Contra: ausência de métricas de I/O impede demonstrar saturação física. A pressão de memória também pode explicar parte dos sintomas.
- Ausente: latência de disco, throughput, utilização de armazenamento, merges e tempos de escrita.
- Decisão: mantida em aberto; pode representar o gargalo inicial.

Hipótese B — pressão primária de memória/JVM.

- A favor (observado): aumento de heap, GC `old`, circuit breaker e redução de cache hit [métricas] [logs].
- Contra: throttling por escrita de segmentos aparece antes do primeiro GC `old` registrado, sem evidência de que memória tenha iniciado a degradação [logs].
- Ausente: allocation rate, GC overhead contínuo, memória por componente e estatísticas detalhadas de cache.
- Decisão: plausível como fator amplificador, não demonstrada como origem.

## 6. Ações recomendadas

### Contenção imediata

- Recomendar avaliar pausa controlada do `reindex task [88123]`, caso suporte suspensão segura. Resolve potencial concorrência de escrita; risco de atraso ou interrupção do trabalho; reversível mediante retomada, se suportada. Requer aprovação humana (hipótese) [logs].
- Recomendar identificar e limitar temporariamente o produtor responsável pelo excesso de indexação, após verificar sua origem. Resolve pressão sobre escrita; risco de backlog e atraso na ingestão; reversível. Requer aprovação humana (inferido) [métricas] [logs].

### Correção

- Recomendar investigar o gargalo de escrita do shard afetado antes de alterar capacidade ou parâmetros. Resolve a incerteza sobre contenção física; baixo risco de coleta; reversível (inferido) [logs].
- Recomendar revisar execução, paralelismo e progresso do reindex. Resolve possível concorrência excessiva ou execução ineficiente; risco de atraso no processamento; alterações reversíveis conforme o mecanismo utilizado. Requer aprovação humana para mudanças operacionais (hipótese) [configuração] [logs].

### Prevenção

- Recomendar alertas correlacionados para fila `write`, GC, circuit breaker, latência de busca e progresso do reindex. Resolve detecção tardia; baixo risco; reversível (inferido) [métricas] [logs].
- Recomendar critérios de admissão e controle de concorrência entre reindex e indexação normal, após identificar a capacidade sustentável. Resolve recorrência por competição de recursos; risco de redução de throughput; reversível. Requer aprovação humana (hipótese) [configuração] [métricas] [logs].

Não há base para recomendar novos valores de heap, filas ou cache.

## 7. O que os dados não permitem concluir

- (observado) Existem três artefatos legíveis: configuração estática, métricas de 08:00 a 10:00 e logs de 08:02:11 a 10:05:50 [configuração] [métricas] [logs]. Não há histórico anterior nem dados após essa janela.
- (inferido) A divergência entre progresso do reindex e throughput de indexação permanece sem resolução. Os contadores podem representar escopos distintos, mas isso não foi demonstrado [métricas] [logs].
- (indeterminado) Não se conhece a data de efetiva aplicação da configuração nem se seus valores correspondiam aos parâmetros ativos.
- (indeterminado) A abrangência do problema no cluster não pode ser estabelecida com logs de apenas um pod.

Dados adicionais necessários: início efetivo e histórico completo do reindex; estatísticas por tarefa e índice; origem das escritas; métricas de I/O e merges; estatísticas JVM/GC, circuit breakers e cache por nó; métricas de filas por thread pool; configuração efetiva; logs dos demais nós; e resolução temporal mais fina das métricas.

Sem esses elementos, não é possível diferenciar conclusivamente sobrecarga causada pelo reindex, saturação de escrita por outros produtores ou pressão primária de memória.
