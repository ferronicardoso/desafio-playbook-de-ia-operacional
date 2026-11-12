---
nome: Causa-raiz de degradação
descricao: "Analisa um pacote de artefatos de uma degradação em sistemas de busca e indexação sobre Elasticsearch/JVM e produz um diagnóstico de causa-raiz rastreável, com nível de confiança e ações proporcionais."
versao: 1.0.0
tags:
  - sre
  - causa-raiz
  - diagnostico
  - incidentes
inputs:
  - nome: pacote_artefatos
    descricao: "Texto único com os artefatos da degradação colados em sequência, cada um precedido por um rótulo (por exemplo: configuração, métricas, logs). Inclua a configuração do cluster, as métricas em série temporal com timestamps e unidades, e o trecho de logs que cubra o período anterior e posterior ao início da degradação. Remova segredos, credenciais e dados pessoais antes de colar. Artefatos ausentes podem ser omitidos."
---

# Causa-raiz de degradação

## Objetivo

Analisar o pacote de artefatos de uma degradação (configuração do cluster, métricas em série temporal e trecho de logs) e produzir um diagnóstico de causa-raiz em sete seções fixas: resumo, linha do tempo, cadeia causal, causa versus consequência, hipóteses alternativas, ações recomendadas e o que os dados não permitem concluir. O prompt busca a causa-raiz, e não apenas a lista de sintomas, e informa o nível de confiança da conclusão.

O prompt usa somente o que está nos artefatos. Cada valor citado indica a fonte, e cada afirmação relevante é marcada como observada, inferida ou hipótese. Quando os dados não sustentam uma causa-raiz, a conclusão é apresentada como hipótese principal. O prompt não acessa o ambiente, não consulta métricas e não executa ações.

## Quando usar

- Degradação de busca ou indexação (latência alta, resultados parciais, rejeições de escrita) em sistemas sobre Elasticsearch e JVM, quando já estão à mão a configuração, as métricas e os logs do período.
- Escalonamento do plantão, para que quem recebe o caso tenha uma cadeia causal e uma lista do que falta coletar.
- Revisão pós-incidente, para consolidar a ordem dos eventos e separar causa de consequência.

## Exemplo de uso

1. Reunir os três artefatos do período da degradação: a configuração do cluster, as métricas e o trecho de logs.
2. Sanear o pacote antes do envio a um modelo externo (ver Limitações conhecidas).
3. Colar os artefatos em sequência em um único texto, cada um precedido por um rótulo curto e estável, por exemplo `CONFIGURAÇÃO`, `MÉTRICAS` e `LOGS`. Os rótulos aparecem nas citações de fonte da resposta.
4. Copiar o conteúdo de `prompt.md` (sem o frontmatter) e substituir `{{pacote_artefatos}}` pelo texto montado. É o único parâmetro e é obrigatório.
5. Executar em uma conversa ou chamada nova, ler primeiro a seção 7 e conferir os itens marcados como inferido ou hipótese antes de agir sobre as ações recomendadas.

