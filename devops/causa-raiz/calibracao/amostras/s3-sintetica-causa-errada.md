## 1. Resumo

A causa-raiz é a queda do cache de queries: o `cache_hit_pct` caiu de 74% para 29% [métricas] e, sem cache, todas as buscas passaram a bater no disco, o que derrubou o cluster. Confiança: alta.

## 2. Linha do tempo

- 08:00: `cache_hit_pct` em 74% (observado) [métricas]
- 09:00: `cache_hit_pct` em 58%, `search_p99_ms` em 2300 (observado) [métricas]
- 09:58: fila `write` cheia em 200/200 e `EsRejectedExecutionException` (observado) [logs]
- 10:00: `cache_hit_pct` em 29%, `heap_used_pct` em 94% (observado) [métricas]

## 3. Cadeia causal

Queda do cache de queries, que causa buscas mais lentas, que causam fila de escrita cheia, que causa rejeição de bulk e circuit breaker.

## 4. Causa versus consequência

| Fenômeno | Papel | Evidência |
| --- | --- | --- |
| Queda do `cache_hit_pct` | causa-raiz | 74% para 29% [métricas] |
| Fila `write` cheia | consequência | 200/200 [logs] |
| Circuit breaker | consequência | `parent` acionado [logs] |
| Heap alto | consequência | 94% [métricas] |

## 5. Hipóteses alternativas consideradas

Nenhuma. A correlação temporal entre a queda do cache e o aumento da latência é conclusiva.

## 6. Ações recomendadas

### Contenção imediata
- Reiniciar o cluster inteiro para limpar o cache e o heap.

### Correção
- Aumentar o `query_cache.size_mb` para 2048.

### Prevenção
- Adicionar mais nós ao cluster.

## 7. O que os dados não permitem concluir

Nada relevante: os dados são suficientes para a conclusão acima.
