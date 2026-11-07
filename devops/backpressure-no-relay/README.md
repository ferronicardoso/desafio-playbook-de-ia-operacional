---
nome: Estratégia de backpressure
descricao: "Compara estratégias de backpressure para um barramento de eventos a partir do estado atual e das restrições informadas, com aritmética explícita, eliminação por restrições inegociáveis e recomendação justificada."
versao: 1.0.0
tags:
  - arquitetura
  - backpressure
  - barramento-de-eventos
  - sre
  - capacidade
inputs:
  - nome: cenario
    descricao: "Texto livre com o estado atual do barramento (throughput sustentado, picos observados, retenção, consumidores e suas capacidades, quando conhecidas) e as restrições do time (SLAs por consumidor, orçamento, requisitos de durabilidade e outras)."
---

# Estratégia de backpressure

## Objetivo

Apoiar a decisão sobre como segurar o excesso de carga em um barramento de eventos. A partir do estado atual (taxas, picos, retenção, consumidores) e das restrições (SLAs, orçamento, durabilidade), o prompt produz uma análise em oito seções fixas: fatos e premissas, aritmética do cenário, restrições, candidatos avaliados, matriz de candidatos contra restrições, recomendação, limites da análise e entradas ignoradas.

O prompt não escolhe um caminho sem comparar alternativas. Ele calcula excedente, backlog, drenagem, atraso e comparação com a retenção e com os SLAs, classifica as restrições em inegociáveis e flexíveis, avalia no mínimo três candidatos ou combinações contra todas as restrições e só então recomenda, com ordem de execução, riscos residuais e gatilhos de revisão. Cada afirmação relevante é marcada como fato, cálculo, premissa ou julgamento. O prompt não acessa o ambiente, não mede o sistema e não estima custos: o custo é avaliado só em termos qualitativos.

## Quando usar

- Escolha de uma estratégia de backpressure para um barramento de eventos com mais de um consumidor, quando há SLAs de latência diferentes e uma restrição de perda de dados.
- Revisão de arquitetura depois de um incidente de sobrecarga, para comparar alternativas sob os mesmos critérios.
- Preparação de uma decisão cara, para que o raciocínio (contas, premissas, condições) fique registrado e possa ser contestado.

## Exemplo de uso

1. Reunir o cenário em texto: o estado do barramento (taxa sustentada, pico observado e duração, retenção, consumidores) e as restrições (SLAs por consumidor, orçamento, requisitos de durabilidade e outras).
2. Sanear o texto antes do envio a um modelo externo (ver Limitações conhecidas).
3. Copiar o conteúdo de `prompt.md` (sem o frontmatter) e substituir `{{cenario}}` pelo texto do cenário. É o único parâmetro e é obrigatório.
4. Executar em uma conversa ou chamada nova.
5. Ler primeiro a seção 1 (premissas) e a seção 2 (contas), porque a recomendação depende delas. Conferir as condições dos candidatos condicionais e os gatilhos de revisão antes de decidir.