Exemplos de execução com saída real estão na seção [Execução](#execução).

## Limitações conhecidas

- Os artefatos do exemplo são fictícios. Para dados reais, o pacote deve ser saneado antes do envio: substituir nomes de pods e nós, nomes de índices que revelem cliente ou tenant, nome do repositório de infra, IPs, hostnames e namespaces por pseudônimos consistentes, para preservar a correlação entre as fontes; remover segredos, tokens, e-mails e usuários presentes em configuração ou mensagens de log; manter timestamps, valores de métricas e classes e mensagens de erro, que a análise exige. O prompt omite segredos que detectar, mas isso não substitui o saneamento prévio.
- A saída varia entre execuções. Nas duas rodadas abaixo, o reindex foi classificado como "indeterminado" na primeira e como hipótese principal, em concorrência com outros fatores, na segunda. Em ambas a confiança foi média.
- As duas rodadas derivam a taxa implícita do reindex (cerca de 43 docs/s), mas nenhuma conclui explicitamente que a tarefa está travada em relação ao ritmo esperado pela configuração.
- Taxas derivadas de contadores arredondados (resolução de 0,1M de documentos) são apresentadas com mais precisão do que os dados permitem, em especial a segunda taxa de cada rodada.
- A análise cobre os artefatos recebidos: um único nó nos logs, cinco pontos de métrica a cada 30 minutos e uma configuração declarativa, sem confirmação dos parâmetros efetivamente aplicados.
- O limite de cerca de 900 palavras é uma orientação ao modelo. Não foi medido o tamanho das saídas abaixo.
- O escopo é Elasticsearch sobre JVM. Para outros motores de busca ou indexação, o método vale, mas o papel e as heurísticas do prompt podem precisar de ajuste.
- Os resultados abaixo vêm de duas execuções com um único modelo.
- O gate (seção "Gate de qualidade com LLM-as-judge") usa um juiz LLM não determinístico, calibrado com cinco amostras, três delas sintéticas. As saídas reais do prompt ficam perto do corte de 6, então uma execução pode reprovar por flutuação do juiz ou do prompt.

## Execução

| Item | Valor |
|---|---|
| Modelo | GPT 6 Sol |
| Identificador na API | `gpt-6-sol` |
| Esforço de raciocínio | médio |

O único parâmetro, `pacote_artefatos`, foi preenchido com o pacote abaixo nas duas rodadas. Cada rodada foi executada em um chat novo e temporário, sem acesso à memória.

### Entrada: degradação de busca e indexação do Cerebro

Valores dos parâmetros:

```text
pacote_artefatos:
CONFIGURAÇÃO (cerebro.yaml, repositório de infra):
# cerebro.yaml — configuração do cluster Elasticsearch (repositório de infra)
cerebro:
  shards: 12
  replicas_per_shard: 1
  jvm_heap: 8g
  refresh_interval: 1s
  reindex_job:
    schedule: "0 2 * * *"     # roda todo dia às 02:00
    avg_duration_min: 90      # normalmente termina ~03:30
  query_cache:
    enabled: true
    size_mb: 512

MÉTRICAS (dashboard do Sentinel, pontos de 30 em 30 minutos):
timestamp              search_p99_ms   indexed_docs_per_s   heap_used_pct   cache_hit_pct
2026-05-13 08:00 UTC   850             4200                 61              74
2026-05-13 08:30 UTC   1100            4100                 68              71
2026-05-13 09:00 UTC   2300            9800                 79              58
2026-05-13 09:30 UTC   4100            11200                88              41
2026-05-13 10:00 UTC   6700            12400                94              29

LOGS (kubectl logs do pod cerebro-node-3, horário em UTC):
[2026-05-13T08:02:11,540][INFO ][o.e.t.LoggingTaskListener              ][cerebro-node-3] reindex task [88123] (scheduled 02:00) progress, created [3.8M]/[10M] docs (38%)
[2026-05-13T08:14:33,019][WARN ][o.e.m.j.JvmGcMonitorService            ][cerebro-node-3] [gc][young] duration [620ms], collections [1] in [10s], heap [4.9gb]->[3.1gb]/[8gb]
[2026-05-13T08:41:07,222][INFO ][o.e.i.IndexingMemoryController         ][cerebro-node-3] now throttling indexing for shard [logs-2026.05][7]: segment writing can't keep up
[2026-05-13T09:03:55,810][WARN ][o.e.t.ThreadPool                       ][cerebro-node-3] write thread pool queue at [150/200]
[2026-05-13T09:12:48,402][WARN ][o.e.m.j.JvmGcMonitorService            ][cerebro-node-3] [gc][old] duration [1.1s], collections [2] in [60s], heap [6.3gb]->[5.9gb]/[8gb]
[2026-05-13T09:20:02,118][INFO ][o.e.t.LoggingTaskListener              ][cerebro-node-3] reindex task [88123] (scheduled 02:00) progress, created [4.0M]/[10M] docs (40%)
[2026-05-13T09:31:17,653][WARN ][o.e.i.b.HierarchyCircuitBreakerService ][cerebro-node-3] [parent] usage [6.9gb/8gb] (86%), approaching limit
[2026-05-13T09:44:29,901][WARN ][o.e.s.SearchService                    ][cerebro-node-3] slow query on shard [logs-2026.05][7] took [2380ms]
[2026-05-13T09:51:08,377][WARN ][o.e.t.ThreadPool                       ][cerebro-node-3] write thread pool queue at [188/200]
[2026-05-13T09:58:41,102][WARN ][o.e.t.ThreadPool                       ][cerebro-node-3] write thread pool queue full [200/200], rejecting bulk — EsRejectedExecutionException
[2026-05-13T09:58:43,210][WARN ][o.e.m.j.JvmGcMonitorService            ][cerebro-node-3] [gc][old] duration [1.8s], collections [4] in [60s], heap [7.6gb]->[7.4gb]/[8gb]
[2026-05-13T09:58:44,005][INFO ][o.e.t.LoggingTaskListener              ][cerebro-node-3] reindex task [88123] (scheduled 02:00) still running, created [4.1M]/[10M] docs (41%), ETA unknown
[2026-05-13T09:58:45,889][WARN ][o.e.i.IndexingMemoryController         ][cerebro-node-3] indexing buffer above limit, throttling shard [logs-2026.05][7]
[2026-05-13T09:58:46,330][DEBUG][o.e.s.query.QueryPhase                 ][cerebro-node-3] shard [logs-2026.05][7] search took [5031ms] (timeout [5000ms])
[2026-05-13T09:58:46,512][ERROR][o.e.s.SearchService                    ][cerebro-node-3] search returned partial results: 11/12 shards succeeded
[2026-05-13T09:58:46,701][WARN ][o.e.i.b.HierarchyCircuitBreakerService ][cerebro-node-3] [parent] circuit breaker tripped, usage [7.7gb/8gb] (96%) over limit
[2026-05-13T09:58:47,001][WARN ][o.e.i.cache.query.IndicesQueryCache    ][cerebro-node-3] query cache eviction rate spiking, hit_ratio dropped to 0.29
[2026-05-13T10:01:12,778][ERROR][o.e.s.SearchService                    ][cerebro-node-3] CircuitBreakingException: [parent] Data too large, would be [7.9gb/8gb]
[2026-05-13T10:03:39,255][WARN ][o.e.t.ThreadPool                       ][cerebro-node-3] write thread pool rejected [1284] bulk requests in last 5min
[2026-05-13T10:05:50,640][ERROR][o.e.s.SearchService                    ][cerebro-node-3] all shards failed for index [logs-2026.05] on 3 of last 20 queries
```

Saída da rodada 1:

```markdown
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
```

Saída da rodada 2:

```markdown
## 1. Resumo

(Hipótese) A principal explicação é a concorrência entre a tarefa prolongada de `reindex` e a carga de indexação, possivelmente agravada por um gargalo de escrita de segmentos no shard `logs-2026.05[7]`. O mecanismo provável envolve `throttling`, acúmulo na `write thread pool`, pressão de heap, GC e degradação das buscas. [configuração] [métricas] [logs]

Confiança: média. Há convergência entre métricas e logs, mas a origem da sobrecarga não está demonstrada, e existe divergência importante entre o progresso do `reindex` e a taxa agregada de indexação.

## 2. Linha do tempo

Referência: UTC para métricas e logs. O fuso do agendamento não está especificado.

- 08:00–08:02 (observado): `search_p99_ms` de 850 ms, `heap_used_pct` de 61% e `cache_hit_pct` de 74%. O `reindex` já registra 38% de progresso. [métricas] [logs]
- 08:14 (observado): `young GC` de 620 ms. [logs]
- 08:30–08:41 (observado): p99 chega a 1100 ms; começa `throttling indexing`, com a mensagem `segment writing can't keep up`. [métricas] [logs]
- 09:00–09:12 (observado): `indexed_docs_per_s` atinge 9800; fila de escrita em 150/200; `old GC` de 1,1 s. [métricas] [logs]
- 09:20 (observado): `reindex` em 40%, sem conclusão. [logs]
- 09:30–09:51 (observado): p99 de 4100 ms, heap de 88%, `parent` em 86%, `slow query` e fila em 188/200. [métricas] [logs]
- 09:58–10:05 (observado): rejeições `EsRejectedExecutionException`, `old GC` de 1,8 s, `reindex` em 41%, timeout de busca, resultados parciais, `circuit breaker tripped`, queda do cache, `CircuitBreakingException` e falhas de shards. [logs]
- 10:00 (observado): p99 de 6700 ms, indexação de 12400 docs/s, heap de 94% e cache hit de 29%. [métricas]

Divergência relevante (inferido): Entre 08:02 e 09:20, o `reindex` avança aproximadamente 200 mil documentos, equivalentes a 43 docs/s; entre 09:20 e 09:58, aproximadamente 100 mil, novamente 43 docs/s. Para 10 milhões de documentos em 90 minutos, a média necessária seria aproximadamente 1852 docs/s. [configuração] [logs]

Essas taxas são muito inferiores aos 4100–12400 `indexed_docs_per_s` apresentados pelo dashboard. [métricas] A divergência pode decorrer de escopos diferentes, mas não está resolvida. Os pontos de 30 minutos também impedem determinar precisamente o início da degradação.

## 3. Cadeia causal

1. Origem — hipótese: carga concorrente de indexação e `reindex` prolongado, com possível limitação na escrita de segmentos. O `reindex` estava ativo e o shard apresentava dificuldade de escrita, mas a causa dessa dificuldade é desconhecida. [logs]
2. Mecanismo intermediário (observado/inferido): `throttling indexing` e crescimento da `write thread pool queue` indicam perda de capacidade de acompanhar a demanda de escrita. [logs]
3. Mecanismo intermediário (inferido): pressão crescente de memória, GC e evicções do `query cache` ampliam a disputa por recursos. A direção exata dessas relações não está comprovada. [métricas] [logs]
4. Consequências (observado): aumento de `search_p99_ms`, rejeições de bulk, timeouts, resultados parciais, acionamento do circuit breaker e falhas de buscas. [métricas] [logs]

O elo entre `reindex`, carga agregada e gargalo efetivo de escrita permanece não comprovado. Portanto, não há causa-raiz definitivamente estabelecida.

## 4. Causa versus consequência

| Fenômeno                           | Papel                   | Evidência                                                                        |
| ---------------------------------- | ----------------------- | -------------------------------------------------------------------------------- |
| Origem da carga excessiva          | indeterminado           | Crescimento de `indexed_docs_per_s`, sem identificação dos produtores [métricas] |
| `reindex` prolongado               | indeterminado           | Tarefa ativa e progresso lento [logs]                                            |
| `segment writing can't keep up`    | mecanismo intermediário | `IndexingMemoryController` [logs]                                                |
| `write thread pool queue` saturada | mecanismo intermediário | 150/200 até 200/200 [logs]                                                       |
| Pressão de heap e GC               | mecanismo intermediário | Heap crescente e `old GC` [métricas] [logs]                                      |
| Evicções do `query cache`          | mecanismo intermediário | `hit_ratio` em 0.29 [logs]                                                       |
| Rejeições de bulk                  | consequência            | `EsRejectedExecutionException` [logs]                                            |
| Latência e falhas de busca         | consequência            | p99, timeout, resultados parciais e `CircuitBreakingException` [métricas] [logs] |

## 5. Hipóteses alternativas consideradas

Hipótese A — Gargalo primário de armazenamento ou escrita de segmentos.

- A favor (observado): `segment writing can't keep up` antecede a saturação da fila. [logs]
- Contra: não existem métricas de I/O, latência de disco ou merges para demonstrar o gargalo físico.
- Situação: mantida em aberto. Poderia explicar tanto o progresso lento do `reindex` quanto as rejeições.

Hipótese B — Pressão primária de memória/JVM.

- A favor (observado): GC, crescimento do heap e acionamento do circuit breaker. [métricas] [logs]
- Contra: o `throttling` por escrita de segmentos já aparece antes dos eventos mais graves de memória.
- Situação: mantida como possível fator agravante, não como causa-raiz demonstrada.

Hipótese C — Aumento independente da carga de indexação.

- A favor (observado): aumento substancial de `indexed_docs_per_s`. [métricas]
- Contra: não há identificação dos produtores, origem dos documentos ou separação do tráfego de `reindex`.
- Situação: mantida em aberto.

## 6. Ações recomendadas

### Contenção imediata

- (Recomendação) Avaliar pausar temporariamente ou reduzir a concorrência do `reindex`. Resolve potencial disputa por escrita e memória. Risco: atraso na atualização dos dados. Reversibilidade: alta, mediante retomada controlada e validação do estado da tarefa. Requer aprovação humana. Evidência: tarefa prolongada, `throttling` e fila cheia. [logs]
- (Recomendação) Identificar e limitar temporariamente produtores não prioritários de bulk, se confirmados como responsáveis pela carga. Risco: atraso na ingestão. Reversibilidade: alta. Requer aprovação humana. Evidência: crescimento de `indexed_docs_per_s` e rejeições. [métricas] [logs]

### Correção

- (Recomendação) Investigar I/O, merges e escrita de segmentos do shard afetado e corrigir o gargalo identificado. Risco: depende da intervenção. Reversibilidade: depende da correção. Mudanças de capacidade exigem aprovação humana. Evidência: `segment writing can't keep up`. [logs]
- (Recomendação) Ajustar a concorrência ou a programação do `reindex` somente após medir sua contribuição real. Risco: maior duração do processamento. Reversibilidade: alta. Requer aprovação humana. Evidência: duração configurada e progresso observado. [configuração] [logs]

### Prevenção

- (Recomendação) Implantar alertas correlacionando progresso do `reindex`, filas de escrita, GC, heap e latência. Resolve: detecção tardia. Risco: ruído de alertas. Reversibilidade: alta. Evidência: degradação progressiva nas duas fontes. [métricas] [logs]

## 7. O que os dados não permitem concluir

(Observado) O pacote contém configuração, métricas e logs legíveis. A configuração não comprova os parâmetros efetivamente aplicados; as métricas cobrem 08:00–10:00 UTC e os logs de um único pod cobrem 08:02–10:05 UTC. [configuração] [métricas] [logs]

Permanecem indefinidos:

- O fuso de `schedule`, o início efetivo do `reindex` e sua relação com a duração esperada.
- O motivo da divergência entre o progresso da tarefa e `indexed_docs_per_s`.
- O estado dos demais nós, a distribuição das escritas e a origem da carga.
- A existência de saturação de disco, merges ou limites de I/O.
- A configuração efetiva e os valores de parâmetros não presentes no arquivo.

Coletas prioritárias: métricas de disco e merges por nó e shard, histórico completo da tarefa, contadores de indexação separados por origem, estatísticas de filas e rejeições, heap/GC por nó e configuração efetiva do cluster.

Conclusão: a sobrecarga de escrita concorrente ao `reindex` é a hipótese principal, mas distinguir entre excesso de demanda, limitação de I/O e pressão primária de memória exige essas evidências adicionais.
```

### Gate de qualidade com LLM-as-judge (CP09)

A análise de causa-raiz não tem resposta única verificável por regex, então a qualidade é avaliada por um juiz LLM que aplica uma rubrica de quatro critérios. O juiz é o gate: reprova a saída quando a nota total fica abaixo do corte. Arquivos desta pasta:

| Arquivo | Função |
|---|---|
| [`promptfooconfig.yaml`](promptfooconfig.yaml) | gate: executa o prompt em dois provedores e submete cada saída ao juiz |
| [`juiz/rubrica.txt`](juiz/rubrica.txt) | rubrica e instruções do juiz (texto completo) |
| [`dados/pacote-cerebro.txt`](dados/pacote-cerebro.txt) | pacote de artefatos usado como entrada (o mesmo da seção anterior) |
| [`calibracao/`](calibracao) | config, amostras e pontuação humana usados para calibrar o juiz |

**Rubrica.** Quatro critérios, cada um de 0 a 2 (0 = não atende, 1 = parcial, 2 = atende), total de 0 a 8. **Aprovação: total ≥ 6 e nenhum critério igual a 0.**

| Critério | 2 (atende) | 1 (parcial) | 0 (não atende) |
|---|---|---|---|
| C1. Causa-raiz correta | aponta o reindex travado ou prolongado (progresso incompatível com a duração configurada) como origem provável e o liga à saturação de heap, circuit breaker e timeouts; hipótese principal com confiança declarada conta como 2 | cita reindex ou heap sem priorizar o reindex como origem, ou acerta o mecanismo de memória sem apontar a origem | só sintomas, ou aponta como causa algo que é consequência (cache, fila, timeouts) |
| C2. Correlação × causa | separa causa, mecanismo intermediário e consequência, trata cache, rejeições, circuit breaker e timeouts como efeitos e distingue precedência de causalidade | separa parte dos fenômenos, mas mistura ao menos um efeito com causa | trata sintoma como causa-raiz ou não separa |
| C3. Ação proporcional | ações coerentes com o diagnóstico, com risco e reversibilidade, sem ação destrutiva como primeira medida, sem valores novos sem base e sem redesenho | em geral coerentes, mas genéricas ou sem risco ou reversibilidade | só "monitorar", ação destrutiva, redesenho de arquitetura ou valores inventados |
| C4. Honestidade epistêmica | confiança coerente, marcação de observado, inferido e hipótese, contradições e lacunas registradas, lista do que coletar | reconhece alguns limites, com confiança incoerente ou lacunas principais ausentes | certeza que os dados não permitem, dados inventados ou nenhum limite reconhecido |

A rubrica completa leva também o gabarito do cenário (o reindex 88123 avançou de 38% para 41% em cerca de 2 h, aproximadamente 43 docs/s, contra 90 min esperados), que o juiz usa como referência.

**Configuração do gate.**

| Item | Valor |
|---|---|
| Versão do promptfoo | 0.125.1 |
| Prompt testado | `prompt.md` v1.0.0 (arquivo inteiro, com frontmatter) |
| Provedor 1 | OpenAI `gpt-6-sol`, `reasoning_effort: medium` |
| Provedor 2 | Anthropic `claude-sonnet-5-5`, `thinking.type: between_tools`, `max_tokens: 16000` |
| Juiz | Anthropic `claude-sonnet-5-5`, `thinking.type: between_tools`, `max_tokens: 8000`, o mesmo para os dois provedores |
| Assert | um `llm-rubric` com `threshold: 0.75` (equivale ao corte total ≥ 6) |

Comando, a partir desta pasta:

```powershell
promptfoo eval -c promptfooconfig.yaml --no-cache --output resultado.json
```

O juiz faz uma única chamada por saída e devolve JSON com `score = total/8` e `pass = (total ≥ 6 e nenhum critério igual a 0)`. A nota de cada critério vem no campo `reason` (`C1=…; C2=…; C3=…; C4=…; total=…`). Com `threshold` definido, o promptfoo exige que `pass` seja verdadeiro e `score ≥ 0.75`, e a regra "nenhum critério zerado" fica dentro do `pass`.

**Calibração do juiz.** Antes de usar o juiz como gate, pontuei cinco saídas à mão, critério por critério, **antes de ver as notas do juiz**, e comparei com as notas dele. A meta do enunciado: no máximo 1 ponto de diferença em cada critério.

| Amostra | Origem |
|---|---|
| s1, s2 | rodadas reais do GPT 6 Sol, mostradas na seção Execução acima |
| s3 | sintética: trata a queda do cache como causa-raiz, com certeza fabricada e reinício do cluster como primeira ação |
| s4 | sintética: aponta o reindex travado, mas propõe apagar o índice, reiniciar todos os nós e redesenhar a arquitetura |
| s5 | sintética: análise vaga, tudo "indeterminado" e só "monitorar" |

Pontuação humana contra a do juiz (C1 · C2 · C3 · C4 = total). As duas rodadas do juiz deram notas idênticas.

| Amostra | Humano | Juiz | Diferença por critério | Veredicto humano | Veredicto do juiz |
|---|---|---|---|---|---|
| s1 | 1 · 1 · 2 · 2 = 6 | 1 · 2 · 1 · 2 = 6 | 0 · 1 · 1 · 0 | aprovada | aprovada |
| s2 | 2 · 1 · 1 · 2 = 6 | 1 · 2 · 2 · 2 = 7 | 1 · 1 · 1 · 0 | aprovada | aprovada |
| s3 | 0 · 0 · 0 · 0 = 0 | 0 · 1 · 0 · 0 = 1 | 0 · 1 · 0 · 0 | reprovada | reprovada |
| s4 | 2 · 2 · 0 · 1 = 5 | 2 · 2 · 0 · 2 = 6 | 0 · 0 · 0 · 1 | reprovada (C3 = 0) | reprovada (C3 = 0) |
| s5 | 0 · 0 · 0 · 1 = 1 | 0 · 0 · 0 · 1 = 1 | 0 · 0 · 0 · 0 | reprovada | reprovada |

Resultado: as 20 diferenças ficaram entre 0 e 1 e os 5 veredictos coincidem, sem ajuste na rubrica. As notas humanas e os motivos estão em [`calibracao/humano.yaml`](calibracao/humano.yaml).

**Execução do gate** (`Results: 2 passed (100%), 0 failed, 0 errors`):

| Modelo | Resultado | Nota do juiz (C1 · C2 · C3 · C4 = total) | Latência | Custo da geração (US$) | Palavras |
|---|---|---|---|---|---|
| gpt-6-sol | PASS (score 0,875) | 1 · 2 · 2 · 2 = 7 | 20,1 s | 0,0295 | 944 |
| claude-sonnet-5-5 | PASS (score 1,0) | 2 · 2 · 2 · 2 = 8 | 27,0 s | 0,0498 | 1250 |

A contagem de palavras divide a saída por espaços e inclui as tabelas. O custo não inclui as chamadas do juiz.

Justificativa do juiz, trechos do `resultado.json` com a acentuação restaurada (o arquivo gravou parte dos acentos corrompidos):

- **GPT 6 Sol, C1 = 1:** a origem é declarada "não demonstrada/indeterminada" e a hipótese principal é contenção na escrita de segmentos, com o reindex apenas como possível agravante, em vez de ser priorizado como origem provável, embora o mecanismo de heap, GC, circuit breaker e timeouts esteja corretamente descrito.
- **GPT 6 Sol, C2 a C4 = 2:** a tabela e a cadeia causal separam origem, mecanismo e consequência e dizem que precedência não demonstra causalidade; as ações trazem risco, reversibilidade e aprovação humana, sem valores inventados; a confiança baixa é justificada, com as lacunas reconhecidas (nó único, métricas de 30 min, sem I/O, sem configuração efetiva) e a lista do que coletar.
- **Sonnet 5.5, C1 = 2:** identifica o reindex task [88123], ainda ativo muito além dos cerca de 90 min esperados, como hipótese principal, nota o progresso incompatível (38% para 41%, cerca de 43 docs/s) e o liga ao mecanismo de saturação de heap, throttling, fila de escrita, circuit breaker e timeouts, com confiança média declarada.

## Curadoria

**Framework:** raciocínio estruturado por hipóteses concorrentes, sem few-shot. O objetivo é chegar à causa-raiz cruzando três fontes, e exemplos de análise completa ancorariam o modelo em uma causa específica e prejudicariam o reuso. Os blocos do prompt mapeiam assim: `<metodo>` define as etapas de análise (coerência entre fontes, linha do tempo, primeiro desvio, hipóteses concorrentes, classificação, critério de causa-raiz e confiança); `<regras>` cobre rastreabilidade, proporcionalidade das ações, segurança da entrada e entrada insuficiente; `<formato_saida>` fixa as sete seções.

**Refinos:** o prompt foi criado por meta-prompting com o Opus 5.5 e refinado com revisão humana. O limite de palavras subiu de 700 para 900, por ser incompatível com os campos exigidos em cada ação, e a exceção da resposta de entrada insuficiente foi explicitada no formato de saída. Três execuções com um único artefato cada, que não constam como entrega, mostraram que nenhuma fonte isolada fecha a análise. A primeira execução com o pacote completo não derivou a taxa de progresso do reindex, e por isso a etapa de coerência entre fontes passou a pedir a taxa implícita de contadores acumulados e sua comparação com a configuração e com as demais métricas. Não foi aplicado novo ajuste para induzir uma causa específica, para manter o prompt reutilizável em outras degradações.

**Modelo:** GPT 6 Sol com esforço de raciocínio médio. Nas duas rodadas seguiu as sete seções, citou a fonte de cada valor, classificou o cache, o circuit breaker e as rejeições como mecanismo ou consequência, e registrou a divergência entre o reindex e a vazão de indexação. A conclusão sobre o reindex variou entre as rodadas, e a confiança foi média em ambas.

**Curadoria do gate (CP09):**

- **Desenho do juiz:** a primeira ideia foi uma asserção `llm-rubric` por critério, agrupadas em `assert-set` com limiar 0,75 (equivalente a total ≥ 6). Testei o `assert-set` com o provedor `echo` e as notas 0, 2, 2 e 2 passaram: o conjunto mede só a média e não aplica "nenhum critério zerado". Para manter a regra sem duplicar chamadas ao juiz, usei uma única chamada que devolve o total e o `pass` já com a regra.
- **Calibração:** pontuei as cinco amostras antes de rodar o juiz. A s1 foi revista de 2 · 2 · 2 · 2 para 1 · 1 · 2 · 2 ao reler as âncoras de C1 e C2, ainda antes de ver qualquer nota do juiz. O juiz ficou a no máximo 1 ponto em todos os critérios e acertou os cinco veredictos nas duas rodadas, com notas idênticas, então a rubrica não precisou de ajuste.
- **Onde o juiz diverge:** nas duas saídas reais (s1 e s2) ele deu 2 em C2, onde dei 1, por aceitar o cache como "mecanismo intermediário" quando o texto diz que a causalidade não está comprovada; em C3 ele foi mais severo na s1 (ações "hedgeadas") e mais brando na s2 (não penalizou o "risco depende da intervenção"); e em C1 deu 1 na s2, onde dei 2. As diferenças se compensam em parte no total (s1 igual, s2 com 1 ponto a mais), mas essas duas amostras estão no corte e é onde um desvio de 1 ponto muda o veredicto.
- **Problema de execução corrigido:** a primeira execução da calibração deu erro nos cinco testes, sem chamar o juiz, porque o `file://` de asserções do promptfoo não aceita `.md`. A rubrica passou a `juiz/rubrica.txt`.
- **O que o gate mostrou:** o prompt v1.0.0 passou nos dois provedores (7 e 8 de 8), sem alteração do prompt. O GPT ficou com C1 = 1 porque priorizou a escrita de segmentos, e o reindex ficou como agravante.
- **Limites:** cinco amostras, três delas escritas por mim; as duas reais vêm do mesmo prompt e do mesmo modelo. O juiz é o Sonnet 5.5 e também avaliou a saída do Sonnet, o que pode favorecer a si mesmo; com uma execução de cada, a nota 8 contra 7 não permite concluir isso. A rubrica não mede tamanho (o Sonnet passou de 1200 palavras contra o limite de cerca de 900 do prompt) nem rastreabilidade (o Sonnet citou "Tasks API" e `requests_per_second`, que não constam dos artefatos).
- **Próximo passo (CP10):** as saídas reais ficam perto do corte, então o pipeline precisa decidir como tratar a flutuação do juiz (limiar, repetição ou só asserts determinísticos como bloqueio) e pode somar asserts de tamanho e de rastreabilidade ao gate.
