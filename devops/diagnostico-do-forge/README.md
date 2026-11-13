---
nome: Diagnóstico do Forge
descricao: Produz o diagnóstico estruturado do estado atual de um pipeline de dados em lote antes da migração para arquitetura orientada a eventos.
versao: 1.0.0
tags:
  - diagnostico
  - pipeline-de-dados
  - arquitetura-orientada-a-eventos
  - migracao
inputs:
  - nome: cenario
    descricao: Descrição do estado atual do pipeline em lote (componentes, agendamentos, volumes, dependentes, falhas conhecidas) e do que a migração precisa garantir.
---

# Diagnóstico do Forge

## Objetivo

Produzir o diagnóstico estruturado do estado atual de um pipeline de dados em lote antes da migração para uma arquitetura orientada a eventos. É o primeiro elo de uma cadeia de três prompts: este diagnóstico alimenta [`passos-da-migracao-do-forge`](../passos-da-migracao-do-forge/), que por sua vez alimenta [`plano-de-migracao-do-forge`](../plano-de-migracao-do-forge/).

A partir do cenário (componentes, agendamentos, volumes, dependentes, falhas conhecidas e o que a migração precisa garantir), o prompt produz sete seções fixas: fatos do cenário, componentes e fluxo de dados, dependentes e forma de consumo, pontos frágeis, riscos da migração, invariantes e lacunas e premissas. Cada item recebe um identificador estável (F, C, D, P, R, I, L, S) e um rótulo de (fato), (premissa) ou (julgamento). A saída é autocontida: quem planeja a migração não precisa ter acesso ao cenário original.

O prompt não propõe etapas, tecnologias nem cronograma. Ele separa o que o cenário informa do que não informa: o que falta vira lacuna, e o que foi suposto para prosseguir vira premissa, com a condição que a invalidaria.

## Quando usar

- Antes de planejar a migração de um pipeline em lote para processamento orientado a eventos, quando o estado atual está descrito de forma parcial e é preciso explicitar o que se sabe e o que falta saber.
- Para registrar, em uma forma que possa ser contestada, os dependentes do pipeline, os pontos frágeis e as propriedades que a migração não pode violar.
- Como primeira etapa da cadeia de migração, para que o elo seguinte receba fatos com números e nomes literais.

## Exemplo de uso

1. Reunir o cenário em texto: ingestão, transformação, destino, ponto frágil conhecido, quem depende do pipeline e o que a migração precisa garantir.
2. Sanear o texto antes do envio a um modelo externo (ver Limitações conhecidas).
3. Copiar o conteúdo de `prompt.md` (sem o frontmatter) e substituir `{{cenario}}` pelo texto do cenário. É o único parâmetro e é obrigatório.
4. Executar em uma conversa ou chamada nova.
5. Ler primeiro as seções 1 e 7 (fatos e lacunas), porque os riscos e os invariantes dependem delas. A saída completa é a entrada do elo 2.

