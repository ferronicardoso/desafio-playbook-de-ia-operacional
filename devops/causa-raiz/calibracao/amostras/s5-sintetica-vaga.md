## 1. Resumo

O cluster Cerebro está degradado, com latência de busca alta, heap alto, rejeições de escrita e falhas de shards. Não é possível dizer a causa. Confiança: baixa.

## 2. Linha do tempo

- Entre 08:00 e 10:00 a latência de busca subiu.
- O heap subiu.
- Houve rejeições de bulk e circuit breaker.

## 3. Cadeia causal

Vários componentes estão com problemas ao mesmo tempo.

## 4. Causa versus consequência

| Fenômeno | Papel | Evidência |
| --- | --- | --- |
| Latência de busca alta | indeterminado | métricas |
| Heap alto | indeterminado | métricas |
| Rejeições de bulk | indeterminado | logs |
| Circuit breaker | indeterminado | logs |

## 5. Hipóteses alternativas consideradas

Pode ser carga, memória ou disco.

## 6. Ações recomendadas

### Contenção imediata
- Monitorar o cluster.

### Correção
- Monitorar e avaliar.

### Prevenção
- Continuar monitorando.

## 7. O que os dados não permitem concluir

Os dados não são suficientes.