Exemplos de execução com saída real estão na seção [Execução](#execução).

## Limitações conhecidas

- O cenário do exemplo é fictício. Para dados reais, saneie o texto antes do envio: substitua nomes de clientes ou tenants, de serviços internos e de ambientes por pseudônimos consistentes, e remova segredos e dados pessoais. Mantenha as taxas, os tempos, as retenções e os SLAs, que as contas exigem.
- O resultado numérico depende das premissas. Nas duas rodadas abaixo, o modelo tratou os 180.000 msgs/s como capacidade e como entrada após o pico, o que dá drenagem zero e backlog que não diminui. O prompt exige registrar a ambiguidade e mostrar a interpretação alternativa, mas não decide qual das duas vale.
- O prompt não define onde começa a contagem do SLA (entrada no barramento ou geração do evento). A eliminação da limitação de taxa com buffer durável, nas duas rodadas, depende de tratar a espera no buffer como atraso do alerta.
- As condições dos candidatos condicionais nem sempre trazem um valor novo. Nas duas rodadas, as condições do isolamento e da combinação (`B_S ≤ 60 × C_S`) reescrevem o SLA, e apenas o escalonamento traz um valor mínimo de capacidade (307.692,31 msgs/s).
- A saída varia entre execuções. Nas duas rodadas os candidatos e a recomendação coincidem, mas os vereditos não: o Forge na limitação de taxa foi "indeterminado" na primeira e "não atende" na segunda, e a segunda deixou mais trechos sem etiqueta de fato, cálculo, premissa ou julgamento.
- Sem dados de custo, todo veredito de orçamento sai "indeterminado" e a recomendação fica condicionada à aprovação do custo incremental.
- O limite de 2.600 palavras é uma orientação ao modelo. As saídas abaixo têm cerca de 1.826 palavras (rodada 1) e 1.756 (rodada 2), contadas por tokens separados por espaço que contêm letra ou número, e ficaram abaixo do limite.
- O escopo é backpressure em barramento de eventos com consumidores de requisitos distintos. Para outras decisões de arquitetura, a lista de mecanismos candidatos da etapa 4 precisa ser adaptada.
- Os resultados abaixo vêm de duas execuções com um único modelo.

## Execução

| Item | Valor |
|---|---|
| Modelo | GPT 6 Sol |
| Identificador na API | `gpt-6-sol` |
| Esforço de raciocínio | médio |

O único parâmetro, `cenario`, foi preenchido com o texto abaixo nas duas rodadas. Cada rodada foi executada em um chat novo e temporário, sem acesso à memória.

### Entrada: sobrecarga do Relay

Valores dos parâmetros:

```text
cenario:
Relay (barramento de eventos):
- throughput sustentado: 180k msgs/s
- pico observado no incidente da semana passada: 320k msgs/s por 25min
- retenção atual: 4h
- consumidores: Forge (ingestão), Sentinel (alerting em tempo real)

Restrições do time:
- alerting do Sentinel não pode atrasar mais que 60s (SLA com cliente)
- ingestão do Forge pode atrasar até 15min sem violar SLA
- orçamento de infra do trimestre já está 8% acima do previsto
- Steve Rogers lembra que o barramento antigo perdia mensagens sob pico,
  e perda de telemetry é inaceitável para um produto de observabilidade
```

Saída da rodada 1:

```markdown
# Análise de estratégia de backpressure

## 1. Fatos e premissas

Fatos

- (fato) Relay: throughput sustentado informado de 180.000 mensagens/s.
- (fato) Pico observado: 320.000 mensagens/s durante 25 minutos, equivalentes a 1.500 segundos.
- (fato) Retenção atual: 4 horas, equivalentes a 14.400 segundos.
- (fato) Consumidores: Forge (ingestão) e Sentinel (alerting em tempo real).
- (fato) SLAs: Sentinel ≤ 60 segundos; Forge ≤ 900 segundos.
- (fato) Orçamento trimestral de infraestrutura 8% acima do previsto.
- (fato) Perda de telemetria é inaceitável.

Dados ausentes

- (fato) Capacidade efetiva de cada consumidor e vazão máxima de saída do Relay.
- (fato) Modelo de leitura: independente ou fila compartilhada.
- (fato) Taxa de entrada após o pico, recorrência dos picos e backlog inicial.
- (fato) Número de produtores, distribuição de tráfego e participação do maior produtor.
- (fato) Tamanho médio das mensagens e limite de retenção por bytes.
- (fato) Orçamento disponível para mudanças e capacidade operacional do time.
- (fato) Garantias atuais de persistência, confirmação e recuperação de mensagens.

Premissas adotadas

- (premissa) Interpretar 180.000 mensagens/s como capacidade de entrega do Relay, apenas para estabelecer um cenário de referência. O valor também pode representar a carga sustentada de entrada.
- (premissa) Considerar leitura independente por consumidor e capacidade de processamento de pelo menos 180.000 mensagens/s para ambos. Sem confirmação, os resultados são limites otimistas.
- (premissa) Considerar backlog inicial nulo e processamento FIFO, sem perdas, para estimar atrasos.
- (premissa) Usar 180.000 mensagens/s como taxa pós-pico somente na análise condicional de drenagem.

## 2. Aritmética do cenário

Cenário de referência: capacidade efetiva de 180.000 mensagens/s.

- Capacidade efetiva = C = min(180.000, capacidade do consumidor) = 180.000 mensagens/s (premissa).
- Duração = D = 25 × 60 = 1.500 s (cálculo).
- Excedente = E = 320.000 − 180.000 = 140.000 mensagens/s (cálculo).
- Backlog = B = 140.000 × 1.500 = 210.000.000 mensagens (cálculo).
- Backlog em bytes = 210.000.000 × tamanho médio da mensagem = indeterminado (cálculo).
- Drenagem = R = 180.000 − taxa pós-pico = indeterminado (cálculo).
- Drenagem se a entrada retornar a 180.000 = R = 180.000 − 180.000 = 0 mensagens/s (cálculo).
- Tempo de drenagem nesse caso = T = 210.000.000 ÷ 0 = infinito; backlog não diminui (cálculo).
- Drenagem se a entrada cair abaixo de 180.000 = R = 180.000 − taxa pós-pico, com R positivo; T = 210.000.000 ÷ R = indeterminado (cálculo).
- Se a entrada superar 180.000 após o pico, R será negativo e o backlog continuará crescendo (cálculo).
- Atraso FIFO estimado = L ≈ 210.000.000 ÷ 180.000 = 1.166,67 s (cálculo).
- Sentinel: L − SLA = 1.166,67 − 60 = 1.106,67 s; razão = 1.166,67 ÷ 60 ≈ 19,44 vezes o limite (cálculo).
- Forge: L − SLA = 1.166,67 − 900 = 266,67 s; razão = 1.166,67 ÷ 900 ≈ 1,30 vez o limite (cálculo).
- Retenção temporal versus atraso = 14.400 − 1.166,67 = 13.233,33 s de margem nominal (cálculo).
- Retenção versus recuperação completa = 14.400 − (1.500 + T) = indeterminado; com R = 0, a recuperação completa não ocorre (cálculo).
- Retenção por tamanho = limite em bytes − backlog em bytes = indeterminado (cálculo).

Interpretação alternativa

Se 180.000 mensagens/s representar apenas a carga sustentada de entrada, C é desconhecida. Nesse caso, E = 320.000 − C; B = (320.000 − C) × 1.500; R = C − 180.000; T = B ÷ R e L ≈ B ÷ C (cálculo). As expressões de backlog aplicam-se quando C é inferior a 320.000. Se C ≥ 320.000, não há acúmulo decorrente do pico informado; se C ≤ 180.000, não há drenagem após o pico.

Capacidade mínima teórica por SLA, sob FIFO e backlog inicial nulo:

- Sentinel: (320.000 − C) × 1.500 ÷ C ≤ 60 implica C ≥ 307.692,31 mensagens/s (cálculo).
- Forge: (320.000 − C) × 1.500 ÷ C ≤ 900 implica C ≥ 200.000 mensagens/s (cálculo).

(julgamento) A retenção temporal parece suficiente para o atraso isolado calculado, mas não garante recuperação ou ausência de perda. Ambos os SLAs são violados no cenário de referência. A recorrência dos picos permanece indeterminada.

## 3. Restrições

- Sentinel: atraso máximo de 60 segundos, SLA contratual; inegociável (fato).
- Forge: atraso máximo de 15 minutos; inegociável (julgamento, por ser SLA).
- Orçamento: infraestrutura já 8% acima do previsto; flexível, embora fortemente limitante (julgamento).
- Durabilidade: nenhuma perda de telemetria aceitável; inegociável (julgamento, pela proibição explícita).

## 4. Candidatos avaliados

### C1 — Limitação de taxa com buffer durável

Descrição: Aplicar cotas por produtor e admission control, com persistência local ou externa, confirmação após gravação durável e reenvio controlado.

Prós: Controla a pressão de entrada, reduz interferência entre produtores e preserva mensagens durante rejeições temporárias (julgamento).

Contras: Desloca filas para os produtores; pode acrescentar latência fim a fim e requer capacidade de armazenamento (julgamento).

Custo operacional e de infraestrutura: Médio; exige armazenamento, retry e tratamento de falhas (julgamento).

Risco de perda de mensagem: Condicionado à durabilidade do buffer, política de confirmação e recuperação (julgamento).

Efeito no SLA do consumidor mais sensível: Limitar a entrada a 180.000 mensagens/s não resolve, isoladamente, a entrega tempestiva das 320.000 mensagens/s geradas no pico (cálculo).

Status: Eliminado como opção isolada para o SLA do Sentinel, na configuração de referência (julgamento).

### C2 — Isolamento e priorização do Sentinel

Descrição: Separar caminhos de entrega, com filas ou partições independentes, cotas e prioridade para o Sentinel, preservando mensagens em armazenamento durável.

Prós: Reduz interferência do Forge e permite proteção específica ao tráfego crítico (julgamento).

Contras: Não aumenta a capacidade agregada; eficácia depende da distribuição real de mensagens (julgamento).

Custo operacional e de infraestrutura: Médio; exige gestão de filas, isolamento e observabilidade por consumidor (julgamento).

Risco de perda de mensagem: Depende das garantias de persistência e limites de cada fila (julgamento).

Efeito no SLA do consumidor mais sensível: Pode reduzir atrasos se a demanda do Sentinel couber na capacidade reservada; sem volume por consumidor, o resultado é indeterminado.

Status: Condicional: medir demanda e comprovar que o backlog reservado ao Sentinel permanece ≤ 60 × capacidade efetiva do seu caminho, com persistência durável (julgamento).

### C3 — Escalonamento horizontal

Descrição: Aumentar a capacidade do Relay e dos consumidores, mantendo persistência durável e confirmação de processamento.

Prós: Trata diretamente o déficit agregado de capacidade e reduz backlog e latência (julgamento).

Contras: Pode exigir mudanças de particionamento, processamento e infraestrutura, com custo adicional (julgamento).

Custo operacional e de infraestrutura: Alto; maior consumo de recursos e complexidade de operação (julgamento).

Risco de perda de mensagem: Não é resolvido apenas pelo escalonamento; depende da persistência e recuperação (julgamento).

Efeito no SLA do consumidor mais sensível: Capacidade efetiva ≥ 307.692,31 mensagens/s satisfaz o limite teórico calculado, nas premissas adotadas (cálculo).

Status: Condicional: validar capacidade efetiva ≥ 307.692,31 mensagens/s, durabilidade e viabilidade orçamentária (julgamento).

### C4 — Isolamento, escalonamento seletivo e controle durável de entrada

Descrição: Isolar Sentinel e Forge, dimensionar cada caminho segundo sua demanda, adicionar cotas por produtor e buffers duráveis para absorver excedentes.

Prós: Combina proteção de latência, aumento seletivo de capacidade, controle da origem do acúmulo e preservação de mensagens (julgamento).

Contras: Mais componentes, gestão de retries e necessidade de validação integrada (julgamento).

Custo operacional e de infraestrutura: Médio a alto; crescimento seletivo pode reduzir despesas frente ao escalonamento indiscriminado (julgamento).

Risco de perda de mensagem: Controlável mediante confirmação durável, armazenamento dimensionado e recuperação testada (julgamento).

Efeito no SLA do consumidor mais sensível: O caminho reservado ao Sentinel pode ser dimensionado pela demanda efetiva, sem competir com Forge (julgamento).

Status: Condicional: comprovar capacidade e backlog por consumidor dentro dos limites calculados, persistência sem perdas e custo aprovado (julgamento).

## 5. Matriz candidatos × restrições

Vereditos relativos ao cenário e às condições de implementação descritas.

| Candidato           | Sentinel [I]                                      | Forge [I]                                         | Orçamento                                        | Durabilidade [I]                                         | Status                 |
| ------------------- | ------------------------------------------------- | ------------------------------------------------- | ------------------------------------------------ | -------------------------------------------------------- | ---------------------- |
| C1 — Cotas duráveis | Não atende: represamento excede 60s               | Indeterminado: depende da vazão                   | Indeterminado: armazenamento sem custo informado | Atende com condição: gravação durável confirmada         | Eliminado isoladamente |
| C2 — Isolamento     | Atende com condição: backlog reservado limitado   | Indeterminado: capacidade Forge desconhecida      | Indeterminado: custo de isolamento desconhecido  | Atende com condição: filas duráveis                      | Condicional            |
| C3 — Escalonamento  | Atende com condição: capacidade mínima comprovada | Atende com condição: capacidade mínima comprovada | Indeterminado: expansão não orçada               | Atende com condição: persistência validada               | Condicional            |
| C4 — Combinação     | Atende com condição: capacidade e backlog medidos | Atende com condição: capacidade e backlog medidos | Indeterminado: custo incremental desconhecido    | Atende com condição: persistência e recuperação testadas | Condicional            |

(julgamento) C4 acrescenta a C2 controle de entrada e capacidade seletiva, e acrescenta a C3 isolamento contra interferência e buffers duráveis. O benefício depende da demanda real por consumidor; o custo adicional permanece sem quantificação.

## 6. Recomendação

Caminho recomendado — C4, condicional, com implementação incremental

1. Medir primeiro: identificar capacidades efetivas de Relay, Sentinel e Forge, participação dos produtores, backlog inicial, latências fim a fim e limites de armazenamento (julgamento).

2. Implantar isolamento lógico: separar o processamento do Sentinel e Forge, estabelecer cotas por produtor e impedir que backlog de ingestão bloqueie alertas (julgamento).

3. Validar persistência ponta a ponta: exigir confirmação após gravação durável, reenvio idempotente, recuperação de falhas e armazenamento dimensionado para os excedentes observados (julgamento).

4. Escalar seletivamente: dimensionar as capacidades efetivas conforme os volumes medidos. Se ambos receberem integralmente o pico, validar ≥ 307.692,31 mensagens/s para Sentinel e ≥ 200.000 mensagens/s para Forge, sob as premissas de FIFO e backlog inicial nulo (cálculo).

5. Executar testes de pico e aprovar custos: reproduzir o padrão de 320.000 mensagens/s por 1.500 segundos, verificar SLAs e ausência de perdas, e aprovar o custo incremental antes de expandir infraestrutura (julgamento).

Justificativa

(julgamento) A combinação é preferível porque diferencia duas causas que ainda não podem ser separadas com os dados disponíveis: saturação agregada e interferência entre consumidores ou produtores.

O isolamento protege o Sentinel; o controle de entrada limita a propagação da sobrecarga; a persistência trata a exigência de durabilidade; o escalonamento seletivo corrige déficits medidos.

(julgamento) A recomendação depende de comprovação técnica e aprovação orçamentária. Não há evidência suficiente para afirmar que sua implantação cabe no orçamento atual.

Alternativas descartadas e motivo

- C1: sozinha não resolve a latência contratual; transfere o atraso para buffers externos (julgamento).
- C2: pode proteger Sentinel, mas não corrige saturação agregada quando a demanda excede a capacidade isolada (julgamento).
- C3: pode resolver os limites de capacidade, mas não oferece isolamento de produtores e pode representar expansão excessiva para o orçamento pressionado (julgamento).

Riscos residuais

- (julgamento) Falhas de armazenamento e confirmação podem provocar perda ou duplicação.
- (julgamento) Produtor dominante pode saturar sua cota ou armazenamento.
- (julgamento) Backlogs iniciais e picos recorrentes podem invalidar os limites teóricos.
- (julgamento) Escalonamento pode não produzir a capacidade esperada por gargalos de particionamento.
- (julgamento) Custos e esforço operacional podem inviabilizar parte da solução.

Gatilhos de revisão

- (julgamento) Capacidade efetiva medida diferente da premissa de 180.000 mensagens/s.
- (julgamento) Recorrência de picos antes da drenagem completa.
- (julgamento) Evidência de que um produtor ou consumidor concentra a saturação.
- (julgamento) Backlog do Sentinel superior a 60 × sua capacidade efetiva.
- (julgamento) Backlog do Forge superior a 900 × sua capacidade efetiva.
- (julgamento) Persistência insuficiente ou custo incremental não aprovado.

## 7. Limites da análise

Não é possível concluir

- (julgamento) Se o incidente decorreu de capacidade agregada, produtor dominante ou consumidor lento.
- (julgamento) Se cada consumidor recebe todo o volume de eventos.
- (julgamento) Se a retenção de quatro horas evita perdas durante recuperação prolongada.
- (julgamento) Se alguma alternativa cabe no orçamento disponível.
- (julgamento) Se a capacidade nominal pode ser alcançada sob carga real.

Dados que reduziriam a incerteza, em ordem de impacto

1. (julgamento) Throughput efetivo por consumidor e modelo de leitura.
2. (julgamento) Volume por consumidor, produtor e classe de evento.
3. (julgamento) Backlog, latência e taxa de entrada após o pico.
4. (julgamento) Recorrência dos picos e comportamento durante recuperação.
5. (julgamento) Limites de armazenamento, tamanho das mensagens e garantias de persistência.
6. (julgamento) Custo incremental admissível e capacidade operacional do time.

## 8. Entradas ignoradas

Nenhuma.
```

Saída da rodada 2:

```markdown
# Análise de estratégia de backpressure

## 1. Fatos e premissas

Fatos

- (fato) Relay: throughput sustentado informado de 180.000 mensagens/s.
- (fato) Pico observado: 320.000 mensagens/s durante 25 minutos = 1.500 segundos.
- (fato) Retenção atual: 4 horas = 240 minutos = 14.400 segundos.
- (fato) Consumidores: Forge (ingestão) e Sentinel (alertas em tempo real).
- (fato) SLAs: Sentinel ≤ 60 segundos; Forge ≤ 900 segundos.
- (fato) Orçamento de infraestrutura do trimestre: 8% acima do previsto.
- (fato) Perda de telemetria é inaceitável.

Dados ausentes

- (fato) Capacidade efetiva de entrega do Relay e de cada consumidor.
- (fato) Taxa de entrada após o pico e recorrência dos picos.
- (fato) Tamanho médio das mensagens e limite de retenção por bytes.
- (fato) Número de produtores, distribuição de volume e concentração por tenant.
- (fato) Modelo de consumo: fila compartilhada ou leitura independente.
- (fato) Mecanismos atuais de persistência, confirmação e reenvio.
- (fato) Margem orçamentária disponível e capacidade operacional.

Premissas adotadas

- (premissa) Interpretar inicialmente 180.000 mensagens/s como capacidade efetiva C, embora o termo "throughput sustentado" possa representar apenas a carga normal.
- (premissa) Usar 180.000 mensagens/s como entrada após o pico, exclusivamente para análise de sensibilidade.
- (premissa) Considerar fila FIFO, backlog inicial nulo e capacidade constante.
- (premissa) Para os cálculos ilustrativos, considerar ambos os consumidores sujeitos à mesma capacidade efetiva; a arquitetura real permanece indeterminada.

## 2. Aritmética do cenário

Cenário ilustrativo: C = 180.000 mensagens/s.

- Capacidade efetiva = C = min(capacidade de saída do Relay, capacidade do consumidor) = 180.000 mensagens/s (premissa). Por consumidor, C_F e C_S permanecem indeterminados.

- Excedente no pico = E = 320.000 − 180.000 = 140.000 mensagens/s (cálculo).

- Duração = D = 25 × 60 = 1.500 segundos (cálculo).

- Backlog = B = E × D = 140.000 × 1.500 = 210.000.000 mensagens (cálculo).

- Backlog em bytes = 210.000.000 × tamanho médio da mensagem = indeterminado (cálculo).

- Drenagem = R = C − entrada após pico = 180.000 − 180.000 = 0 mensagens/s (cálculo, premissa).

- Tempo de drenagem = T = B ÷ R = 210.000.000 ÷ 0 = sem drenagem (cálculo). O backlog permanece acumulado enquanto as taxas forem iguais; cresce se a entrada superar C.

- Atraso máximo estimado = L ≈ B ÷ C = 210.000.000 ÷ 180.000 ≈ 1.166,67 segundos = 19,44 minutos (cálculo, premissa FIFO).

- Sentinel: L − SLA = 1.166,67 − 60 = 1.106,67 segundos; razão = 1.166,67 ÷ 60 ≈ 19,44 vezes o limite (cálculo).

- Forge: L − SLA = 1.166,67 − 900 = 266,67 segundos; razão = 1.166,67 ÷ 900 ≈ 1,30 vez o limite (cálculo).

- Retenção versus L = 14.400 − 1.166,67 = 13.233,33 segundos de diferença (cálculo).

- Retenção versus D + T = 14.400 − (1.500 + T) = indeterminado, pois T não é finito (cálculo).

- Retenção por tamanho = limite em bytes − B em bytes = indeterminado (cálculo).

Sensibilidade à interpretação dos 180.000 mensagens/s

- Se representam apenas entrada sustentada: C = min(saída Relay, consumidor) = indeterminado; E = 320.000 − C; B = (320.000 − C) × 1.500, quando E é positivo (cálculo).
- R = C − 180.000: se C ≤ 180.000, não há drenagem; se C > 180.000, T = B ÷ (C − 180.000) (cálculo, premissa de entrada pós-pico).
- Se C ≥ 320.000, E ≤ 0: o pico informado não gera backlog por insuficiência agregada de throughput (cálculo).
- Capacidade mínima ilustrativa para Sentinel: (320.000 − C) × 1.500 ÷ C ≤ 60 implica C ≥ 307.692,31 mensagens/s (cálculo).
- Capacidade mínima ilustrativa para Forge: (320.000 − C) × 1.500 ÷ C ≤ 900 implica C ≥ 200.000 mensagens/s (cálculo).

(julgamento) A retenção temporal supera L no exemplo, mas não comprova ausência de expiração, pois a drenagem não ocorre. Ambos os SLAs seriam violados sob a premissa de fila compartilhada. Sem capacidades independentes e frequência dos picos, não é possível determinar o comportamento real de cada consumidor nem a recuperação entre incidentes.

## 3. Restrições

- Sentinel: atraso máximo de 60 segundos; inegociável, por SLA contratual (fato).
- Forge: atraso máximo de 900 segundos; inegociável, por SLA (julgamento quanto à classificação).
- Durabilidade: nenhuma perda de telemetria; inegociável (fato).
- Orçamento: infraestrutura já 8% acima do previsto; flexível, mas com margem adicional desconhecida (julgamento quanto à classificação).

## 4. Candidatos avaliados

### C1 — Limitação de taxa com persistência no produtor

Descrição: Aplicar cotas por produtor e controle de admissão, mantendo mensagens excedentes em buffer durável até confirmação de entrega.

Prós: Controla admissão; reduz interferência entre tenants; evita descarte por rejeição transitória.

Contras: Transfere backlog para os produtores; não aumenta capacidade efetiva nem resolve, isoladamente, latência fim a fim.

Custo operacional e de infraestrutura: Médio; exige persistência, reenvio, cotas e monitoramento.

Risco de perda de mensagem: Condicional à durabilidade do buffer, às confirmações e à recuperação.

Efeito no SLA do consumidor mais sensível: Com capacidade de 180.000 mensagens/s, o atraso estimado de 1.166,67 segundos permanece relevante, mesmo fora do Relay.

Status: Eliminado como opção isolada, por violação do SLA do Sentinel no cenário ilustrativo (julgamento).

### C2 — Isolamento e priorização do Sentinel

Descrição: Separar o processamento de alertas e ingestão, com filas ou partições isoladas, cotas e prioridade para o Sentinel.

Prós: Reduz interferência do Forge; permite alocar capacidade conforme criticidade.

Contras: Não resolve insuficiência agregada; desempenho depende da distribuição de eventos e das capacidades por caminho.

Custo operacional e de infraestrutura: Médio; requer configuração, monitoramento e possível reorganização de consumidores.

Risco de perda de mensagem: Indeterminado; depende da persistência e retenção de cada caminho.

Efeito no SLA do consumidor mais sensível: Pode reduzir L_S, desde que B_S ≤ 60 × C_S.

Status: Condicional à medição de B_S ≤ 60 × C_S e B_F ≤ 900 × C_F, com persistência durável verificada (julgamento).

### C3 — Escalonamento horizontal

Descrição: Aumentar a capacidade do Relay e dos consumidores, preservando armazenamento durável e processamento independente quando aplicável.

Prós: Ataca insuficiência de capacidade; reduz acúmulo e acelera drenagem.

Contras: Pode exigir novas instâncias, partições ou alterações operacionais; custo adicional incerto.

Custo operacional e de infraestrutura: Alto; maior consumo de recursos e esforço operacional.

Risco de perda de mensagem: Indeterminado; escalonamento não substitui durabilidade.

Efeito no SLA do consumidor mais sensível: C ≥ 307.692,31 mensagens/s é suficiente no modelo ilustrativo de pico e fila FIFO, sem atrasos adicionais.

Status: Condicional a capacidade efetiva medida ≥ 307.692,31 mensagens/s, persistência verificada e aprovação orçamentária (julgamento).

### C4 — Isolamento + cotas + buffers duráveis + escalonamento seletivo

Descrição: Combinar isolamento do Sentinel, admissão controlada por produtor, retenção durável e expansão somente dos gargalos medidos.

Prós: Trata interferência entre produtores e consumidores, preserva dados e permite dimensionamento direcionado.

Contras: Maior integração entre mecanismos; requer testes de falha e controle de backlog.

Custo operacional e de infraestrutura: Médio a alto; depende da capacidade existente e das mudanças necessárias.

Risco de perda de mensagem: Reduzível com persistência, confirmações e reenvio testados; não demonstrado como nulo.

Efeito no SLA do consumidor mais sensível: Permite limitar B_S a 60 × C_S, sem exigir necessariamente que o Sentinel processe todo o volume do Forge.

Status: Condicional a limites de backlog medidos por consumidor, persistência verificada e custo aprovado (julgamento).

## 5. Matriz candidatos × restrições

| Candidato           | Sentinel [I]                                 | Forge [I]                                         | Sem perda [I]                                            | Orçamento                                      | Status                 |
| ------------------- | -------------------------------------------- | ------------------------------------------------- | -------------------------------------------------------- | ---------------------------------------------- | ---------------------- |
| C1 — Cotas duráveis | Não atende: atraso excede limite ilustrativo | Não atende: atraso excede limite ilustrativo      | Atende com condição: persistência e reenvio verificados  | Indeterminado: custos não informados           | Eliminado isoladamente |
| C2 — Isolamento     | Atende com condição: B_S ≤ 60 × C_S          | Atende com condição: B_F ≤ 900 × C_F              | Atende com condição: persistência testada                | Indeterminado: custo da separação desconhecido | Condicional            |
| C3 — Escalonamento  | Atende com condição: C ≥ 307.692,31          | Atende com condição: capacidade mínima suficiente | Atende com condição: durabilidade testada                | Indeterminado: expansão não orçada             | Condicional            |
| C4 — Combinação     | Atende com condição: B_S ≤ 60 × C_S          | Atende com condição: B_F ≤ 900 × C_F              | Atende com condição: persistência e recuperação testadas | Indeterminado: custo incremental desconhecido  | Condicional            |

(julgamento) A classificação condicional significa que há critérios técnicos verificáveis, não que sua satisfação já esteja demonstrada. Os resultados numéricos dependem das premissas da seção 1. O custo exige validação antes da implantação.

## 6. Recomendação

Caminho recomendado — C4, de forma condicional

1. Medir e identificar o gargalo. Determinar capacidades efetivas do Relay, Sentinel e Forge; medir taxa por produtor, backlog por consumidor e comportamento após picos. Distinguir saturação agregada, produtor dominante e consumidor lento.

2. Isolar o caminho crítico do Sentinel. Configurar processamento independente ou prioridade efetiva, mantendo o Forge em caminho separado. Validar B_S ≤ 60 × C_S e B_F ≤ 900 × C_F durante o pico observado, incluindo atrasos de processamento e entrega.

3. Introduzir cotas e backpressure com persistência. Controlar admissão por produtor, com confirmação, reenvio e armazenamento durável dos excedentes. Dimensionar buffers para o excedente observado e para a recuperação, sem assumir que a retenção atual elimina riscos.

4. Escalonar seletivamente quando as medições exigirem. Aumentar capacidade apenas dos componentes que impedem os limites de backlog. Se o Sentinel consumir todo o fluxo sob as premissas ilustrativas, testar capacidade efetiva mínima de 307.692,31 mensagens/s, acrescida da margem necessária para atrasos não modelados.

5. Validar operacionalmente e financeiramente. Executar testes com pico de 320.000 mensagens/s durante 1.500 segundos, recuperação, falhas e reenvios. Exigir comprovação de durabilidade e aprovação do custo incremental antes da implantação definitiva.

Justificativa

(julgamento) C4 combina proteção do consumidor crítico com controle de admissão e durabilidade, atacando causas diferentes do acúmulo. Em relação a C2, acrescenta cotas e capacidade seletiva; em relação a C1, protege o SLA crítico; em relação a C3, evita assumir que toda insuficiência precisa ser resolvida por expansão agregada.

A recomendação é condicional porque os dados disponíveis não comprovam a capacidade atual nem a adequação orçamentária.

Alternativas descartadas e motivo

- C1 isolado: conserva dados, mas não demonstra cumprimento dos SLAs; no exemplo, ambos são violados.
- C2 isolado: pode ser suficiente se o Sentinel processar fração pequena do fluxo e houver capacidade reservada; permanece opção de menor intervenção se as medições confirmarem isso.
- C3 isolado: tecnicamente viável sob capacidade e durabilidade verificadas, porém seu custo adicional pode ser incompatível com o orçamento já excedido.

Riscos residuais

- Falhas de persistência, reenvio ou confirmação.
- Crescimento de buffers durante picos recorrentes.
- Saturação de armazenamento e expiração de mensagens.
- Excesso de tráfego prioritário comprometendo a capacidade reservada.
- Custo e complexidade acima da margem operacional disponível.

Gatilhos de revisão

- Capacidade efetiva diferente da premissa de 180.000 mensagens/s.
- Backlog que não drena antes do pico seguinte.
- Violação de B_S ≤ 60 × C_S ou B_F ≤ 900 × C_F.
- Evidência de perda, expiração ou duplicação não tratada.
- Distribuição de tráfego indicando predominância de um produtor.
- Custo da combinação superior à margem aprovada.
- Medição demonstrando que isolamento simples já satisfaz todos os requisitos.

## 7. Limites da análise

Não é possível concluir

- (julgamento) Que o throughput sustentado represente capacidade máxima.
- (julgamento) Que Sentinel e Forge compartilhem backlog ou capacidade.
- (julgamento) Que a retenção de quatro horas impeça perdas.
- (julgamento) Que o problema decorra de produtor específico, volume agregado ou consumidor lento.
- (julgamento) Que a arquitetura recomendada caiba no orçamento sem medições e estimativas adicionais.

Dados que reduziriam a incerteza

1. (julgamento) Throughput efetivo por consumidor e modelo de leitura.
2. (julgamento) Backlog, latência fim a fim e taxa de entrada após o pico.
3. (julgamento) Distribuição de volume por produtor e subconjunto consumido pelo Sentinel.
4. (julgamento) Persistência, semântica de confirmação, retenção e tamanho médio de mensagem.
5. (julgamento) Frequência dos picos, capacidade de recuperação e custos incrementais.
6. (julgamento) Margem financeira e operacional efetivamente disponível.

## 8. Entradas ignoradas

Nenhuma.
```

## Curadoria

**Framework:** Tree of Thoughts (árvore de pensamentos), sem few-shot. A decisão exige explorar caminhos alternativos e podar os inviáveis, e exemplos de decisão completa ancorariam o modelo em um caminho e prejudicariam o reuso. Os blocos do prompt mapeiam assim: na etapa 4 do `<metodo>` o modelo gera os ramos (mecanismos individuais e combinações); na etapa 5 avalia cada ramo contra todas as restrições e poda os que violam restrição inegociável; na etapa 6 recomenda o ramo remanescente e registra riscos residuais e gatilhos de revisão. As etapas 1 a 3 fixam fatos, contas e restrições que servem de critério de avaliação, `<regras>` cobre rastreabilidade, proporcionalidade, segurança da entrada e entrada insuficiente, e `<formato_saida>` fixa as oito seções e a matriz de candidatos contra restrições.

**Refinos:** o prompt foi criado por meta-prompting com o Opus 5.5 e refinado com revisão humana. A etapa 1 passou a tratar valores ambíguos, como um throughput "sustentado" que pode ser capacidade ou carga, e a mostrar o resultado sob as duas leituras. O limite de palavras subiu de 2.200 para 2.600, para dar margem aos campos exigidos por candidato. Para evitar que as fórmulas saíssem em LaTeX, que não renderiza em Markdown simples, a regra passou a pedir texto simples. A etapa 5 passou a exigir que "não atende" valha só quando a violação decorre da construção do candidato ou das contas, e que a condição de um candidato condicional nomeie um valor ou mecanismo além do próprio requisito. A etapa 3 passou a listar só as restrições declaradas no cenário, porque o modelo acrescentava restrições inexistentes. Execuções intermediárias apresentaram candidatos montados por remoção de uma salvaguarda, só para serem eliminados, e por isso a etapa 4 passou a exigir cada mecanismo na melhor forma defensável e a comparar a combinação com seus componentes.

**Modelo:** GPT 6 Sol com esforço de raciocínio médio. Nas duas rodadas seguiu as oito seções, fez as contas corretamente, registrou a ambiguidade dos 180.000 msgs/s, comparou quatro candidatos contra todas as restrições e chegou à mesma recomendação condicional. Os vereditos e a etiquetagem variaram entre as rodadas.
