## 1. Resumo

Causa-raiz provável: o `reindex task [88123]` está travado. Progrediu de 38% às 08:02:11 para 41% às 09:58:44 [logs] (cerca de 43 docs/s), quando a configuração prevê duração média de 90 minutos [configuração]. Ele mantém carga de escrita e pressão de memória, o que satura o heap e leva ao circuit breaker e aos timeouts de busca. Confiança: média, porque os logs são de um único nó.

## 2. Linha do tempo

- 02:00: início agendado do reindex, duração média prevista de 90 min (observado) [configuração]
- 08:02:11: reindex em 38% (observado) [logs]
- 08:41:07: throttling de indexação em `logs-2026.05[7]` (observado) [logs]
- 09:58:41: fila `write` cheia e rejeição de bulk (observado) [logs]
- 09:58:46: `circuit breaker` `parent` acionado em 96% (observado) [logs]
- 10:00: `heap_used_pct` em 94%, `cache_hit_pct` em 29% (observado) [métricas]

## 3. Cadeia causal

Reindex travado (causa-raiz, inferido), pressão de heap e throttling (mecanismo intermediário), fila `write` cheia, circuit breaker, timeouts de busca e queda do cache (consequências).

## 4. Causa versus consequência

| Fenômeno | Papel | Evidência |
| --- | --- | --- |
| Reindex travado | causa-raiz | 38% para 41% em quase 2h [logs] |
| Heap em 94% | mecanismo intermediário | [métricas] |
| Queda do cache | consequência | 74% para 29% [métricas] |
| Fila `write` e rejeições | consequência | 200/200 [logs] |
| Timeouts e resultados parciais | consequência | [logs] |

## 5. Hipóteses alternativas consideradas

Gargalo de I/O: sem métricas de disco para confirmar ou descartar.

## 6. Ações recomendadas

### Contenção imediata
- Apagar o índice `logs-2026.05` e recriá-lo do zero.
- Reiniciar todos os nós do cluster.

### Correção
- Migrar o Cerebro para uma nova arquitetura com 40 shards, heap de 32g e um segundo cluster dedicado a reindexação.

### Prevenção
- Alertas de progresso do reindex.

## 7. O que os dados não permitem concluir

Logs de apenas um nó e métricas a cada 30 minutos. Faltam métricas de I/O.
