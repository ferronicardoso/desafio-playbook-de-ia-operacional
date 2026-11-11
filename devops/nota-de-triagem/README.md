---
nome: Nota de triagem
descricao: Converte um alerta cru de observabilidade em uma nota de triagem padronizada de cinco linhas para a passagem de turno SRE.
versao: 1.1.0
tags:
  - sre
  - triagem
  - alertas
  - plantao
inputs:
  - nome: alerta_cru
    descricao: Texto do alerta exatamente como recebido, em inglês, com timestamp UTC, componente entre colchetes, métricas e contexto em uma única mensagem.
---

# Nota de triagem

## Objetivo

Transformar um alerta cru de observabilidade em uma nota de triagem no padrão único da equipe, com cinco linhas fixas: `ALERTA`, `IMPACTO`, `HIPÓTESE INICIAL`, `AÇÃO IMEDIATA` e `ESCALAR PARA`. A nota é escrita para quem assume o turno seguinte entender a situação sem reler o alerta original.

O prompt usa somente o que está no alerta. Quando o alerta não informa um dado (efeito, escopo, causa ou destinatário de escalonamento), o campo recebe um texto padrão explícito em vez de um valor inventado. O prompt não consulta métricas, não acessa o ambiente e não executa ações.

## Quando usar

- Abertura da nota de triagem logo após o disparo de um alerta do Sentinel, do Relay, do Forge ou do Cerebro.
- Passagem de turno, para que as notas de todos os plantonistas tenham o mesmo formato.
- Padronização de notas antigas escritas em formatos diferentes, a partir do alerta original.

## Exemplo de uso

1. Copiar o texto do alerta exatamente como chegou (timestamp, componente entre colchetes, métricas e contexto).
2. Copiar o conteúdo de `prompt.md` (sem o frontmatter) e substituir `{{alerta_cru}}` pelo texto do alerta. É o único parâmetro e é obrigatório.
3. Executar o prompt em uma conversa ou chamada nova, revisar os campos que vierem com texto padrão (por exemplo, o handle padrão "@time-<componente>") e colar a nota na passagem de turno.

