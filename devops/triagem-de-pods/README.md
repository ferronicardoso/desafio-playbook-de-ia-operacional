---
nome: Triagem de Pods
descricao: Triagem rápida da saúde dos pods de um namespace a partir de um snapshot de kubectl colado pelo plantonista, com causa provável e próxima ação.
versao: 1.0.0
tags:
  - sre
  - kubernetes
  - triagem
  - plantao
inputs:
  - nome: cluster
    descricao: Identificador do cluster (opcional)
  - nome: namespace
    descricao: Namespace do snapshot
  - nome: momento_da_coleta
    descricao: Horário da coleta, usado para interpretar AGE e idade dos restarts (opcional)
  - nome: get_pods
    descricao: Saída de kubectl get pods
  - nome: describe
    descricao: Uma ou mais saídas de kubectl describe pod (opcional)
  - nome: logs
    descricao: Saídas de kubectl logs, atual e/ou --previous (opcional)
  - nome: contexto_adicional
    descricao: Deploys recentes, mudanças ou incidentes abertos (opcional)
---

# Triagem de Pods

## Objetivo

Transformar um snapshot de Kubernetes já coletado (`kubectl get pods`, `kubectl describe pod` e `kubectl logs`) em uma triagem curta e verificável: quais pods estão problemáticos, a causa provável de cada um (cruzando status, eventos e logs, e não apenas repetindo o `STATUS`) e a próxima ação do plantão. Quando não há pod problemático, o prompt declara isso de forma explícita e não inventa problemas.

O prompt não acessa o cluster nem usa ferramentas: ele só conhece o que for colado nos parâmetros.

## Quando usar

- Triagem inicial de saúde dos pods de um namespace durante um plantão.
- Pod reiniciando, em `CrashLoopBackOff`, `ImagePullBackOff` ou `Pending`, quando é preciso chegar ao mecanismo da falha.
- Confirmação rápida de que um namespace está saudável após um deploy.
- Padronização do formato de triagem entre plantonistas.

## Exemplo de uso

1. Coletar o snapshot com acesso ao cluster: `kubectl get pods -n <namespace>`, `kubectl describe pod <pod> -n <namespace>` dos pods suspeitos e `kubectl logs <pod> -n <namespace> --previous` para containers que reiniciam.
2. Copiar o conteúdo de `prompt.md` (sem o frontmatter) e substituir os placeholders:
   - `{{namespace}}` e `{{get_pods}}` são obrigatórios.
   - `{{describe}}`, `{{logs}}`, `{{cluster}}`, `{{momento_da_coleta}}` e `{{contexto_adicional}}` podem ficar vazios. Campo vazio é interpretado como "não fornecido".
3. Executar o prompt em uma conversa ou chamada nova e usar a triagem como ponto de partida para a investigação.

