---
nome: Endurecimento de NetworkPolicy
descricao: Analisa uma NetworkPolicy permissiva e gera manifestos endurecidos e rastreáveis a um padrão de segurança, usando somente identificadores do mapa de serviços.
versao: 1.0.0
tags:
  - kubernetes
  - networkpolicy
  - seguranca
  - default-deny
  - endurecimento
inputs:
  - nome: manifesto
    descricao: Manifesto YAML da NetworkPolicy atual, considerada permissiva demais, que será analisada e corrigida.
  - nome: padrao
    descricao: Regras do padrão de segurança de rede que a política corrigida deve atender, preferencialmente com um identificador por regra.
  - nome: mapa_de_servicos
    descricao: Mapa de como cada serviço é identificado no cluster (namespace, labels de pod e de namespace, portas, protocolos e fluxos legítimos entre serviços).
---

# Endurecimento de NetworkPolicy

## Objetivo

Receber o manifesto de uma NetworkPolicy do Kubernetes considerada permissiva demais, as regras de um padrão de segurança e um mapa de como cada serviço é identificado no cluster, e devolver a versão corrigida e endurecida, pronta para revisão de segurança.

A saída traz cinco seções: a análise do manifesto original (o que o torna permissivo e qual regra do padrão viola); os manifestos corrigidos em YAML, com uma política de negação por padrão e uma política de liberação por fluxo, cada regra com comentário do fluxo que libera; a rastreabilidade entre cada regra do padrão e o trecho do YAML que a implementa; as lacunas e premissas, incluindo a comparação entre o manifesto original e o corrigido; e a validação antes de aplicar. O prompt usa somente namespaces, labels, portas e protocolos do mapa, rotula cada afirmação como fato, premissa ou julgamento e registra como lacuna o que o mapa não informa.

## Quando usar

- Quando uma NetworkPolicy foi barrada pela revisão de segurança por liberar mais do que o necessário (`podSelector: {}`, regras `- {}`, ausência de `ports`) e é preciso uma versão corrigida e auditável.
- Para converter um padrão de segurança de rede (ingress permitido, egress permitido, default-deny, comentário por regra) em manifestos rastreáveis regra a regra, a partir de um mapa de serviços.
- Antes de aplicar uma política em produção, para obter a lista de lacunas que ainda dependem de informação do time dono do serviço e um roteiro de validação.

## Exemplo de uso

1. Copiar o conteúdo de `prompt.md` (sem o frontmatter).
2. Substituir `{{manifesto}}` pelo YAML da NetworkPolicy permissiva, `{{padrao}}` pelas regras do padrão e `{{mapa_de_servicos}}` pelo mapa (namespace, label de pod e porta de cada serviço). Os três parâmetros são obrigatórios.
3. Executar em uma conversa ou chamada nova.
4. Ler primeiro a seção 4 (lacunas e premissas) e conferir a seção 3 (rastreabilidade) contra o padrão. Revisar cada premissa de protocolo e a label de namespace.
5. Resolver as lacunas com os responsáveis pelos serviços (em especial as portas de ingress) e executar a validação da seção 5 em ambiente não produtivo antes de aplicar.