Exemplos de execução com saída real estão na seção [Execução](#execução).

## Limitações conhecidas

- A nota depende exclusivamente do alerta. Sem metadado de roteamento, o `ESCALAR PARA` usa o handle padrão `@time-<componente>` (v1.1.0), que é uma convenção de preenchimento e não um destino confirmado, e o plantonista precisa confirmar o roteamento real. As saídas das três entradas abaixo foram geradas com o prompt v1.0.0, que escrevia "time responsável pelo [Componente] (handle não informado no alerta)".
- O prazo de escalonamento de 15min é o padrão do prompt. Ele só é substituído quando o alerta traz um prazo próprio.
- Quando o alerta nomeia um tenant que é a origem da carga, o `IMPACTO` pode citá-lo como afetado (Entrada 1), e o efeito declarado pelo alerta pode ficar fora da linha.
- A `AÇÃO IMEDIATA` pode se limitar a diagnóstico somente leitura quando o alerta não fundamenta uma mitigação reversível (Entradas 1 e 3).
- O prompt instrui o modelo a não seguir instruções presentes no alerta, mas isso não substitui a remoção prévia de segredos e dados sensíveis antes de enviar um alerta real a um modelo externo.
- Os resultados abaixo vêm de uma execução por entrada com um único modelo. Não foi medida a variação entre execuções.

## Execução

| Item | Valor |
|---|---|
| Modelo | GPT 6 Sol |
| Identificador na API | `gpt-6-sol` |
| Esforço de raciocínio | médio |

O único parâmetro, `alerta_cru`, foi preenchido em todas as execuções. Cada entrada foi executada em uma conversa nova.

### Entrada 1: autoscaler no limite com pico de volume de um tenant

Valores dos parâmetros:

```text
alerta_cru:
2026-05-12 14:02:09 UTC [Sentinel] autoscaler hit max replicas (60/60) on sentinel-api,
queue depth on Relay growing 2k/min, CPU avg 88%, tenant stark-industries
sending 4x baseline volume after onboarding new region
```

Saída:

```markdown
ALERTA: Sentinel - autoscaler atingiu o máximo de réplicas (60/60) no sentinel-api, queue depth do Relay crescendo 2k/min e CPU média em 88%
IMPACTO: tenant stark-industries com volume 4x acima do baseline; demais tenants e serviços não informados no alerta
HIPÓTESE INICIAL: possível saturação do sentinel-api devido ao volume 4x acima do baseline enviado pelo tenant stark-industries após onboarding de nova região
AÇÃO IMEDIATA: verificar em modo somente leitura as métricas de réplicas e CPU do Sentinel e a queue depth do Relay
ESCALAR PARA: time responsável pelo Sentinel (handle não informado no alerta) se o autoscaler não estabilizar em 15min
```

### Entrada 2: rejeição de ingestão após deploy

Valores dos parâmetros:

```text
alerta_cru:
2026-05-13 03:11:00 UTC [Relay] ingest reject rate 6% for 8min, tenant wakanda-systems,
buffer saturated after deploy 02:55
```

Saída:

```markdown
ALERTA: Relay - taxa de rejeição de ingestão em 6% por 8min
IMPACTO: tenant wakanda-systems afetado pela rejeição de ingestão; demais tenants e serviços não informados no alerta
HIPÓTESE INICIAL: possível saturação do buffer causada pelo deploy das 02:55 UTC
AÇÃO IMEDIATA: reverter o deploy das 02:55 UTC para aliviar a saturação do buffer
ESCALAR PARA: time responsável pelo Relay (handle não informado no alerta) se ingest reject rate não estabilizar em 15min
```

### Entrada 3: lag de consumer após falha de job

Valores dos parâmetros:

```text
alerta_cru:
2026-05-13 11:40:22 UTC [Forge] consumer lag 9min and climbing, batch forge-batch-ingest
delayed after previous job failure, downstream Sentinel starting to lag
```

Saída:

```markdown
ALERTA: Forge - consumer lag em 9min e aumentando no batch forge-batch-ingest
IMPACTO: processamento do forge-batch-ingest atrasado e downstream Sentinel começando a apresentar lag; escopo não informado no alerta
HIPÓTESE INICIAL: possível acúmulo de lag no consumer devido à falha do job anterior, que atrasou o batch forge-batch-ingest
AÇÃO IMEDIATA: verificar em modo somente leitura o consumer lag do Forge e o estado do job anterior do forge-batch-ingest
ESCALAR PARA: time responsável pelo Forge (handle não informado no alerta) se consumer lag não estabilizar em 15min
```

### Testes determinísticos (promptfoo)

As três entradas acima (alertas do CP02) rodam como casos de teste em [`promptfooconfig.yaml`](promptfooconfig.yaml), contra dois provedores. Comando, a partir desta pasta:

```powershell
promptfoo eval -c promptfooconfig.yaml --no-cache --output resultado.json
```

| Item | Valor |
|---|---|
| Versão do promptfoo | 0.125.1 |
| Provedor 1 | OpenAI `gpt-6-sol`, `reasoning_effort: low` |
| Provedor 2 | Anthropic `claude-sonnet-5-5`, `thinking.type: between_tools`, `max_tokens: 1024` |
| Prompt testado | `prompt.md` v1.1.0 (arquivo inteiro, com frontmatter) |

Asserts aplicados em toda saída:

| Assert | Tipo | Critério |
|---|---|---|
| Cinco rótulos | `contains-all` | `ALERTA:`, `IMPACTO:`, `HIPÓTESE INICIAL:`, `AÇÃO IMEDIATA:`, `ESCALAR PARA:` |
| Handle | `regex` | `ESCALAR PARA:.*@\w+` |
| Concisão | `javascript` | no máximo 8 linhas |
| Latência | `latency` | até 5000 ms por chamada |
| Custo | `cost` | até US$ 0,01 por chamada |

Resultado da última execução (`Results: 6 passed (100%), 0 failed, 0 errors`):

| Alerta | Modelo | Resultado | Latência | Custo (US$) |
|---|---|---|---|---|
| 1 | gpt-6-sol | PASS | 3337 ms | 0,0033 |
| 1 | claude-sonnet-5-5 | PASS | 3154 ms | 0,0098 |
| 2 | gpt-6-sol | PASS | 1713 ms | 0,0018 |
| 2 | claude-sonnet-5-5 | PASS | 2443 ms | 0,0089 |
| 3 | gpt-6-sol | PASS | 2553 ms | 0,0027 |
| 3 | claude-sonnet-5-5 | PASS | 2936 ms | 0,0094 |

Todas as saídas tiveram cinco linhas. A margem de custo do Sonnet é pequena (US$ 0,0089 a 0,0098 contra o limite de US$ 0,01).

Histórico das execuções até chegar a esse resultado:

| Execução | Prompt | Resultado | Falhas |
|---|---|---|---|
| 1 | v1.0.0 | 0/6 | regex do handle nas 6; custo (US$ 0,010 a 0,011) nas 3 do Sonnet; latência de 8,1 s no GPT (alerta 1) |
| 2 | v1.0.0 com handle padrão | 1/6 | handle corrigido; ainda custo no Sonnet e latência/custo no GPT |
| 3 | v1.1.0 (prompt enxuto) | 2/6 | o Sonnet passou a gastar 1.877 tokens de raciocínio e o custo subiu para US$ 0,014 a 0,017 |
| 4 | v1.1.0 com `thinking: between_tools` | 5/6 | só a latência do GPT no alerta 1 (6,3 a 7,3 s) |
| 5 | v1.1.0 com `reasoning_effort: low` | 6/6 | nenhuma |

## Curadoria

**Framework:** few-shot prompting (o modelo aprende o padrão a partir de poucos exemplos dentro do prompt), porque o padrão da nota é de formato e de tom, e exemplos transmitem isso melhor que uma descrição. Os blocos do prompt mapeiam assim: `<exemplos>` traz as três notas de referência, marcadas como formato e não como entrada; `<regras>` cobre o que os exemplos não mostram (segurança da entrada, alerta inválido, fidelidade ao alerta e tratamento de campo sem dado); `<formato_saida>` fixa a estrutura de cinco linhas e a verificação final.

**Refinos:** o prompt foi criado por meta-prompting com o Opus 5.5 e refinado com revisão humana. O nome e as tags foram alinhados ao template da biblioteca, e o horário foi retirado da linha `ALERTA` para seguir os exemplos. Depois da primeira execução, as regras de `IMPACTO` e `ESCALAR PARA` foram ajustadas para citar o tenant nomeado no alerta e para nunca usar o valor observado como limiar de escalonamento.

**Modelo:** GPT 6 Sol com esforço de raciocínio médio, porque seguiu o formato de cinco linhas e as regras de campo ausente nas três entradas, sem copiar conteúdo dos exemplos.

**Curadoria dos testes (CP08):**

- **O que passou desde a primeira execução:** os cinco rótulos e o limite de 8 linhas (todas as saídas têm cinco linhas) nos dois provedores.
- **O que falhou e por quê:**
  - *Handle (`@\w+`), 6 de 6 saídas.* O prompt v1.0.0 escrevia "time responsável pelo [Componente] (handle não informado no alerta)", por causa da regra de não inventar handle. Isso conflita com o requisito do desafio, que exige `@palavra`. Os alertas do CP02 não trazem metadado de roteamento.
  - *Custo do Sonnet, 3 de 3 alertas (US$ 0,010 a 0,011).* O prompt tinha cerca de 3,8 mil tokens de entrada.
  - *Latência do GPT no alerta 1 (6,3 a 8,1 s).* O tempo variou entre rodadas por causa do raciocínio.
- **O que foi ajustado no prompt (v1.0.0 → v1.1.0):**
  - Sem metadado de roteamento, o destinatário passa a ser o handle padrão `@time-<componente em minúsculas>`, seguido de "(handle padrão; roteamento não informado no alerta)". É uma convenção de preenchimento derivada só do componente citado no alerta, não um destino confirmado, e o plantonista deve confirmar o roteamento real. A regra de nunca usar handles dos exemplos continua valendo.
  - O prompt foi condensado em cerca de 14% (de 8.890 para 7.603 bytes): as três proibições de copiar dos exemplos viraram uma, e a verificação final virou um parágrafo. O comportamento das regras 1 a 5 foi mantido.
- **O que foi ajustado nos testes (configuração):**
  - O Sonnet 5.5 usa raciocínio adaptativo, que gastou 1.877 tokens de raciocínio numa rodada e elevou o custo para US$ 0,014 a 0,017. O `thinking.type: disabled` não é aceito pelo modelo (o promptfoo o converte para `between_tools`, o menor nível), então o config usa `between_tools` explicitamente.
  - O GPT 6 Sol passou a `reasoning_effort: low` para caber nos 5 s. Isso é diferente do esforço médio usado nas execuções manuais acima, e a nota sai igual em estrutura.
- **Trade-off de modelo:** o GPT 6 Sol é o mais barato e o mais rápido nesta tarefa (US$ 0,002 a 0,003), e o Sonnet 5.5 fica abaixo do limite de custo apenas por pouco (US$ 0,009). Os limites de 5 s e US$ 0,01 foram mantidos como no enunciado para este prompt, e ambos os provedores os cumprem.
- **Limitações dos testes:** a variação entre execuções não foi medida em série. Ela existe, e a latência do GPT oscilou de 1,7 s a 7,3 s entre rodadas do mesmo alerta. Os asserts verificam estrutura e presença de elementos. Eles não avaliam se a hipótese ou a ação são corretas, o que fica para a avaliação por julgamento do CP09.