Um exemplo de execução com saída real está na seção [Execução](#execução).

## Limitações conhecidas

- O cenário do exemplo é fictício. Para dados reais, saneie o texto antes do envio: substitua nomes de clientes, de serviços internos e de ambientes por pseudônimos consistentes, e remova segredos e dados pessoais. Mantenha números, unidades, horários e prazos, que o diagnóstico preserva literalmente.
- Na execução registrada, a lista de invariantes ficou com um único item (`I1`), que reúne três verificações diferentes: leitura das tabelas pelo Sentinel, indexação pelo Cerebro e execução dos relatórios de billing. A regra do passo 7 (não registrar o estado-alvo como invariante, preferir a lacuna a um invariante não verificável e unir invariantes verificados da mesma forma) eliminou itens inadequados, mas também itens úteis, como a disponibilidade dos dados de billing e a ausência de perdas ou duplicações. O refino proposto é pedir um invariante por dependente quando o mecanismo de verificação diferir.
- O rótulo é único por linha. Linhas de componentes e dependentes misturam fato e julgamento sob um só rótulo, e os dependentes saíram todos como (premissa) mesmo quando parte da informação consta no cenário (por exemplo, que o Sentinel lê tabelas agregadas).
- A seção 7 ficou com 524 palavras, contra o limite de 500. O limite total (2.000 palavras) foi respeitado, com cerca de 1.585 palavras, contadas por tokens com letra ou número.
- O prompt não calcula nada: não estima, por exemplo, que um lote de cerca de 40 minutos num ciclo de 60 minutos deixa pouca folga quando o volume dobra. Isso seria uma hipótese, e o prompt proíbe inventar dados. O efeito é um diagnóstico prudente, com menos aritmética do que o do prompt de backpressure.
- Os pontos frágeis e os riscos têm sobreposição (por exemplo, a coexistência inconsistente aparece como ponto frágil e como risco de divergência entre caminhos), e os riscos apontam quase todos para os mesmos dependentes e para o mesmo invariante, o que reduz a utilidade da matriz de riscos por dependente.
- O escopo é pipeline em lote com dependentes conhecidos e migração para eventos. Para outro tipo de mudança, a lista de dimensões de risco do passo 6 precisa ser adaptada.
- O resultado vem de duas execuções com um único modelo. A primeira usou uma versão anterior do prompt e serviu para ajustá-lo; a saída registrada abaixo é a da versão final.
- Os testes promptfoo (seção "Testes determinísticos") verificam estrutura, preservação de números e nomes, rótulos e limites. Não avaliam se os pontos frágeis, riscos e invariantes estão corretos. O Sonnet 5.5 às vezes junta várias afirmações numa linha e rotula só parte delas; o teste tolera até 5% de itens sem rótulo final.

## Execução

| Item | Valor |
|---|---|
| Modelo | GPT 6 Sol |
| Identificador na API | `gpt-6-sol` |
| Esforço de raciocínio | médio |

O único parâmetro, `cenario`, foi preenchido com o texto abaixo. A execução foi feita em um chat novo e temporário, sem acesso à memória.

### Entrada: migração do Forge

Valores dos parâmetros:

```text
cenario:
Forge hoje:
- ingestão: um job em cron acorda a cada 60min (o "forge-batch-ingest")
- transformação: 14 etapas de processamento encadeadas (em Spark), ~40min no total
- destino: grava em tabelas no data warehouse, particionadas por hora
- ponto frágil: se um lote falha, o próximo acumula o dobro de volume
- quem depende do Forge: Sentinel (lê as tabelas agregadas), Cerebro (indexa
  os eventos transformados) e os relatórios de billing da Pepper (rodam de madrugada)

O que a migração precisa garantir:
- consumir do Relay continuamente, processando em pequenos blocos no lugar do lote de 1h
- manter quem depende do Forge funcionando durante a transição
- nada de virada única (big-bang): a migração tem que ir em passos e poder voltar atrás
```

### Saída

A saída foi copiada sem edição, exceto a remoção das barras de cópia do ChatGPT (`\~`, `\-`).

```markdown
# DIAGNÓSTICO DO PIPELINE

## 1. Fatos do cenário

- F1: O Forge utiliza um job em cron chamado "forge-batch-ingest", que acorda a cada 60min para ingestão. (fato)
- F2: A transformação possui 14 etapas de processamento encadeadas, executadas em Spark, com duração total de ~40min. (fato)
- F3: O destino do processamento são tabelas no data warehouse, particionadas por hora. (fato)
- F4: Se um lote falha, o próximo acumula o dobro de volume. (fato)
- F5: Sentinel depende do Forge e lê as tabelas agregadas. (fato)
- F6: Cerebro depende do Forge e indexa os eventos transformados. (fato)
- F7: Os relatórios de billing da Pepper dependem do Forge e rodam de madrugada. (fato)
- F8: A migração precisa consumir do Relay continuamente, processando em pequenos blocos no lugar do lote de 1h. (fato)
- F9: A migração precisa manter quem depende do Forge funcionando durante a transição. (fato)
- F10: A migração não pode realizar uma virada única (big-bang); precisa ocorrer em passos e permitir voltar atrás. (fato)

## 2. Componentes e fluxo de dados

- C1: "forge-batch-ingest" — função: ingestão do Forge — gatilho: cron a cada 60min — entrada: não informado → saída: dados para processamento — granularidade: lote de 1h — base: F1, F8. (julgamento)
- C2: Spark — função: transformação em 14 etapas de processamento encadeadas — gatilho: dependência do fluxo de ingestão (premissa) — entrada: dados ingeridos → saída: eventos transformados e dados destinados às tabelas — granularidade: lote; ~40min no total — base: F2, F3, F6. (julgamento)
- C3: Data warehouse — função: armazenamento do resultado do Forge — gatilho: gravação após processamento (premissa) — entrada: dados processados → saída: tabelas particionadas por hora — granularidade: partição por hora — base: F3. (julgamento)
- C4: Relay — função: origem prevista para consumo contínuo — gatilho: não informado — entrada: não informado → saída: dados consumidos em pequenos blocos — granularidade: pequenos blocos, tamanho não informado — base: F8. (julgamento)
- C5: Forge — função: pipeline de ingestão, transformação e disponibilização de dados — gatilho: cron de C1 — entrada: origem atual não informada → saída: tabelas e eventos transformados utilizados pelos dependentes — granularidade: lote de 1h no estado atual — base: F1, F2, F3, F5, F6, F7, F8. (julgamento)
- Fluxo: C1 → C2 → C3 — ingestão antecede transformação e disponibilização nas tabelas; as 14 etapas de C2 são encadeadas, mas sua ordem individual não foi informada. (julgamento)
- Fluxo: C4 → processamento em pequenos blocos → disponibilização aos dependentes — fluxo requerido, sem componentes intermediários ou contratos de entrega especificados. (julgamento)

## 3. Dependentes e forma de consumo

- D1: Sentinel — consome tabelas agregadas de C3 — mecanismo: leitura das tabelas — quando: frequência e horário não informados — garantia assumida: continuidade de acesso e consistência dos agregados; completude e disponibilidade temporal específicas não informadas — base: F5, F9. (premissa)
- D2: Cerebro — consome eventos transformados de C5 — mecanismo: indexação; interface de entrega não informada — quando: frequência e horário não informados — garantia assumida: continuidade de recebimento, integridade e ausência de duplicações indevidas dos eventos; ordenação não informada — base: F6, F9. (premissa)
- D3: Relatórios de billing da Pepper — consomem resultados do Forge C5; tabelas específicas não informadas — mecanismo: execução de relatórios — quando: de madrugada, sem horário exato informado — garantia assumida: dados necessários disponíveis e completos no momento da execução; critérios de fechamento não informados — base: F7, F9. (premissa)

## 4. Pontos frágeis

- P1: C1/C2 — falha de um lote → volume não processado acumula no próximo → próximo lote recebe o dobro de volume, podendo ampliar o tempo de processamento e atrasar entregas — dependentes afetados: D1, D2, D3 — base: F1, F2, F4, F5, F6, F7. (julgamento)
- P2: C1/C2 — execução em intervalos de 60min combinada com transformação de ~40min → disponibilização condicionada à conclusão do processamento → possível defasagem dos dados consumidos por D1, D2 e D3 — dependentes afetados: D1, D2, D3 — base: F1, F2, F3. (julgamento)
- P3: C2 — falha em uma das 14 etapas encadeadas → possível interrupção das etapas subsequentes → risco de atraso ou indisponibilidade dos resultados do lote — dependentes afetados: D1, D2, D3 — base: F2, F5, F6, F7. (julgamento)
- P4: C3/C5 — disponibilização inconsistente durante a coexistência de caminhos de processamento → possíveis diferenças entre resultados consumidos → risco de divergência em agregações, indexação e billing — dependentes afetados: D1, D2, D3 — base: F5, F6, F7, F9, F10. (julgamento)

## 5. Riscos da migração

- R1: Completude — substituição do lote de 1h por pequenos blocos → agregados ou relatórios podem consumir dados antes da conclusão lógica do período — atinge: D1, D3, I1. (julgamento)
- R2: Ordenação — processamento incremental pode alterar a ordem de disponibilização dos eventos → possível impacto na indexação e nos agregados dependentes de ordem — atinge: D1, D2, I1. (julgamento)
- R3: Duplicidade e idempotência — coexistência ou reprocessamento pode entregar o mesmo evento mais de uma vez → risco de duplicação de índices, agregações e valores de billing — atinge: D1, D2, D3, I1. (julgamento)
- R4: Dados atrasados e reprocessamento — eventos recebidos após o processamento de determinado período podem modificar resultados já disponibilizados → risco de inconsistência temporal — atinge: D1, D2, D3, I1. (julgamento)
- R5: Divergência entre caminhos — caminhos antigo e novo podem produzir resultados diferentes durante a coexistência → risco de consumidores observarem versões incompatíveis dos mesmos dados — atinge: D1, D2, D3, I1. (julgamento)
- R6: Picos e contrapressão — acúmulo de dados ou aumento de volume pode superar a capacidade de processamento → atrasos propagados aos consumidores — atinge: D1, D2, D3, I1; evidência atual: F4. (julgamento)
- R7: Reconciliação — diferença de granularidade entre lote de 1h e pequenos blocos dificulta estabelecer equivalência entre resultados → risco de discrepâncias não identificadas — atinge: D1, D2, D3, I1. (julgamento)
- R8: Reversibilidade — rollback com dados parcialmente processados em caminhos distintos pode produzir lacunas, duplicações ou divergências → risco de comprometer a continuidade dos consumidores — atinge: D1, D2, D3, I1. (julgamento)

## 6. Invariantes

- I1: Sentinel, Cerebro e relatórios de billing da Pepper devem permanecer funcionais durante a transição, a coexistência e após eventual rollback — origem: F9, F10 — como verificar: confirmar, em cada condição, a continuidade da leitura das tabelas agregadas por Sentinel, da indexação dos eventos transformados por Cerebro e da execução dos relatórios de billing da Pepper de madrugada; critérios quantitativos de aceitação não informados. (julgamento)

## 7. Lacunas e premissas

### Lacunas

- L1: Origem atual do "forge-batch-ingest", protocolo de ingestão, identificação e limites dos lotes não informados — impede caracterizar integralmente a entrada e a recuperação de dados — itens afetados: C1, C5, R4, R7. (julgamento)
- L2: Gatilhos individuais, entradas, saídas, dependências internas e comportamento de falha das 14 etapas Spark não informados — impede determinar a propagação exata de falhas — itens afetados: C2, R6. (julgamento)
- L3: Nomes, schemas, chaves, regras de agregação e critérios de publicação das tabelas não informados — impede definir equivalência, completude e compatibilidade verificáveis — itens afetados: C3, D1, R1, R3, R5, R7. (julgamento)
- L4: Interface, formato, identificador, ordenação, retenção, possibilidade de replay e comportamento de entrega do Relay não informados — impede avaliar integralmente consumo incremental e recuperação — itens afetados: C4, R2, R3, R4. (julgamento)
- L5: Frequência, horários, consultas e exigências de completude de Sentinel não informados — impede estabelecer critérios temporais e quantitativos de continuidade — itens afetados: D1, R1, I1. (julgamento)
- L6: Origem física, mecanismo de entrega, frequência, regras de indexação, unicidade e ordenação de Cerebro não informados — impede determinar garantias efetivamente necessárias — itens afetados: D2, R2, R3, I1. (julgamento)
- L7: Horário exato de madrugada, origem consultada, período de apuração, fechamento e tolerância a correções dos relatórios de billing da Pepper não informados — impede verificar completude financeira e disponibilidade exigida — itens afetados: D3, R1, R4, I1. (julgamento)
- L8: Volumes absolutos, distribuição temporal, capacidade, picos, atrasos admissíveis e comportamento após falhas sucessivas não informados — impede quantificar acúmulo e contrapressão — itens afetados: C1, C2, R6. (julgamento)
- L9: Estratégia atual de retry, checkpoints, reprocessamento, deduplicação e tratamento de lotes parcialmente concluídos não informada — impede avaliar recuperação e idempotência — itens afetados: C2, R3, R4, R8. (julgamento)
- L10: Critérios de reconciliação, tolerância a divergências e comportamento de leitura durante coexistência e rollback não informados — impede definir invariantes adicionais de equivalência e recuperação verificáveis — itens afetados: R5, R7, R8, I1. (julgamento)
- L11: Contratos de evolução de schema, compatibilidade entre versões e consumidores não informados — impede avaliar o risco de alterações estruturais sobre D1, D2 e D3 — itens afetados: C3, D1, D2, D3. (julgamento)
- L12: Custos atuais, recursos disponíveis, monitoramento, responsabilidades operacionais e limites de carga não informados — impede avaliar quantitativamente custo e carga operacional da coexistência — itens afetados: C1, C2, C3, C4, R6. (julgamento)

### Premissas assumidas

- S1: A ingestão de C1 alimenta C2, que precede a gravação em C3 — invalidada se houver processamento independente, paralelo ou outro caminho de publicação. (premissa)
- S2: Sentinel depende de consistência dos agregados disponíveis — invalidada se aceitar resultados parciais ou inconsistências temporárias. (premissa)
- S3: Cerebro precisa preservar a integridade dos eventos indexados e evitar duplicações indevidas — invalidada se sua indexação admitir ou corrigir automaticamente duplicações. (premissa)
- S4: Os relatórios de billing da Pepper dependem de dados completos no momento da execução de madrugada — invalidada se aceitarem fechamento parcial ou correções posteriores. (premissa)
- S5: A continuidade exigida em F9 inclui a preservação dos contratos de consumo durante coexistência e rollback — invalidada se forem permitidas alterações ou interrupções previamente aceitas pelos consumidores. (premissa)
```

### Testes determinísticos (promptfoo)

O cenário do Forge acima roda como caso de teste em [`promptfooconfig.yaml`](promptfooconfig.yaml), contra dois provedores, junto com um caso de entrada insuficiente. Comando, a partir desta pasta:

```powershell
promptfoo eval -c promptfooconfig.yaml --no-cache --output resultado.json
```

| Item | Valor |
|---|---|
| Versão do promptfoo | 0.125.1 |
| Provedor 1 | OpenAI `gpt-6-sol`, `reasoning_effort: medium` |
| Provedor 2 | Anthropic `claude-sonnet-5-5`, `thinking.type: between_tools`, `max_tokens: 16000` |
| Prompt testado | `prompt.md` v1.0.0 (arquivo inteiro, com frontmatter) |
| Entrada | [`dados/cenario-forge.txt`](dados/cenario-forge.txt), o cenário da seção anterior |

Asserts:

| Caso | Tipo | Critério |
|---|---|---|
| Cenário | `javascript` | título `# DIAGNÓSTICO DO PIPELINE` e as sete seções, na ordem |
| Cenário | `contains-all` | `60min`, `14 etapas`, `40min`, `forge-batch-ingest`, `Sentinel`, `Cerebro`, `Pepper` |
| Cenário | `javascript` | ao menos 95% dos itens identificados (F1, C1, D1...) terminam com rótulo `(fato)`, `(premissa)` ou `(julgamento)`; rótulo composto como `(fato para X; julgamento para Y)` é aceito |
| Cenário | `not-icontains-any` | `kafka`, `flink`, `kinesis`, `pulsar`, `structured streaming` (o prompt proíbe propor solução) |
| Cenário | `javascript` | não afirma ausência no sistema (`não documentado`, `inexistente`, `sem regras`); `sem regras informadas` é aceito |
| Cenário | `javascript` | até **2.400** palavras (limite do prompt, 2.000, mais 20%) |
| Cenário | `latency` e `cost` | até **60000 ms** e **US$ 0,08** (limites próprios; enunciado: 5000 ms e US$ 0,01) |
| Insuficiente | `javascript` | saída exatamente igual à frase fixa `ENTRADA INSUFICIENTE: ...` |
| Insuficiente | `latency` e `cost` | até 15000 ms e US$ 0,02 |

Antes da primeira execução, os asserts foram validados sem custo contra a saída real registrada acima (passam) e contra saídas truncadas (falham).

Resultado da última execução (`Results: 4 passed (100%), 0 failed, 0 errors`):

| Caso | Modelo | Resultado | Latência | Custo (US$) | Palavras |
|---|---|---|---|---|---|
| Cenário | gpt-6-sol | PASS | 17664 ms | 0,0216 | 850 |
| Cenário | claude-sonnet-5-5 | PASS | 24524 ms | 0,0450 | 1365 |
| Insuficiente | gpt-6-sol | PASS | 1236 ms | 0,0007 | 20 |
| Insuficiente | claude-sonnet-5-5 | PASS | 1789 ms | 0,0065 | 20 |

Itens com rótulo final: 100% de 30 no GPT e 97,9% de 48 no Sonnet (um item sem rótulo final).

Histórico das execuções até chegar a esse resultado:

| Execução | Configuração | Resultado | Falhas |
|---|---|---|---|
| 1 | até 2.000 palavras, rótulo final estrito, limites de 120 s e US$ 0,15 | 3/4 | Sonnet: o item P1 terminou com rótulo composto, `(fato para o dobro de volume; julgamento para a extrapolação do tempo)` |
| 2 | rótulo composto aceito, até 2.400 palavras, limites de 60 s e US$ 0,08 | 2/4 | GPT: o assert de ausência reprovou `sem regras informadas`, forma correta de dizer "não informado"; Sonnet: a linha `Fluxo` terminou sem rótulo final |
| 3 | assert de ausência com exceção para `sem regras informadas`; taxa mínima de 95% de itens com rótulo | 4/4 | nenhuma |

## Curadoria

**Técnica:** encadeamento de prompts (prompt chaining), sem few-shot. A migração é grande demais para um prompt só, e uma resposta única sairia rasa. O prompt de diagnóstico resolve uma etapa do raciocínio: separar o que o cenário informa do que ele não informa e entregar um artefato estruturado que vira a entrada do elo seguinte. Os blocos do prompt mapeiam assim: o passo 2 do `<metodo>` extrai os fatos literais; os passos 3 a 5 mapeiam componentes, dependentes e pontos frágeis; os passos 6 e 7 identificam riscos e invariantes; o passo 8 separa lacunas de premissas; `<regras>` fixa rótulos, identificadores e limites; `<formato_de_saida>` fixa as sete seções que o elo 2 espera ler. Exemplos de diagnóstico completo ancorariam o modelo em um pipeline específico e prejudicariam o reuso.

**Cadeia:** este é o elo 1 de 3. A saída é a entrada de [`passos-da-migracao-do-forge`](../passos-da-migracao-do-forge/), que exige as seções 1, 3 e 6 e herda os identificadores F, C, D, P, I, R, L e S. O elo 3, [`plano-de-migracao-do-forge`](../plano-de-migracao-do-forge/), recebe a saída do elo 2. Como cada elo só enxerga a saída do anterior, a seção 1 preserva números, unidades e nomes do cenário.

**Refinos:** o prompt foi criado por meta-prompting com o Opus 5.5 e refinado com revisão humana. O `nome` e as tags passaram ao padrão do repositório, a frase de insuficiência passou para caixa alta e o exemplo de fórmula, que não tinha uso, foi removido. A primeira execução mostrou três problemas, corrigidos antes da execução registrada: o estado-alvo da migração (consumir do Relay) saiu como invariante, embora deixe de valer após um rollback; invariantes não verificáveis ("conforme critérios ainda não informados") e duplicados; e afirmações como "não documentado" para o que o cenário apenas não informa. O passo 7 passou a tratar o estado-alvo como fato, a preferir a lacuna a um invariante não verificável e a unir invariantes verificados da mesma forma, e uma regra nova pediu "não informado" no lugar de "não documentado" ou "inexistente". A seção 7 tinha limite de 150 palavras e saiu com 460; o limite passou a ser de 500 e o total, de 1.800 para 2.000 palavras, com redistribuição por seção.

**Modelo:** GPT 6 Sol com esforço de raciocínio médio. Na execução final seguiu as sete seções, preservou os números e os nomes do cenário, não assumiu a origem do lote (registrou como lacuna), não propôs solução e usou "não informado" nas lacunas. O ponto fraco foi o invariante único.

**Curadoria dos testes (CP10):**

- **Entradas:** o cenário do Forge, igual ao da seção Execução, e um texto sem relação com pipeline (uma receita de bolo) para o caso de entrada insuficiente. O segundo é curto e barato (cerca de 1 a 2 s e menos de US$ 0,01 por modelo).
- **O que passou desde a primeira execução:** os casos de entrada insuficiente nos dois modelos, a preservação dos números e nomes do cenário, as sete seções na ordem e a ausência de solução proposta. O prompt não precisou de alteração: a versão testada é a v1.0.0.
- **O que falhou e o que mudou nos testes:** o Sonnet 5.5 junta, às vezes, várias afirmações numa linha e rotula só parte delas. Na execução 1 foi um item com rótulo composto (`(fato para ...; julgamento para ...)`); na execução 2, a linha `Fluxo` terminou sem rótulo. São desvios de linhas diferentes, 1 em cerca de 47 itens, e o assert estrito reprovaria o Sonnet de forma intermitente. O assert passou a aceitar rótulo composto e a exigir ao menos 95% dos itens com rótulo final, o que tolera um desvio isolado e reprova a perda de rótulos em massa (confirmado contra uma saída degradada, em 50%). O prompt ficou como estava. O outro ajuste foi um falso positivo meu: o assert de ausência reprovou o GPT por `sem regras informadas`, que é a forma correta de dizer "não informado"; a regra passou a aceitar essa expressão e continua reprovando `sem regras` e `inexistente` afirmados.
- **Limites de latência e custo:** os limites iniciais (120 s e US$ 0,15) ficaram de 2,5 a 4 vezes acima do observado e foram apertados para cerca de 2 vezes o maior valor medido (60 s e US$ 0,08). Cada valor vem de uma ou poucas execuções, e o custo do Sonnet varia com o raciocínio adaptativo.
- **Limite de palavras:** o limite do prompt (2.000) é orientação ao modelo; o teste aceita 20% a mais. Nenhuma execução chegou perto de 2.000 (850 a 1545 palavras).
- **O que os testes não cobrem:** a correção do diagnóstico (se os pontos frágeis e os riscos estão certos) não é verificada; isso exigiria um juiz, que não foi configurado para este prompt.
