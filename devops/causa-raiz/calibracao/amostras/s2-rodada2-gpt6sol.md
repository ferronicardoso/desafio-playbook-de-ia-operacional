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
