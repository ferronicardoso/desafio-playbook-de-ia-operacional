# Playbook de IA operacional

Biblioteca de prompts parametrizáveis, versionada e testada como código, para o time de engenharia da Aegis. Desenvolvida como entrega do desafio técnico "Playbook de IA operacional" da pós-graduação em AIOps e Inteligência Artificial com Engenharia Cloud.

A estrutura segue o template [prompt-registry](https://github.com/fabricioveronez/prompt-registry): cada prompt vive em sua própria pasta, com `prompt.md` (frontmatter + texto do prompt, com placeholders `{{variavel}}`) e `README.md` (mesmo frontmatter + documentação). Convenções em [`CLAUDE.md`](./CLAUDE.md).

## Como usar

1. Navegar até a categoria de interesse.
2. Abrir o `README.md` do prompt para entender objetivo, variáveis esperadas e limitações.
3. Copiar o conteúdo do `prompt.md` e substituir os placeholders `{{nome_variavel}}` pelos valores desejados.

## Como o playbook mapeia o template

O repositório foi montado a partir das convenções do `prompt-registry`, sem fork: o template era uma sugestão, e a estrutura foi reproduzida e adaptada neste repositório.

| Convenção do template | Aplicação neste playbook |
|---|---|
| Categoria como pasta na raiz, em kebab-case, sem aninhar | Uma única categoria, `devops/`, porque todos os prompts tratam de operação e confiabilidade da plataforma. As demais categorias do template não foram criadas por não terem prompts. |
| Um prompt por pasta, nomeada pelo resultado | Oito pastas, como `triagem-de-pods` e `networkpolicy-sentinel`. Nenhuma é nomeada pela técnica usada. |
| `prompt.md` com frontmatter e texto puro, parâmetros em `{{variavel}}` | Cada `prompt.md` traz só o frontmatter e o texto do prompt, autocontido. |
| `README.md` com o mesmo frontmatter e documentação humana | Frontmatter idêntico nos dois arquivos, com objetivo, quando usar, exemplo e limitações. |
| `inputs` com `nome` e `descricao` | Um item por placeholder do corpo, incluindo prompts com vários parâmetros (`triagem-de-pods` tem sete). |
| `versao` em semver | Todos nascem em `1.0.0`. |
| Commits semânticos com a categoria como escopo | Por exemplo, `feat(devops): adiciona prompt de backpressure`. |
| Índices nos READMEs | Atualizados na mesma mudança, na raiz e em `devops/`. |

**Mapeamento dos checkpoints:**

| Checkpoint | Prompt(s) |
|---|---|
| CP01 | `triagem-de-pods` |
| CP02 | `nota-de-triagem` |
| CP03 | `causa-raiz` |
| CP04 | `backpressure-no-relay` |
| CP05 | `diagnostico-do-forge`, `passos-da-migracao-do-forge`, `plano-de-migracao-do-forge`: cadeia de três elos, em que a saída de cada um é a entrada do seguinte |
| CP06 | `networkpolicy-sentinel` |
| CP07 | organização do repositório: a estrutura em `devops/`, os índices e as convenções do [`CLAUDE.md`](./CLAUDE.md) |
| CP08 | testes promptfoo de `nota-de-triagem`, `triagem-de-pods` e `networkpolicy-sentinel` (saída estruturada) |
| CP09 | gate de qualidade com juiz LLM em `causa-raiz`: rubrica de quatro critérios, calibração contra a pontuação humana e `promptfooconfig.yaml` |
| CP10 | testes dos oito prompts, workflow do GitHub Actions e justificativa do gate em [`docs/gate-de-qualidade.md`](./docs/gate-de-qualidade.md) |

**Extensões em relação ao template:** os `README.md` ganham as seções **Execução** (modelo, entradas e saída real, com as iterações quando houve refino) e **Curadoria** (técnica, justificativa e refinos). O `CLAUDE.md` registra essas convenções e limita as tags a 2 a 5 por prompt.

**Exemplo completo:** [`devops/networkpolicy-sentinel/`](./devops/networkpolicy-sentinel/) é o mais completo, com `prompt.md` de três placeholders, `README.md` com as iterações v1, v2 e v3 e a execução de validação.

## Testes e pipeline

Cada pasta de prompt tem um `promptfooconfig.yaml` com os testes do prompt, executados contra OpenAI (`gpt-6-sol`) e Anthropic (`claude-sonnet-5-5`). A documentação dos testes, com os resultados e a curadoria, fica na seção **Execução** do `README.md` de cada prompt.

- **Executar um prompt localmente:** a partir da pasta do prompt, `promptfoo eval -c promptfooconfig.yaml`, com `OPENAI_API_KEY` e `ANTHROPIC_API_KEY` no ambiente. A versão usada é a 0.125.1.
- **Pipeline:** [`.github/workflows/promptfoo.yml`](./.github/workflows/promptfoo.yml) roda em pull request, só para os prompts alterados, e sob disparo manual, para a suíte completa. Não há gatilho de push nem agenda, porque cada execução chama modelos pagos (a suíte completa custa cerca de US$ 1). Qualquer teste reprovado derruba o job.
- **Decisões do gate, alternativas comparadas e evidências das execuções:** [`docs/gate-de-qualidade.md`](./docs/gate-de-qualidade.md).
- **Chaves dos provedores:** secrets do repositório (`OPENAI_API_KEY` e `ANTHROPIC_API_KEY`). Nunca entram no repositório.

## Categorias

### [DevOps](./devops/)

Operação e confiabilidade da plataforma: triagem de incidentes, causa-raiz, decisões de arquitetura, migração de pipelines e endurecimento de configuração.

- [triagem-de-pods](./devops/triagem-de-pods/) — Triagem rápida da saúde dos pods de um namespace a partir de um snapshot de kubectl colado pelo plantonista, com causa provável e próxima ação.
- [nota-de-triagem](./devops/nota-de-triagem/) — Converte um alerta cru de observabilidade em uma nota de triagem padronizada de cinco linhas para a passagem de turno SRE.
- [causa-raiz](./devops/causa-raiz/) — Analisa um pacote de artefatos de uma degradação em sistemas de busca e indexação sobre Elasticsearch/JVM e produz um diagnóstico de causa-raiz rastreável, com nível de confiança e ações proporcionais.
- [backpressure-no-relay](./devops/backpressure-no-relay/) — Compara estratégias de backpressure para um barramento de eventos a partir do estado atual e das restrições informadas, com aritmética explícita, eliminação por restrições inegociáveis e recomendação justificada.
- [diagnostico-do-forge](./devops/diagnostico-do-forge/) — Produz o diagnóstico estruturado do estado atual de um pipeline de dados em lote antes da migração para arquitetura orientada a eventos.
- [passos-da-migracao-do-forge](./devops/passos-da-migracao-do-forge/) — Converte o diagnóstico de um pipeline em lote em uma sequência ordenada de etapas incrementais de migração para arquitetura orientada a eventos.
- [plano-de-migracao-do-forge](./devops/plano-de-migracao-do-forge/) — Transforma a sequência de etapas de migração de um pipeline para arquitetura orientada a eventos em plano executável, verificável e reversível.
- [networkpolicy-sentinel](./devops/networkpolicy-sentinel/) — Analisa uma NetworkPolicy permissiva e gera manifestos endurecidos e rastreáveis a um padrão de segurança, usando somente identificadores do mapa de serviços.