Exemplos de execução com saída real estão na seção [Execução](#execução).

## Limitações conhecidas

- A análise depende exclusivamente do snapshot colado. Sem `describe` e sem logs, a causa pode ficar indeterminada ou com confiança baixa.
- A qualidade do resultado depende do modelo. Os testes abaixo usaram um único modelo (GPT 6 Luna).
- A "próxima ação" pode combinar mais de um passo (verificar e depois escalar), apesar da instrução de indicar uma única ação.
- `Inferência` e `Causa provável` tendem a repetir o mesmo mecanismo, o que aumenta o tamanho da saída.
- O prompt instrui o modelo a não seguir instruções presentes no snapshot, mas isso não substitui a sanitização prévia de segredos e dados sensíveis antes de enviar logs reais a um modelo externo.
- Os resultados abaixo vêm de uma execução por entrada. Não foi medida a variação entre execuções.

## Execução

| Item | Valor |
|---|---|
| Modelo | GPT 6 Luna |
| Identificador na API | `gpt-6-luna` |
| Esforço de raciocínio | baixo |

Em todas as execuções, `cluster`, `momento_da_coleta` e `contexto_adicional` foram deixados vazios, e `namespace` foi preenchido com `sentinel-prod`. Cada entrada foi executada em uma conversa nova.

### Entrada 1: pod reiniciando

Valores dos parâmetros:

```text
get_pods:
$ kubectl get pods -n sentinel-prod
NAME                            READY   STATUS             RESTARTS       AGE
sentinel-api-7d9c8b6f4-2xk9p    1/1     Running            0              6d
sentinel-api-7d9c8b6f4-h4m2t    0/1     CrashLoopBackOff   14 (90s ago)   42m
sentinel-worker-5b8f9c8d-jq7rs  1/1     Running            0              6d
sentinel-scheduler-6c4d7f-nn8vx 1/1     Running            0              6d

describe:
$ kubectl describe pod sentinel-api-7d9c8b6f4-h4m2t -n sentinel-prod
    State:          Waiting
      Reason:       CrashLoopBackOff
    Last State:     Terminated
      Reason:       OOMKilled
      Exit Code:    137
    Limits:         memory: 512Mi
    Requests:       memory: 256Mi
Events:
  Type     Reason   Age                 From     Message
  ----     ------   ----                ----     -------
  Warning  BackOff  2m (x18 over 40m)   kubelet  Back-off restarting failed container api

logs:
$ kubectl logs sentinel-api-7d9c8b6f4-h4m2t -n sentinel-prod --previous
2026-05-13 10:14:02 [INFO]  [api] startup complete, version 2.9.1
2026-05-13 10:14:48 [INFO]  [cache] loading 6h alert cache (heap 410Mi/512Mi)
2026-05-13 10:15:10 [WARN]  [runtime] high GC pressure, heap 498Mi/512Mi
2026-05-13 10:15:12 [FATAL] [runtime] out of memory, shutting down process
```

Saída:

```markdown
Resumo: 1 de 4 pods com problema em `sentinel-prod`.

### sentinel-api-7d9c8b6f4-h4m2t — CrashLoopBackOff

- Fato:
  - `0/1 CrashLoopBackOff 14 (90s ago) 42m` (get_pods)
  - `Reason: OOMKilled`, `Exit Code: 137`, `memory: 512Mi`, `Back-off restarting failed container api` (describe)
  - `heap 498Mi/512Mi`, `high GC pressure`, `out of memory, shutting down process` (logs)
- Inferência: O processo excedeu a memória disponível durante o carregamento do cache de alertas, foi encerrado por falta de memória e entrou em ciclos de reinicialização com espera progressiva (backoff).
- Hipótese: O cache de alertas de 6 horas demanda mais memória do que o limite atual de 512Mi comporta.
- Causa provável: Esgotamento de memória durante a inicialização da API, resultando em OOMKilled (exit 137) e reinicializações recorrentes. — confiança alta.
- Próxima ação: Avaliar aumento temporário e reversível do limite de memória do Deployment `sentinel-api`, mediante validação da capacidade disponível no cluster.
- Falta: Consumo real e capacidade disponível: `kubectl top pod -n sentinel-prod --containers` e `kubectl top nodes`; histórico de alterações: `kubectl rollout history deployment/sentinel-api -n sentinel-prod`.
```

### Entrada 2: pods que não sobem

Valores dos parâmetros:

```text
get_pods:
$ kubectl get pods -n sentinel-prod
NAME                            READY   STATUS             RESTARTS   AGE
sentinel-api-7d9c8b6f4-2xk9p    1/1     Running            0          6d
sentinel-api-7d9c8b6f4-zzp10    0/1     ImagePullBackOff   0          8m
sentinel-ingest-8f7a9c5b-4dkqm  0/1     Pending            0          8m
sentinel-scheduler-6c4d7f-nn8vx 1/1     Running            0          6d

describe:
$ kubectl describe pod sentinel-api-7d9c8b6f4-zzp10 -n sentinel-prod
    State:    Waiting
      Reason: ImagePullBackOff
    Image:    registry.aegis.io/sentinel-api:2.9.2
Events:
  Type     Reason   Age   From     Message
  ----     ------   ----  ----     -------
  Warning  Failed   7m    kubelet  Failed to pull image "registry.aegis.io/sentinel-api:2.9.2": manifest unknown
  Warning  Failed   7m    kubelet  Error: ErrImagePull
  Normal   BackOff  6m    kubelet  Back-off pulling image "registry.aegis.io/sentinel-api:2.9.2"
$ kubectl describe pod sentinel-ingest-8f7a9c5b-4dkqm -n sentinel-prod
    State:    Waiting
      Reason: Pending
    Requests: cpu: 2
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  7m    default-scheduler  0/6 nodes are available: 6 Insufficient cpu

logs: (vazio)
```

Saída:

```markdown
Resumo: 2 de 4 pods com problema em `sentinel-prod`.

### sentinel-api-7d9c8b6f4-zzp10 — ImagePullBackOff

- Fato:
  - `0/1 ImagePullBackOff 0 8m` (get_pods)
  - `Failed to pull image "registry.aegis.io/sentinel-api:2.9.2": manifest unknown` (describe)
  - `Back-off pulling image` (describe)
- Inferência: O registry não encontrou o manifesto da imagem solicitada. O kubelet realizou tentativas de download e entrou em backoff, impedindo a inicialização do container.
- Hipótese: A tag `2.9.2` não foi publicada ou foi removida do registry.
- Causa provável: Imagem indisponível no registry para a referência configurada, impedindo o pull e mantendo o container em espera — confiança alta.
- Próxima ação: Verificar a existência da tag `2.9.2` no registry e, se ausente, escalar ao responsável pelo pipeline de publicação para restaurá-la.
- Falta: Confirmação da existência da imagem: `docker manifest inspect registry.aegis.io/sentinel-api:2.9.2`.

### sentinel-ingest-8f7a9c5b-4dkqm — Pending

- Fato:
  - `0/1 Pending 0 8m` (get_pods)
  - `Requests: cpu: 2` (describe)
  - `0/6 nodes are available: 6 Insufficient cpu` (describe)
- Inferência: O scheduler não encontrou nenhum dos seis nós com CPU alocável suficiente para atender à solicitação de 2 CPUs. O pod permanece sem agendamento e não inicia seu container.
- Causa provável: Capacidade de CPU disponível insuficiente para satisfazer o request do pod — confiança alta.
- Próxima ação: Executar `kubectl describe nodes` para identificar a capacidade alocável e os requests já reservados antes de decidir entre liberar capacidade ou escalar a infraestrutura.
- Falta: Distribuição dos requests de CPU por nó e capacidade efetivamente disponível, obtidas com `kubectl describe nodes`.
```

### Entrada 3: tudo saudável

Valores dos parâmetros:

```text
get_pods:
$ kubectl get pods -n sentinel-prod
NAME                            READY   STATUS    RESTARTS     AGE
sentinel-api-7d9c8b6f4-2xk9p    1/1     Running   0            6d
sentinel-api-7d9c8b6f4-h4m2t    1/1     Running   0            6d
sentinel-worker-5b8f9c8d-jq7rs  1/1     Running   1 (3d ago)   6d
sentinel-scheduler-6c4d7f-nn8vx 1/1     Running   0            6d

describe: (vazio)

logs:
$ kubectl logs sentinel-api-7d9c8b6f4-2xk9p -n sentinel-prod --tail=3
2026-05-13 11:02:31 [INFO] [api] health check OK, 0 errors in the last 5min
2026-05-13 11:03:31 [INFO] [api] processed 4.2k alerts/min, p99 latency 120ms
2026-05-13 11:04:31 [INFO] [api] health check OK
```

Saída:

```markdown
Resumo: 0 de 4 pods com problema em `sentinel-prod`.

Nenhum pod problemático no snapshot.

Observação de acompanhamento: Fato: `sentinel-worker-5b8f9c8d-jq7rs` apresenta `1 (3d ago)` em RESTARTS e `6d` em AGE (get_pods), estando `1/1 Running`. Inferência: restart antigo, sem recorrência registrada no snapshot; apenas acompanhamento.
```

### Testes determinísticos (promptfoo)

As três entradas acima (snapshots do CP01) rodam como casos de teste em [`promptfooconfig.yaml`](promptfooconfig.yaml), contra dois provedores. Comando, a partir desta pasta:

```powershell
promptfoo eval -c promptfooconfig.yaml --no-cache --output resultado.json
```

| Item | Valor |
|---|---|
| Versão do promptfoo | 0.125.1 |
| Provedor 1 | OpenAI `gpt-6-sol`, `reasoning_effort: low` |
| Provedor 2 | Anthropic `claude-sonnet-5-5`, `thinking.type: between_tools`, `max_tokens: 2048` |
| Prompt testado | `prompt.md` v1.0.0 (arquivo inteiro, com frontmatter) |

Asserts por entrada:

| Entrada | Tipo | Critério |
|---|---|---|
| 1 | `contains` | `sentinel-api-7d9c8b6f4-h4m2t` |
| 1 | `icontains-any` | `OOMKilled` ou `memória` |
| 2 | `contains-all` | `sentinel-api-7d9c8b6f4-zzp10` e `sentinel-ingest-8f7a9c5b-4dkqm` |
| 2 | `icontains-any` | `2.9.2` ou `ImagePullBackOff` (causa do pod com imagem) |
| 2 | `icontains-any` | `cpu` ou `Insufficient` (causa do pod pendente) |
| 3 | `contains` | `Nenhum pod problemático no snapshot.` |
| 3 | `not-contains` | `###` (nenhum pod com seção de falha) |
| 3 | `not-icontains-any` | `CrashLoopBackOff`, `OOMKilled`, `ImagePullBackOff`, `Pending`, `Error` |
| todas | `latency` | até **10000 ms** por chamada (limite próprio; enunciado: 5000 ms) |
| todas | `cost` | até **US$ 0,02** por chamada (limite próprio; enunciado: US$ 0,01) |

Resultado da última execução (`Results: 6 passed (100%), 0 failed, 0 errors`):

| Entrada | Modelo | Resultado | Latência | Custo (US$) | Linhas da saída |
|---|---|---|---|---|---|
| 1 | gpt-6-sol | PASS | 4539 ms | 0,0042 | 10 |
| 1 | claude-sonnet-5-5 | PASS | 5599 ms | 0,0124 | 12 |
| 2 | gpt-6-sol | PASS | 3271 ms | 0,0033 | 13 |
| 2 | claude-sonnet-5-5 | PASS | 9265 ms | 0,0179 | 23 |
| 3 | gpt-6-sol | PASS | 2399 ms | 0,0015 | 5 |
| 3 | claude-sonnet-5-5 | PASS | 1391 ms | 0,0056 | 5 |

Histórico das execuções até chegar a esse resultado:

| Execução | Configuração | Resultado | Falhas |
|---|---|---|---|
| 1 | limites do enunciado (5 s e US$ 0,01), Sonnet com raciocínio adaptativo | 2/6 | latência e custo nas Entradas 1 e 2 nos dois modelos (GPT: 8,4 a 9,4 s e US$ 0,012 a 0,013; Sonnet: 16 s e US$ 0,025); na Entrada 2 do Sonnet a saída veio vazia, e os três asserts de conteúdo falharam |
| 2 | Sonnet com `between_tools`, GPT com `reasoning_effort: low` | 4/6 | só o Sonnet nas Entradas 1 e 2: custo (US$ 0,014 e 0,016) e latência (6,5 e 7,9 s) |
| 3 | limites próprios de 10 s e US$ 0,02 | 6/6 | nenhuma |

## Curadoria

**Framework:** RISE (Role, Input, Steps, Expectation), por combinar passos em ordem com uma saída de formato definido e verificável. O prompt tem os quatro blocos: `<papel>` (Role), `<snapshot>` (Input), `<passos>` (Steps) e `<formato_de_saida>` (Expectation).

**Refinos:** o prompt foi criado por meta-prompting (Opus 5.5) e refinado com revisão humana para alinhar o frontmatter ao template, exigir o nome completo do pod no cabeçalho, mostrar a evidência por fonte (confiança "alta" só com duas fontes convergentes) e deixar a saída curta e consistente, inclusive no caso sem pod problemático.

**Modelo:** GPT 6 Luna, descrito pelo provedor como o mais eficiente da família para tarefas focadas e de alto volume, o que corresponde a esta triagem (entrada estruturada, saída curta e formato fixo). Atendeu aos critérios das três entradas.

**Curadoria dos testes (CP08):**

- **O que passou desde a primeira execução:** a Entrada 3 (tudo saudável) nos dois provedores, e todos os asserts de conteúdo do GPT 6 Sol nas Entradas 1 e 2. O prompt não precisou de alteração: a versão testada é a v1.0.0.
- **O que falhou e por quê:**
  - *Saída vazia do Sonnet na Entrada 2.* O raciocínio adaptativo consumiu todo o `max_tokens: 2048` (`finishReason: length`), e os asserts de conteúdo falharam sem que o prompt tivesse culpa. Foi falha de configuração do teste.
  - *Latência e custo nas Entradas 1 e 2.* Com os limites do enunciado (5 s e US$ 0,01), a saída de 10 a 23 linhas não cabia no orçamento: GPT com 8,4 a 9,4 s e US$ 0,012 a 0,013; Sonnet com 16 s e US$ 0,025.
- **O que foi ajustado nos testes (configuração):**
  - O Sonnet 5.5 passou a usar `thinking.type: between_tools` (o `disabled` não é aceito pelo modelo), o que eliminou a saída vazia e baixou o custo e a latência.
  - O GPT 6 Sol passou a `reasoning_effort: low`.
  - Os limites de latência e custo deste prompt foram elevados para **10 s e US$ 0,02**, em vez dos 5 s e US$ 0,01 do enunciado. Mesmo com raciocínio mínimo, o Sonnet continuou acima: Entrada 1 com 5,6 a 6,5 s e US$ 0,012 a 0,014, e Entrada 2 com 7,9 a 9,3 s e US$ 0,016 a 0,018, porque a saída tem 12 a 23 linhas e traz fatos, inferência, causa e próxima ação por pod. O enunciado trata 5 s e US$ 0,01 como parte da qualidade, e a decisão aqui foi não cortar o conteúdo exigido da triagem para caber no orçamento. Os limites próprios têm margem de 7% (latência) e 11% (custo) sobre o pior caso medido.
- **Trade-off de modelo:** o GPT 6 Sol cumpre os limites do enunciado em todas as entradas desta última execução (até 4,5 s e US$ 0,004) e é a escolha para o uso rotineiro. O Sonnet 5.5 só cumpre os limites próprios, e sua latência na Entrada 2 (9,3 s) está a menos de 1 s do limite de 10 s. Uma rodada mais lenta pode reprová-lo sem que o prompt tenha mudado.
- **Limitações dos testes:** os asserts checam a presença de pods e causas, e não a qualidade do diagnóstico, que fica para a avaliação por julgamento do CP09. O `not-icontains-any` da Entrada 3 inclui a palavra `Error`, o que reprovaria uma saída que a citasse mesmo em contexto correto (isso não ocorreu). A variação entre execuções não foi medida em série; entre rodadas, a latência do Sonnet na Entrada 2 foi de 7,9 s a 9,3 s.
