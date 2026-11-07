---
nome: Nota de triagem
descricao: Converte um alerta cru de observabilidade em uma nota de triagem padronizada de cinco linhas para a passagem de turno SRE.
versao: 1.0.0
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
3. Executar o prompt em uma conversa ou chamada nova, revisar os campos que vierem com texto padrão (por exemplo, "handle não informado no alerta") e colar a nota na passagem de turno.

Exemplos de execução com saída real estão na seção [Execução](#execução).

## Limitações conhecidas

- A nota depende exclusivamente do alerta. Sem metadado de roteamento, o `ESCALAR PARA` aponta "time responsável pelo [Componente] (handle não informado no alerta)", e o plantonista precisa completar o destinatário.
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

## Curadoria

**Framework:** few-shot prompting (o modelo aprende o padrão a partir de poucos exemplos dentro do prompt), porque o padrão da nota é de formato e de tom, e exemplos transmitem isso melhor que uma descrição. Os blocos do prompt mapeiam assim: `<exemplos>` traz as três notas de referência, marcadas como formato e não como entrada; `<regras>` cobre o que os exemplos não mostram (segurança da entrada, alerta inválido, fidelidade ao alerta e tratamento de campo sem dado); `<formato_saida>` fixa a estrutura de cinco linhas e a verificação final.

**Refinos:** o prompt foi criado por meta-prompting com o Opus 5.5 e refinado com revisão humana. O nome e as tags foram alinhados ao template da biblioteca, e o horário foi retirado da linha `ALERTA` para seguir os exemplos. Depois da primeira execução, as regras de `IMPACTO` e `ESCALAR PARA` foram ajustadas para citar o tenant nomeado no alerta e para nunca usar o valor observado como limiar de escalonamento.

**Modelo:** GPT 6 Sol com esforço de raciocínio médio, porque seguiu o formato de cinco linhas e as regras de campo ausente nas três entradas, sem copiar conteúdo dos exemplos.