Um exemplo de execução com saída real está na seção [Execução](#execução).

## Limitações conhecidas

- O prompt não executa nada no cluster. Os comandos de validação da saída não foram executados, e o isolamento efetivo depende do CNI, de outras NetworkPolicies que selecionem os mesmos pods (efeito aditivo) e de pods com `hostNetwork`. A saída registra isso como lacuna.
- Quando o mapa não informa a porta de ingress, a regra é gerada com origem restrita e sem `ports`, ou seja, libera todas as portas e protocolos daquela origem até a lacuna ser resolvida. É menos permissiva que o original, mas não é menor privilégio em nível de porta. Essa é uma decisão de desenho que o time deve revisar.
- O protocolo de serviços nomeados no mapa (Postgres e Elasticsearch em TCP, DNS em UDP e TCP) entra como premissa, não como fato. Se o serviço real usar outro protocolo, a comunicação fica bloqueada.
- A label `kubernetes.io/metadata.name` é premissa: em clusters anteriores ao Kubernetes 1.21 que não a atribuem, os seletores de namespace não funcionam e os fluxos entre namespaces ficam bloqueados.
- Dependências de infraestrutura que o mapa não identifica (API do Kubernetes, health checks, telemetria) ficam bloqueadas e aparecem como lacuna. A saída não as libera.
- Na execução registrada, os comentários YAML das regras de ingress trazem a lacuna, mas não repetem o risco rotulado como julgamento (o rótulo está na seção 4), e o prompt pede os dois.
- As seções 1 e 5 da saída trazem subtítulos e blocos que o formato não prevê (inventário de fluxos legítimos, comandos de validação, critérios para aprovação), e a comparação entre original e corrigido não inclui uma linha para a saída do Sentinel ao Relay, embora a nota de direção esteja na seção 4.
- Uma premissa da saída cita "a especificação da tarefa", expressão que não consta das entradas e que vem do texto do prompt.
- O prompt tem cerca de 1.975 palavras, contra 1.800 do limite inicial (cerca de 10% acima), porque as exceções de protocolo e de porta ausente exigiram texto. Não foi cortado mais para não remover regras úteis.
- O resultado vem de um único modelo e de uma execução com a versão final do prompt. As versões v1 a v3 do histórico foram feitas em um mesmo chat, e a v1 usou uma versão anterior do prompt.

## Execução

| Item | Valor |
|---|---|
| Modelo | GPT 6 Sol |
| Identificador na API | `gpt-6-sol` |
| Esforço de raciocínio | médio |

A execução tem duas partes. Primeiro, a v1 e as duas rodadas de verificação (v2 e v3), feitas em um mesmo chat, com a versão do prompt anterior aos ajustes descritos em [Curadoria](#curadoria). Depois, uma execução de validação com o prompt corrigido, em chat novo e temporário, sem acesso à memória, para confirmar que ele chega à v3 sem direção humana.

### Entradas

`manifesto` (manifesto barrado pela revisão de segurança):

```yaml
# manifesto barrado pela revisão de segurança — permissivo demais
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: sentinel-allow
  namespace: sentinel-prod
spec:
  podSelector: {}          # aplica a todos os pods do namespace
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - {}                   # libera QUALQUER origem
  egress:
    - {}                   # libera QUALQUER destino
```

`padrao`:

```text
NetworkPolicy para o namespace "sentinel-prod":
- pods do Sentinel só aceitam tráfego de entrada do Relay (consumo de
  eventos) e do gateway de API da plataforma
- pods do Sentinel só fazem saída para: Forge (warehouse, porta 5432),
  Cerebro (busca, porta 9200) e DNS interno
- nada de "allow all" em ingress ou egress
- política default-deny explícita no namespace
- toda regra precisa de comentário dizendo qual fluxo legítimo ela libera
```

`mapa_de_servicos`:

```text
Sentinel     → namespace sentinel-prod, pods com label app=sentinel
Relay        → namespace relay-prod,    pods com label app=relay
API gateway  → namespace edge,          pods com label app=api-gateway
Forge        → namespace forge-prod,    pods com label app=forge,   porta 5432 (Postgres do warehouse)
Cerebro      → namespace cerebro-prod,  pods com label app=cerebro, porta 9200 (Elasticsearch)
DNS interno  → namespace kube-system,   pods com label k8s-app=kube-dns, porta 53
```

### Iteração 1: v1

A saída da v1 tem as cinco seções. O diagnóstico do manifesto original está correto, com o caminho no YAML de cada trecho, e as lacunas L-01 a L-07 cobrem portas de ingress, protocolos de Forge, Cerebro e DNS, dependências de infraestrutura e CNI. O modelo não inventou labels, portas nem serviços, e registrou que o default-deny não elimina o `sentinel-allow` enquanto este existir (efeito aditivo).

O problema é o manifesto: um único recurso de negação por padrão, com `ingress: []` e `egress: []` e nenhuma regra de liberação. O modelo tratou a falta de protocolo no mapa como motivo para bloquear os cinco fluxos legítimos, inclusive o DNS. A própria conclusão diz que "não se recomenda aplicar essa configuração isoladamente em produção". Manifesto da v1:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: sentinel-default-deny
  namespace: sentinel-prod
spec:
  # P-04: isola todos os pods de sentinel-prod.
  # A-02: nome do recurso definido nesta correção.
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress

  # P-01: nenhuma entrada é liberada até a
  # identificação completa dos fluxos legítimos.
  #
  # L-01: Relay -> Sentinel.
  # Pendente: porta e protocolo de entrada.
  #
  # L-02: API gateway -> Sentinel.
  # Pendente: porta e protocolo de entrada.
  ingress: []

  # P-02: nenhuma saída é liberada até a
  # identificação completa dos fluxos legítimos.
  #
  # L-03: Sentinel -> Forge, porta 5432.
  # Pendente: protocolo.
  #
  # L-04: Sentinel -> Cerebro, porta 9200.
  # Pendente: protocolo.
  #
  # L-05: Sentinel -> DNS interno, porta 53.
  # Pendente: protocolo ou protocolos.
  #
  # L-06: outras dependências de infraestrutura
  # sem identificação completa no mapa.
  egress: []
```

Achados da conferência da v1 (feita antes da verificação pelo modelo):

| # | Achado |
|---|---|
| 1 | O prompt não permitia inferir o protocolo de um serviço nomeado (Postgres, Elasticsearch, DNS), o que bloqueou todos os fluxos |
| 2 | O ingress sem porta foi bloqueado em vez de liberado com origem restrita e a lacuna sinalizada |
| 3 | A saída não tem nenhuma regra com seletores, então a premissa da label do namespace e o E/OU dos seletores ficaram sem uso |
| 4 | A direção do Relay (consumo de eventos) não foi registrada como ambiguidade |
| 5 | Subseções extras e uma tabela de comparação sem lugar previsto no formato, e rótulo `(fato)` em atribuições que são interpretação |

### Iteração 2: verificação pelo modelo e v2

Pedido de verificação, enviado no mesmo chat:

```text
Atue agora como revisor de segurança de plataformas Kubernetes, e não como autor da resposta anterior. Critique a v1 que você produziu.

1. Liste as perguntas de verificação que um revisor de segurança faria sobre essa NetworkPolicy, cobrindo pelo menos: se atende cada regra do padrão, se bloqueia fluxos legítimos, correção dos seletores (E/OU), portas e protocolos, DNS, direção dos fluxos e efeito aditivo das políticas.
2. Responda a cada pergunta com evidência extraída da v1 ou do mapa. Marque cada resposta como atendida, parcial ou falha.
3. Para cada falha ou parcial, proponha a correção mínima.
4. Entregue a v2 completa, no mesmo formato da v1, e uma tabela "ponto apontado → como a v2 endereçou".
```

O modelo levantou 16 perguntas (R-01 a R-16):

| ID | Pergunta | Situação na v1 |
|---|---|---|
| R-01 | P-01 permite somente Relay e API gateway? | Parcial |
| R-02 | P-02 permite Forge, Cerebro e DNS e bloqueia outros destinos? | Parcial |
| R-03 | P-03 elimina regras allow all? | Atendida |
| R-04 | P-04 implementa default-deny explícito? | Atendida |
| R-05 | P-05 exige comentários em cada regra de liberação; foi atendida? | Falha |
| R-06 | O escopo da política está correto? | Atendida |
| R-07 | Os seletores entre namespaces usam interseção (E)? | Parcial |
| R-08 | Portas e protocolos são explícitos e comprovados? | Parcial |
| R-09 | O DNS interno funciona de acordo com o mapa? | Parcial |
| R-10 | A direção de cada fluxo foi identificada corretamente? | Atendida |
| R-11 | O efeito aditivo foi tratado como critério obrigatório? | Falha |
| R-12 | As dependências de infraestrutura foram tratadas sem inventar destinos? | Atendida |
| R-13 | O resultado foi validado quanto à sintaxe e ao CNI? | Parcial |
| R-14 | A ordem de implantação evita permissividade residual e indisponibilidade? | Falha |
| R-15 | Os fluxos removidos foram comparados com os da política original? | Parcial |
| R-16 | A v1 respeitou a proibição de inventar namespaces, labels, portas e protocolos? | Atendida |

O parecer do modelo informou 5 atendidas, 8 parciais e 3 falhas, mas a tabela tem 6, 7 e 3. Como v2, o modelo manteve o default-deny sem nenhuma regra de liberação e justificou: "não é aceitável reduzir as lacunas simplesmente escolhendo protocolos usuais de aplicações conhecidas". Ponto apontado → como a v2 endereçou:

| Ponto apontado | Como a v2 endereçou | Resultado |
|---|---|---|
| R-01, ausência de liberações de ingress | Identifica Relay e gateway separadamente e exige porta e protocolo antes de criar as regras | Documentado; pendente |
| R-02, ausência de liberações de egress | Identifica Forge, Cerebro e DNS e preserva os bloqueios até confirmação | Documentado; pendente |
| R-05, P-05 classificado como não aplicável | Altera P-05 para parcial | Corrigido |
| R-07, E/OU sem verificação concreta | Explicita a estrutura dos seletores combinados, sem regra no YAML | Desenho definido; implementação pendente |
| R-08, portas e protocolos incompletos | Adiciona matriz por fluxo com dados conhecidos e ausentes | Rastreabilidade corrigida |
| R-09, DNS bloqueado | Mantém L-05 e exige confirmação do caminho efetivo | Pendente |
| R-11, efeito aditivo | Define inventário e testes de isolamento efetivo | Procedimento corrigido |
| R-13, validação não executada | Separa validação estática, dry-run e teste de enforcement | Critério corrigido |
| R-14, migração com risco operacional | Introduz etapas de aprovação e critérios de interrupção | Procedimento corrigido |
| R-15, remoção não comprovada | Diferencia estado declarado e efetivo | Análise corrigida |

Manifesto da v2 (semanticamente igual ao da v1, com comentários mais detalhados):

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: sentinel-default-deny
  namespace: sentinel-prod
spec:
  # P-04: default-deny para todos os pods
  # do namespace sentinel-prod.
  #
  # A-02: nome escolhido para o recurso.
  #
  # O seletor vazio é intencional apenas aqui:
  # o padrão exige isolamento no namespace.
  podSelector: {}

  # P-04: isolamento nas duas direções.
  policyTypes:
    - Ingress
    - Egress

  # P-01/P-03: negar entrada por padrão.
  #
  # L-01: Relay -> Sentinel.
  # Origem: relay-prod, app=relay.
  # Destino: sentinel-prod, app=sentinel.
  # Porta e protocolo pendentes.
  #
  # L-02: API gateway -> Sentinel.
  # Origem: edge, app=api-gateway.
  # Destino: sentinel-prod, app=sentinel.
  # Porta e protocolo pendentes.
  #
  # P-05: nenhuma regra de liberação criada
  # enquanto os atributos estiverem incompletos.
  ingress: []

  # P-02/P-03: negar saída por padrão.
  #
  # L-03: Sentinel -> Forge.
  # Destino: forge-prod, app=forge.
  # Porta: 5432. Protocolo pendente.
  #
  # L-04: Sentinel -> Cerebro.
  # Destino: cerebro-prod, app=cerebro.
  # Porta: 9200. Protocolo pendente.
  #
  # L-05: Sentinel -> DNS interno.
  # Destino: kube-system, k8s-app=kube-dns.
  # Porta: 53. Protocolo(s) pendente(s).
  #
  # L-06: dependencias de infraestrutura
  # sem identificacao completa no mapa.
  #
  # P-05: excecoes futuras devem documentar
  # origem -> destino, porta/protocolo e ID.
  egress: []
```

Parecer da v2: "aprovável como manifesto-base de default-deny, mas não como solução completa pronta para produção". O modelo declarou que a mudança entre v1 e v2 é "predominantemente de rigor técnico, rastreabilidade e controle de implantação, e não de abertura de fluxos".

### Iteração 3: verificação dirigida e v3

A autoverificação não encontrou a falha central: o manifesto bloqueava todos os fluxos do Sentinel. A rodada seguinte foi dirigida por humano, com o pedido abaixo, no mesmo chat:

```text
Um revisor de disponibilidade reprovaria esta v2: o manifesto bloqueia todos os fluxos do Sentinel, inclusive o DNS, e portanto não opera. Bloquear fluxo legítimo que poderia ser liberado sem inventar valores é falha, não "parcial".

Refaça a revisão e entregue a v3 com estas orientações:
1. O protocolo de serviços nomeados no mapa pode entrar como (premissa) com identificador: Postgres (5432) e Elasticsearch (9200) em TCP; DNS (53) em UDP e TCP. Isso não vale para portas.
2. Para Relay e API gateway, o mapa dá origem e destino mas não a porta de entrada no Sentinel. Gere as regras de ingress com origem restrita (namespaceSelector e podSelector no mesmo item), sem `ports`, em política separada, com comentário "LACUNA: porta não informada; libera todas as portas desta origem" e o risco rotulado como (julgamento).
3. Mantenha namespaceSelector com `kubernetes.io/metadata.name` conforme a premissa A-03.
4. Registre como nota ou lacuna a direção do Relay: o padrão não autoriza egress do Sentinel ao Relay.
5. Inclua a tabela de comparação entre o manifesto original e o corrigido, e corrija a contagem do parecer da revisão (6 atendidas, 7 parciais, 3 falhas).
6. Comentários do YAML com acentuação correta e bloco de código `yaml`.

Entregue a v3 completa, no mesmo formato, e a tabela "ponto apontado → como a v3 endereçou".
```

A v3 reclassificou a v2 em 6 atendidas, 7 parciais e 3 falhas (R-02, R-09 e R-14 passaram a falha; R-05 e R-11 a parcial) e entregou seis recursos: o default-deny, duas políticas de ingress (Relay e gateway) com origem restrita e sem `ports`, e três políticas de egress (Forge TCP/5432, Cerebro TCP/9200 e DNS UDP e TCP/53). Ponto apontado → como a v3 endereçou:

| Ponto apontado | Como a v3 endereçou | Resultado |
|---|---|---|
| A v2 bloqueava todos os fluxos | Acrescentou cinco políticas de liberação para os serviços identificados | Corrigido no YAML |
| Forge sem acesso | Liberou `forge-prod/app=forge` em TCP/5432 | Corrigido sob A-04 |
| Cerebro sem acesso | Liberou `cerebro-prod/app=cerebro` em TCP/9200 | Corrigido sob A-05 |
| DNS bloqueado | Liberou UDP e TCP 53 para `kube-system/k8s-app=kube-dns` | Corrigido sob A-06; sujeito a L-05 |
| Relay sem ingresso | Ingress com origem `relay-prod/app=relay` | Corrigido com risco L-01 |
| Gateway sem ingresso | Ingress com origem `edge/app=api-gateway` | Corrigido com risco L-02 |
| Portas de ingress desconhecidas | Omitiu `ports` somente nessas duas regras, com comentários de lacuna e risco | Exceção documentada |
| Seletores E/OU não demonstrados | Aplicou os dois seletores no mesmo item de `from` e `to` | Corrigido |
| Identificação de namespace | Aplicou `kubernetes.io/metadata.name` conforme A-03 | Corrigido |
| Direção do Relay ambígua | Não adicionou egress do Sentinel ao Relay e registrou a direção legítima | Corrigido |
| Comentários incompletos | Comentou cada regra com fluxo e ID | Corrigido |
| Efeito aditivo | Exige remoção da permissividade original e inventário das políticas coexistentes | Pendente de validação |
| Comparação ausente | Adicionou tabela original × v3 | Corrigido |
| Contagem incorreta do parecer | Reclassificou a v2 em 6, 7 e 3 | Corrigido |
| Acentuação dos comentários | Comentários YAML em português com acentuação | Corrigido |

Manifestos da v3:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: sentinel-default-deny
  namespace: sentinel-prod
spec:
  # P-04: isolamento explícito de todos os pods
  # do namespace sentinel-prod.
  # A-02: nome definido para esta política.
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
  ingress: []
  egress: []

---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: sentinel-allow-relay-ingress
  namespace: sentinel-prod
spec:
  podSelector:
    matchLabels:
      app: sentinel
  policyTypes:
    - Ingress
  ingress:
    # P-01/P-05: Relay (relay-prod, app=relay)
    # → Sentinel (sentinel-prod, app=sentinel).
    # Porta/protocolo: todos, por omissão de ports.
    # LACUNA L-01: porta não informada;
    # libera todas as portas desta origem.
    # (julgamento) Risco: exposição de quaisquer
    # portas e protocolos do Sentinel aos pods Relay.
    # A-03: identificação do namespace pela
    # label kubernetes.io/metadata.name.
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: relay-prod
          podSelector:
            matchLabels:
              app: relay

---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: sentinel-allow-gateway-ingress
  namespace: sentinel-prod
spec:
  podSelector:
    matchLabels:
      app: sentinel
  policyTypes:
    - Ingress
  ingress:
    # P-01/P-05: API gateway (edge, app=api-gateway)
    # → Sentinel (sentinel-prod, app=sentinel).
    # Porta/protocolo: todos, por omissão de ports.
    # LACUNA L-02: porta não informada;
    # libera todas as portas desta origem.
    # (julgamento) Risco: exposição de quaisquer
    # portas e protocolos do Sentinel ao gateway.
    # A-03: identificação do namespace pela
    # label kubernetes.io/metadata.name.
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: edge
          podSelector:
            matchLabels:
              app: api-gateway

---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: sentinel-allow-forge-egress
  namespace: sentinel-prod
spec:
  podSelector:
    matchLabels:
      app: sentinel
  policyTypes:
    - Egress
  egress:
    # P-02/P-05: Sentinel (sentinel-prod, app=sentinel)
    # → Forge (forge-prod, app=forge), 5432/TCP.
    # (fato) Porta 5432 declarada no mapa.
    # (premissa A-04) Postgres utiliza TCP.
    # (premissa A-03) Label automática do namespace.
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: forge-prod
          podSelector:
            matchLabels:
              app: forge
      ports:
        - protocol: TCP
          port: 5432

---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: sentinel-allow-cerebro-egress
  namespace: sentinel-prod
spec:
  podSelector:
    matchLabels:
      app: sentinel
  policyTypes:
    - Egress
  egress:
    # P-02/P-05: Sentinel (sentinel-prod, app=sentinel)
    # → Cerebro (cerebro-prod, app=cerebro), 9200/TCP.
    # (fato) Porta 9200 declarada no mapa.
    # (premissa A-05) Elasticsearch utiliza TCP.
    # (premissa A-03) Label automática do namespace.
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: cerebro-prod
          podSelector:
            matchLabels:
              app: cerebro
      ports:
        - protocol: TCP
          port: 9200

---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: sentinel-allow-dns-egress
  namespace: sentinel-prod
spec:
  podSelector:
    matchLabels:
      app: sentinel
  policyTypes:
    - Egress
  egress:
    # P-02/P-05: Sentinel (sentinel-prod, app=sentinel)
    # → DNS interno (kube-system, k8s-app=kube-dns).
    # (fato) Porta 53 declarada no mapa.
    # (premissa A-06) DNS utiliza UDP e TCP.
    # (premissa A-03) Label automática do namespace.
    # L-05: validar o caminho DNS efetivo no cluster.
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

Parecer da v3: "aprovada como configuração candidata para homologação, com riscos residuais documentados". A saída registra que L-01 e L-02 permanecem abertos (as duas origens alcançam todas as portas do Sentinel) e que a aplicação em produção depende de validar o CNI, o caminho DNS, as políticas coexistentes e os testes funcionais.

A v3 trouxe resíduos da interface do ChatGPT (imagens de favicon, `&#x20;`, "Copiar checklist"), que não foram transcritos. Também descreveu como "mesmos da revisão anterior" 16 pontos dos quais reclassificou cinco, sem declarar quais.

### Execução de validação com o prompt corrigido

Com os ajustes descritos em [Curadoria](#curadoria), o prompt foi executado em chat novo e temporário, com as três entradas acima e sem nenhuma orientação adicional. A saída reproduziu a v3: seis recursos, protocolos como premissa, ingress de Relay e gateway com origem restrita e sem `ports` (com `LACUNA L-01` e `L-02`), `namespaceSelector` e `podSelector` no mesmo item, nota de que o padrão não autoriza egress ao Relay e tabela de comparação entre original e corrigido. A decomposição do padrão passou a ter oito regras (P-01 a P-08), com Relay e gateway separados, e as dependências de infraestrutura passaram a lacunas individuais (L-03 a L-05).

#### Saída

A saída foi copiada sem edição, exceto a remoção das barras de cópia do ChatGPT e a conversão do bloco interativo de checklist da seção 5 em lista numerada.

````markdown
## 1. Análise do manifesto original

### 1.1. Inventário e diagnóstico

(fato) O manifesto contém uma única `NetworkPolicy`, denominada `sentinel-allow`, no namespace `sentinel-prod`. Sua configuração seleciona todos os pods do namespace e permite tráfego irrestrito de entrada e saída.

(fato) O padrão de segurança foi decomposto nas seguintes regras atômicas, com identificadores atribuídos para rastreabilidade:

| ID   | Requisito do padrão                                                                    |
| ---- | -------------------------------------------------------------------------------------- |
| P-01 | Permitir ingress no Sentinel somente a partir do Relay.                                |
| P-02 | Permitir ingress no Sentinel somente a partir do API gateway.                          |
| P-03 | Permitir egress do Sentinel para Forge, porta 5432.                                    |
| P-04 | Permitir egress do Sentinel para Cerebro, porta 9200.                                  |
| P-05 | Permitir egress do Sentinel para DNS interno, porta 53.                                |
| P-06 | Proibir regras `allow all` em ingress e egress.                                        |
| P-07 | Implementar default-deny explícito para ingress e egress no namespace.                 |
| P-08 | Documentar cada regra de liberação com um comentário que identifique o fluxo legítimo. |

(premissa) A atribuição desses identificadores é registrada como A-01, pois o padrão original não os numera.

### 1.2. Tabela de análise

| Política       | Caminho no YAML                                  | Trecho              | Por que é permissivo                                                                                                 | Regra do padrão violada                        | Rótulo |
| -------------- | ------------------------------------------------ | ------------------- | -------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------- | ------ |
| sentinel-allow | `metadata.namespace`                             | `sentinel-prod`     | Não é permissivo isoladamente; determina o escopo da política.                                                       | Nenhuma                                        | (fato) |
| sentinel-allow | `spec.podSelector`                               | `{}`                | Seleciona todos os pods do namespace. Não é incorreto isoladamente, mas amplia o alcance das permissões irrestritas. | P-01 a P-07, em conjunto com as regras abertas | (fato) |
| sentinel-allow | `spec.policyTypes`                               | `Ingress`, `Egress` | Inclui ambas as direções, mas não restringe os fluxos autorizados. Não há ausência de Egress.                        | Nenhuma isoladamente                           | (fato) |
| sentinel-allow | `spec.ingress[0]`                                | `{}`                | Permite qualquer origem, qualquer porta e qualquer protocolo, incluindo serviços não identificados no mapa.          | P-01, P-02, P-06                               | (fato) |
| sentinel-allow | `spec.egress[0]`                                 | `{}`                | Permite qualquer destino, porta e protocolo, inclusive destinos externos ao cluster.                                 | P-03, P-04, P-05, P-06                         | (fato) |
| sentinel-allow | `spec.ingress[0].from`                           | Ausente             | Não existe restrição por namespace ou label de origem.                                                               | P-01, P-02                                     | (fato) |
| sentinel-allow | `spec.egress[0].to`                              | Ausente             | Não existe restrição por namespace ou label de destino.                                                              | P-03, P-04, P-05                               | (fato) |
| sentinel-allow | `spec.ingress[0].ports` e `spec.egress[0].ports` | Ausentes            | Todas as portas e protocolos permanecem liberados.                                                                   | P-03, P-04, P-05, P-06                         | (fato) |

(fato) As verificações adicionais não identificaram `ipBlock` amplo, `endPort` excessivo, portas nomeadas não previstas, nem união acidental entre `namespaceSelector` e `podSelector`. Esses mecanismos simplesmente não aparecem no manifesto. Também não há ausência de `Egress` em `policyTypes`.

Avaliação geral de risco — (julgamento): crítico. A política original cria uma superfície de comunicação excessivamente ampla, permitindo acessos não previstos entre serviços e conexões de saída sem restrições. Isso aumenta o risco de movimentação lateral e comunicação com destinos não autorizados. Embora a política selecione os pods para isolamento em ambas as direções, suas regras vazias neutralizam as restrições efetivas.

### 1.3. Inventário de fluxos legítimos

(fato) O padrão define cinco relações de comunicação autorizadas. O mapa identifica seus serviços, namespaces, labels e as portas dos destinos de egress.

| Origem                          | Destino                           | Porta         | Protocolo     | Condição |
| ------------------------------- | --------------------------------- | ------------- | ------------- | -------- |
| `relay-prod`, `app=relay`       | `sentinel-prod`, `app=sentinel`   | Não informada | Não informado | L-01     |
| `edge`, `app=api-gateway`       | `sentinel-prod`, `app=sentinel`   | Não informada | Não informado | L-02     |
| `sentinel-prod`, `app=sentinel` | `forge-prod`, `app=forge`         | 5432          | TCP           | A-03     |
| `sentinel-prod`, `app=sentinel` | `cerebro-prod`, `app=cerebro`     | 9200          | TCP           | A-03     |
| `sentinel-prod`, `app=sentinel` | `kube-system`, `k8s-app=kube-dns` | 53            | UDP e TCP     | A-04     |

(premissa) TCP para Postgres e Elasticsearch e UDP/TCP para DNS são as exceções de protocolo expressamente autorizadas pela especificação da tarefa.

(julgamento) Como as portas de ingress do Relay e do API gateway não foram informadas, as liberações correspondentes serão implementadas em políticas separadas, restritas pela identidade da origem, mas sem restrição de porta. Essa é a exceção prevista nas regras de não invenção e constitui uma limitação de segurança documentada.

## 2. Manifestos corrigidos

(fato) Todos os recursos abaixo usam `networking.k8s.io/v1` e permanecem no namespace `sentinel-prod`.

(premissa) Os nomes das novas políticas seguem a convenção descritiva definida em A-05. A identificação de namespaces utiliza a premissa A-02.

(julgamento) A separação entre default-deny e políticas de liberação torna as permissões individuais mais fáceis de auditar. As regras de ingress sem porta ficam isoladas das regras com portas explícitas.

```
# P-07: negação por padrão para todos os pods de sentinel-prod.
# P-06: nenhuma liberação irrestrita de ingress ou egress.
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: sentinel-default-deny
  namespace: sentinel-prod
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
  ingress: []
  egress: []

---
# P-01: restringe as origens de entrada ao Relay.
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: sentinel-allow-relay-ingress
  namespace: sentinel-prod
spec:
  podSelector:
    matchLabels:
      app: sentinel
  policyTypes:
    - Ingress
  ingress:
    # P-01, P-08: Relay (relay-prod/app=relay) →
    # Sentinel (sentinel-prod/app=sentinel).
    # LACUNA L-01: porta não informada; libera todas as portas desta origem.
    # Protocolo não informado; sem ports, todos os protocolos
    # alcançados por NetworkPolicy ficam permitidos nesta origem.
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: relay-prod
          podSelector:
            matchLabels:
              app: relay

---
# P-02: restringe as origens de entrada ao API gateway.
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: sentinel-allow-api-gateway-ingress
  namespace: sentinel-prod
spec:
  podSelector:
    matchLabels:
      app: sentinel
  policyTypes:
    - Ingress
  ingress:
    # P-02, P-08: API gateway (edge/app=api-gateway) →
    # Sentinel (sentinel-prod/app=sentinel).
    # LACUNA L-02: porta não informada; libera todas as portas desta origem.
    # Protocolo não informado; sem ports, todos os protocolos
    # alcançados por NetworkPolicy ficam permitidos nesta origem.
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: edge
          podSelector:
            matchLabels:
              app: api-gateway

---
# P-03: permite somente o destino Forge na saída correspondente.
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: sentinel-allow-forge-egress
  namespace: sentinel-prod
spec:
  podSelector:
    matchLabels:
      app: sentinel
  policyTypes:
    - Egress
  egress:
    # P-03, P-08: Sentinel (sentinel-prod/app=sentinel) →
    # Forge (forge-prod/app=forge), porta 5432/TCP.
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: forge-prod
          podSelector:
            matchLabels:
              app: forge
      ports:
        - protocol: TCP
          port: 5432

---
# P-04: permite somente o destino Cerebro na saída correspondente.
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: sentinel-allow-cerebro-egress
  namespace: sentinel-prod
spec:
  podSelector:
    matchLabels:
      app: sentinel
  policyTypes:
    - Egress
  egress:
    # P-04, P-08: Sentinel (sentinel-prod/app=sentinel) →
    # Cerebro (cerebro-prod/app=cerebro), porta 9200/TCP.
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: cerebro-prod
          podSelector:
            matchLabels:
              app: cerebro
      ports:
        - protocol: TCP
          port: 9200

---
# P-05: permite resolução DNS somente nos pods identificados.
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: sentinel-allow-dns-egress
  namespace: sentinel-prod
spec:
  podSelector:
    matchLabels:
      app: sentinel
  policyTypes:
    - Egress
  egress:
    # P-05, P-08: Sentinel (sentinel-prod/app=sentinel) →
    # DNS interno (kube-system/k8s-app=kube-dns), porta 53/UDP.
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53

    # P-05, P-08: Sentinel (sentinel-prod/app=sentinel) →
    # DNS interno (kube-system/k8s-app=kube-dns), porta 53/TCP.
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: TCP
          port: 53
```

(julgamento) O conjunto estabelece negação por padrão para todos os pods de `sentinel-prod` e libera somente as identidades de origem e destino autorizadas. As duas exceções de ingress continuam mais amplas que o ideal por ausência das portas, e a efetividade geral depende da inexistência de outras políticas aditivas que ampliem as permissões.

## 3. Rastreabilidade

(fato) A tabela relaciona cada requisito atômico do padrão aos recursos e caminhos YAML da versão corrigida.

| ID da regra do padrão | Requisito resumido                         | Recurso (namespace/nome)                           | Trecho que implementa (caminho no YAML)                                            | Situação       | Rótulo       |
| --------------------- | ------------------------------------------ | -------------------------------------------------- | ---------------------------------------------------------------------------------- | -------------- | ------------ |
| P-01                  | Ingress somente do Relay                   | `sentinel-prod/sentinel-allow-relay-ingress`       | `spec.podSelector`, `spec.ingress[0].from[0]`                                      | parcial — L-01 | (julgamento) |
| P-02                  | Ingress somente do API gateway             | `sentinel-prod/sentinel-allow-api-gateway-ingress` | `spec.podSelector`, `spec.ingress[0].from[0]`                                      | parcial — L-02 | (julgamento) |
| P-03                  | Egress para Forge, 5432/TCP                | `sentinel-prod/sentinel-allow-forge-egress`        | `spec.egress[0].to[0]`, `spec.egress[0].ports[0]`                                  | atendida       | (premissa)   |
| P-04                  | Egress para Cerebro, 9200/TCP              | `sentinel-prod/sentinel-allow-cerebro-egress`      | `spec.egress[0].to[0]`, `spec.egress[0].ports[0]`                                  | atendida       | (premissa)   |
| P-05                  | Egress para DNS interno, 53/UDP e 53/TCP   | `sentinel-prod/sentinel-allow-dns-egress`          | `spec.egress[0]`, `spec.egress[1]`                                                 | atendida       | (premissa)   |
| P-06                  | Nenhum allow-all                           | Todos os seis recursos                             | `spec.ingress`, `spec.egress` e respectivos seletores                              | atendida       | (julgamento) |
| P-07                  | Default-deny explícito de ingress e egress | `sentinel-prod/sentinel-default-deny`              | `spec.podSelector`, `spec.policyTypes`, `spec.ingress`, `spec.egress`              | atendida       | (fato)       |
| P-08                  | Comentário em cada regra de liberação      | Cinco recursos de liberação                        | Comentários imediatamente acima de cada item de `spec.ingress[]` e `spec.egress[]` | atendida       | (fato)       |

(julgamento) P-03, P-04 e P-05 estão atendidas segundo a identificação dos pods e os protocolos admitidos nas premissas. A efetividade operacional deve ser confirmada pelos testes da seção 5, especialmente para caminhos de tráfego mediados por Services.

(julgamento) P-01 e P-02 permanecem parcialmente atendidas porque os seletores restringem a origem, mas a ausência das portas impede o menor privilégio completo.

## 4. Lacunas e premissas

### 4.1. Lacunas

- L-01 — Porta de ingress do Relay. (fato) O mapa identifica `relay-prod/app=relay` como origem e `sentinel-prod/app=sentinel` como destino, mas não especifica porta ou protocolo. Afeta P-01. (julgamento) A regra sem `ports` libera todos os protocolos e portas alcançados por NetworkPolicy para essa origem, aumentando a superfície de ataque. É necessário identificar os valores efetivos para restringir a permissão.
- L-02 — Porta de ingress do API gateway. (fato) O mapa identifica `edge/app=api-gateway`, mas não especifica porta ou protocolo para a comunicação com o Sentinel. Afeta P-02. (julgamento) O impacto é equivalente ao de L-01: a origem é restrita, mas todas as portas alcançáveis permanecem permitidas a ela.
- L-03 — Acesso à API do Kubernetes. (fato) Não há no mapa identificação de destino, namespace, labels ou porta para acesso à API. Afeta o levantamento de dependências de P-07. (julgamento) Esse tráfego permanece bloqueado para os pods isolados, salvo permissões de outras políticas ou caminhos não controlados pelo CNI. Aplicações que dependam da API podem falhar.
- L-04 — Health checks. (fato) Não foram identificados fluxos de health checks, suas origens ou portas. Afeta a avaliação operacional de P-01, P-02 e P-07. (julgamento) Verificações sujeitas ao controle de NetworkPolicy podem ser bloqueadas. Verificações originadas no próprio nó podem ter comportamento distinto conforme o ambiente e o CNI.
- L-05 — Telemetria. (fato) O mapa não identifica coletores, portas ou protocolos. Afeta a avaliação operacional de P-07. (julgamento) A coleta ou exportação de métricas, logs e traces pode ser interrompida caso dependa de conexões não autorizadas.
- L-06 — Caminho efetivo do DNS interno. (fato) O mapa identifica os pods `kube-system/k8s-app=kube-dns` e a porta 53, mas não descreve se há Service, tradução de endereços ou outro mecanismo intermediário. Afeta P-05. (julgamento) A regra corresponde à identidade fornecida, porém o funcionamento deve ser confirmado no CNI em uso, particularmente quando o destino é um endereço virtual de Service.
- L-07 — Identificação e implementação do CNI. (fato) O CNI não foi informado. Afeta a garantia operacional de P-01 a P-07. (julgamento) Não é possível confirmar apenas pelos manifestos que as políticas serão aplicadas efetivamente, inclusive para pods com `hostNetwork`.

### 4.2. Premissas

| ID   | Descrição                                                                                                                                    | Consequência caso esteja errada                                                                                                                 |
| ---- | -------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| A-01 | (premissa) Identificadores P-01 a P-08 atribuídos sequencialmente, pois o padrão não possui numeração própria.                               | A rastreabilidade precisará ser adaptada aos identificadores oficiais.                                                                          |
| A-02 | (premissa) A label automática `kubernetes.io/metadata.name` identifica cada namespace informado no mapa.                                     | Em clusters que não atribuem essa label, como versões anteriores ao Kubernetes 1.21, os seletores não funcionarão e os fluxos serão bloqueados. |
| A-03 | (premissa) Forge/Postgres e Cerebro/Elasticsearch utilizam TCP, conforme a exceção autorizada.                                               | Conexões com outro protocolo não serão autorizadas.                                                                                             |
| A-04 | (premissa) O DNS interno utiliza UDP e TCP na porta 53, conforme a exceção autorizada.                                                       | A resolução pode falhar caso dependa de outro protocolo ou caminho de rede.                                                                     |
| A-05 | (premissa) Os nomes `sentinel-default-deny` e `sentinel-allow-*-ingress/egress` são uma convenção nova para facilitar auditoria.             | Os nomes poderão precisar ser ajustados ao padrão operacional do repositório.                                                                   |
| A-06 | (premissa) As liberações específicas serão destinadas aos pods `app=sentinel`, enquanto o default-deny protegerá todos os pods do namespace. | Outros pods do namespace permanecerão isolados; caso existam dependências legítimas, precisarão de documentação e autorização específicas.      |

(fato) O padrão autoriza Relay e API gateway como origens de ingress, não como destinos de egress do Sentinel. Por isso, nenhuma regra de saída do Sentinel para esses serviços foi criada.

(fato) Não foram identificadas divergências explícitas entre as labels ou portas do manifesto original e do mapa, pois o manifesto original não contém essas identificações. Também não há conflito identificado entre o padrão e os fluxos descritos no mapa, recursos não NetworkPolicy a ignorar ou segredos a remover.

### 4.3. Comparação entre manifesto original e corrigido

| Fluxo ou comportamento                                   | Original                            | Corrigido                                                   | Justificativa                                                     |
| -------------------------------------------------------- | ----------------------------------- | ----------------------------------------------------------- | ----------------------------------------------------------------- |
| Ingress do Relay                                         | Qualquer porta, protocolo e origem  | Somente origem Relay; porta e protocolo sem restrição       | (julgamento) Origem restringida conforme P-01; L-01 pendente.     |
| Ingress do API gateway                                   | Qualquer porta, protocolo e origem  | Somente origem API gateway; porta e protocolo sem restrição | (julgamento) Origem restringida conforme P-02; L-02 pendente.     |
| Ingress de outras origens                                | Permitido                           | Bloqueado pela política proposta                            | (fato) Não previsto em P-01 ou P-02.                              |
| Egress para Forge                                        | Qualquer porta/protocolo            | 5432/TCP para pods identificados                            | (premissa) Atende P-03 e A-03.                                    |
| Egress para Cerebro                                      | Qualquer porta/protocolo            | 9200/TCP para pods identificados                            | (premissa) Atende P-04 e A-03.                                    |
| Egress para DNS interno                                  | Qualquer destino, porta e protocolo | 53/UDP e 53/TCP para pods identificados                     | (premissa) Atende P-05 e A-04.                                    |
| Egress para destinos externos não declarados             | Permitido                           | Bloqueado pela política proposta                            | (julgamento) Reduz a possibilidade de comunicação não autorizada. |
| Tráfego de outros pods em `sentinel-prod`                | Irrestrito pelas regras originais   | Negado por padrão                                           | (julgamento) Necessário para P-07.                                |
| Portas e protocolos não previstos nos egress autorizados | Permitidos                          | Bloqueados                                                  | (julgamento) Aplicação do menor privilégio.                       |

(julgamento) Todas as categorias de comunicação anteriormente permitidas e não previstas no padrão foram removidas intencionalmente das permissões da política proposta. A ausência de outras NetworkPolicies permissivas é condição necessária para que o bloqueio seja efetivo.

## 5. Validação antes de aplicar

A validação deve ocorrer inicialmente em um ambiente não produtivo equivalente ao cluster de destino.

Checklist de validação e implantação (9 etapas):

1. Validar sintaxe, schema e dry-run: salvar o bloco YAML em arquivo e executar kubectl apply --dry-run=client e kubectl apply --dry-run=server contra o ambiente de testes. Confirmar que os seis recursos são aceitos.
2. Confirmar a implementação de NetworkPolicy: identificar o CNI, verificar suporte a ingress e egress, avaliar hostNetwork e confirmar a existência das labels automáticas nos namespaces.
3. Inventariar políticas aditivas: listar todas as NetworkPolicies no namespace e verificar quais selecionam app=sentinel ou todos os pods. Identificar políticas que possam manter permissões excessivas.
4. Validar labels e destinos: verificar os pods de Relay, API gateway, Sentinel, Forge, Cerebro e DNS. Confirmar os seletores, serviços intermediários e o percurso real do tráfego DNS.
5. Testar ingress autorizado: em homologação, validar Relay → Sentinel e API gateway → Sentinel. Registrar portas e protocolos efetivamente usados e resolver L-01 e L-02 antes da aprovação final.
6. Testar egress autorizado: verificar Sentinel → Forge em 5432/TCP, Sentinel → Cerebro em 9200/TCP e Sentinel → DNS em 53/UDP e 53/TCP, incluindo resolução DNS real.
7. Testar bloqueios: testar origens não autorizadas, destinos não previstos, outras portas de egress e pods do namespace que não tenham permissões específicas. Confirmar que as conexões são negadas.
8. Planejar aplicação e monitoramento: em produção, aplicar primeiro as liberações específicas verificadas, depois o default-deny e, na mesma janela controlada, retirar a política permissiva original. Monitorar falhas, eventos e observabilidade do CNI.
9. Preparar e testar reversão: manter manifests versionados e uma estratégia de restauração. Em caso de impacto, reverter a alteração de forma controlada, preservando os registros de auditoria e reavaliando os fluxos antes de nova aplicação.

### Comandos de validação

(julgamento) Os seguintes comandos exemplificam verificações de schema, identificação de pods, inspeção de políticas e dry-run. O nome do arquivo é apenas uma referência operacional local, não um recurso ou destino novo do cluster.

```
# Validar o arquivo YAML localmente.
kubectl apply \
  --dry-run=client \
  -f sentinel-networkpolicies.yaml

# Validar junto ao servidor Kubernetes.
kubectl apply \
  --dry-run=server \
  -f sentinel-networkpolicies.yaml

# Inventariar políticas existentes.
kubectl get networkpolicies \
  -n sentinel-prod -o yaml

# Identificar os pods protegidos.
kubectl get pods \
  -n sentinel-prod -l app=sentinel

# Conferir as labels dos namespaces.
kubectl get namespaces --show-labels

# Conferir os pods e seletores esperados.
kubectl get pods -n relay-prod -l app=relay
kubectl get pods -n edge -l app=api-gateway
kubectl get pods -n forge-prod -l app=forge
kubectl get pods -n cerebro-prod -l app=cerebro
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

### Critérios para aprovação

(julgamento) A implantação somente deve ser aprovada quando:

- Os seis manifestos forem aceitos pelo servidor e o CNI demonstrar comportamento compatível com as políticas.
- Todas as conexões legítimas funcionarem e os testes negativos demonstrarem isolamento efetivo.
- L-01 e L-02 tiverem sido avaliadas, com as portas e protocolos documentados e, preferencialmente, incorporados às regras de ingress.
- As dependências L-03 a L-06 tiverem seus impactos verificados, sem introdução de permissões não autorizadas.
- Não existirem outras políticas aditivas que restabeleçam as permissões abertas da configuração original.
- O procedimento de aplicação e reversão tiver sido validado em ambiente não produtivo.

Conclusão — (julgamento): os manifestos corrigidos implementam uma base default-deny explícita, restringem os acessos pelas identidades informadas no mapa e eliminam as liberações genéricas da política original. As duas lacunas de portas de ingress e as dependências operacionais identificadas impedem afirmar que o menor privilégio está totalmente alcançado ou que a implantação pode ocorrer sem impacto. A aprovação final depende das validações descritas.
````

### Testes determinísticos (promptfoo)

O manifesto permissivo, o padrão e o mapa de serviços do CP06 rodam como caso de teste em [`promptfooconfig.yaml`](promptfooconfig.yaml), contra dois provedores. Comando, a partir desta pasta:

```powershell
promptfoo eval -c promptfooconfig.yaml --no-cache --output resultado.json
```

| Item | Valor |
|---|---|
| Versão do promptfoo | 0.125.1 |
| Provedor 1 | OpenAI `gpt-6-sol`, `reasoning_effort: low` |
| Provedor 2 | Anthropic `claude-sonnet-5-5`, `thinking.type: between_tools`, `max_tokens: 16000` |
| Prompt testado | `prompt.md` (arquivo inteiro, com frontmatter) |

Asserts aplicados:

| Requisito | Tipo | Critério |
|---|---|---|
| YAML de NetworkPolicy | `contains` | `kind: NetworkPolicy` |
| `policyTypes` com Ingress e Egress | `javascript` | extrai o bloco ```` ```yaml ````, coleta os itens de `policyTypes` de todos os documentos e exige `Ingress` e `Egress` |
| Sem allow-all | `javascript` | nenhuma linha `- {}` no bloco yaml corrigido |
| Egress para Forge e Cerebro | `contains-all` | `5432` e `9200` |
| Ingress do Relay | `contains` | `app: relay` |
| Toda regra comentada | `javascript` | cada item de `ingress` e `egress` tem um comentário `#` na linha imediatamente acima |
| Latência | `latency` | até **60000 ms** (limite próprio; enunciado: 5000 ms) |
| Custo | `cost` | até **US$ 0,10** (limite próprio; enunciado: US$ 0,01) |

Resultado da última execução (`Results: 2 passed (100%), 0 failed, 0 errors`):

| Modelo | Resultado | Latência | Custo (US$) | Linhas da saída |
|---|---|---|---|---|
| gpt-6-sol | PASS | 20315 ms | 0,0285 | 172 |
| claude-sonnet-5-5 | PASS | 35190 ms | 0,0709 | 221 |

Histórico das execuções até chegar a esse resultado:

| Execução | Configuração | Resultado | Falhas |
|---|---|---|---|
| 1 | limites do enunciado (5 s e US$ 0,01), Sonnet com raciocínio adaptativo, assert `not-contains: '- {}'` sobre a saída inteira | 0/2 | latência e custo nos dois (GPT: 35,6 s e US$ 0,059; Sonnet: 95,2 s e US$ 0,158); no Sonnet, também o `- {}`, que é falso positivo (a tabela de análise cita o manifesto original) |
| 2 | Sonnet com `between_tools`, GPT com `reasoning_effort: low`, `- {}` verificado só no bloco yaml | 0/2 | só latência e custo (GPT: 19,8 s e US$ 0,039; Sonnet: 32,4 s e US$ 0,067) |
| 3 | limites próprios de 60 s e US$ 0,10 | 2/2 | nenhuma |

## Curadoria

**Técnica:** prompt de geração único, sem few-shot, seguido de verificação e refino conduzidos como turnos de conversa, fora do `prompt.md`. Os blocos do prompt mapeiam assim: `<contexto>` fixa a semântica da API `networking.k8s.io/v1` (aditividade, E/OU dos seletores, regra sem `ports`, `ipBlock`, CNI) para que o modelo não dependa de memória sobre elas; o passo 2 do `<metodo>` lista o que verificar no manifesto original; o passo 3 limita os fluxos legítimos ao que o mapa declara; o passo 5 projeta a política (default-deny e uma política de liberação por fluxo); o passo 6 trata dependências de infraestrutura; o passo 7 compara original e corrigido; `<regras>` impõe a rotulagem (fato, premissa, julgamento) e a não invenção; `<verificacao_final>` repassa a saída contra o mapa, o padrão e a semântica da API. Exemplos de manifesto completo ancorariam o modelo em uma forma específica e prejudicariam o reuso.

**Refinos do prompt:** o prompt foi criado por meta-prompting com o Opus 5.5 e refinado com revisão humana. Antes da primeira execução, a conferência apontou que a regra de não invenção impedia a label `kubernetes.io/metadata.name` (o mapa dá o nome do namespace, não a label do namespace), o que bloquearia todos os fluxos entre namespaces. Foi criada a exceção de label, registrada como premissa com consequência (clusters anteriores ao Kubernetes 1.21). Depois da v1 e da verificação, foram aplicados cinco ajustes:

| # | Ajuste | Origem |
|---|---|---|
| 1 | Exceção de protocolo: Postgres e Elasticsearch em TCP e DNS em UDP e TCP podem entrar como premissa com identificador. Não vale para portas | v1 bloqueou os cinco fluxos por falta de protocolo no mapa |
| 2 | Porta de ingress ausente com origem e destino conhecidos: regra com origem restrita e sem `ports`, em política separada, com `LACUNA Lx` e risco rotulado | v1 e v2 mantiveram `ingress: []` |
| 3 | A verificação final passou a tratar fluxo legítimo bloqueado, que poderia ser liberado sem inventar valor, como falha de disponibilidade | A v2 classificou o bloqueio como "parcial" |
| 4 | Serviço autorizado só como origem de ingress não recebe egress, e a seção 4 registra nota de direção | A v1 não registrou a ambiguidade do Relay |
| 5 | A comparação entre original e corrigido ganhou lugar na seção 4, e os comentários do YAML passaram a exigir acentuação | O passo 7 do método não tinha lugar no formato; a v2 trouxe comentários sem acento |

Para compensar o texto acrescentado, foram cortadas a recomendação "não desabilitar mecanismos de segurança", a lista de campos gerados pelo servidor e frases redundantes. O corpo ficou em cerca de 1.975 palavras (contadas por tokens com letra ou número), contra 1.800 do limite inicial.

**Processo de verificação:** a autoverificação (v2) confirmou a v1 em vez de questioná-la. O revisor aplicou a mesma regra de não invenção que causara o problema, classificou o bloqueio de todos os fluxos como parcial e emitiu parecer favorável ao manifesto-base. A falha central só foi vista na conferência humana e corrigida na v3 por direção explícita (protocolo como premissa e ingress sem porta com lacuna sinalizada). O prompt corrigido, em chat novo e sem direção, reproduziu a v3. A lição é que o modelo verifica a saída contra as regras que recebeu, e uma regra excessiva passa pela autoverificação.

**Modelo:** GPT 6 Sol com esforço de raciocínio médio. Na execução de validação, produziu seis recursos corretos (default-deny, dois ingress com origem restrita e sem `ports`, três egress com portas e protocolos), manteve `namespaceSelector` e `podSelector` no mesmo item, registrou a direção do Relay, decompôs o padrão em oito regras, separou as dependências de infraestrutura em lacunas individuais e entregou a tabela original × corrigido. Os pontos fracos foram o ingress sem restrição de porta (risco residual documentado em L-01 e L-02), os comentários de ingress sem o risco rotulado, as seções com subtítulos extras e os comandos de validação não executados.

**Curadoria dos testes (CP08):**

- **O que passou desde a primeira execução:** todos os asserts de conteúdo, nos dois provedores: `kind: NetworkPolicy`, `policyTypes` com `Ingress` e `Egress`, `5432`, `9200`, `app: relay` e comentário `#` acima de cada regra de ingress e de egress. O prompt não precisou de alteração.
- **O que falhou e por quê:**
  - *`- {}` no Sonnet.* Falso positivo do teste. O assert original procurava `- {}` na saída inteira, e a tabela de análise da seção 1 cita o manifesto original (`spec.ingress[0]` e `spec.egress[0]`, ambos `- {}`). O manifesto corrigido não tinha regra allow-all.
  - *Latência e custo, nos dois provedores.* A saída tem 172 a 221 linhas: análise, inventário, YAML com 4 a 6 recursos NetworkPolicy conforme o modelo, rastreabilidade, lacunas e validação. Isso não cabe em 5 s e US$ 0,01 (primeira execução: GPT 35,6 s e US$ 0,059; Sonnet 95,2 s e US$ 0,158).
- **O que foi ajustado nos testes (configuração):**
  - O assert do `- {}` passou a olhar só as linhas do bloco yaml corrigido (a saída tem um único bloco yaml). Ele continua reprovando um `- {}` que aparecesse no manifesto gerado, inclusive com comentário na mesma linha.
  - O Sonnet passou a `thinking.type: between_tools`, e o GPT a `reasoning_effort: low`. Isso reduziu a latência e o custo em cerca de 34% a 66% (GPT: 35,6 para 19,8 s e US$ 0,059 para US$ 0,039; Sonnet: 95,2 para 32,4 s e US$ 0,158 para US$ 0,067), mas sem chegar perto de 5 s e US$ 0,01.
  - Os limites foram elevados para **60 s e US$ 0,10** neste prompt. A saída longa é o que o prompt entrega por definição (análise, manifestos, rastreabilidade, lacunas e validação), e cortá-la para caber nos limites tiraria justamente o conteúdo que o CP06 exige. Foi a opção de menor perda, e os limites do enunciado continuam registrados acima como medição de referência. O uso previsto é uma revisão de segurança, e não uma resposta de plantão.
- **Trade-off de modelo:** o GPT 6 Sol é mais rápido e mais barato neste prompt (cerca de 20 s e US$ 0,03, contra 35 s e US$ 0,07 do Sonnet 5.5). Nenhum dos dois se aproxima do orçamento do enunciado, e o custo e a latência são do tamanho da saída, não de uma escolha de modelo.
- **Limitações dos testes:** os asserts verificam a presença de elementos e a estrutura do YAML por texto e expressão regular. Eles não validam o manifesto contra o servidor de API do Kubernetes nem o isolamento efetivo (`kubectl apply --dry-run=server`), e não avaliam se as lacunas e premissas estão corretas. O teste usa um único manifesto de entrada, e a variação entre execuções não foi medida em série.
