# Playbook de IA operacional

Biblioteca de prompts parametrizáveis, versionada e testada como código, para o time de engenharia da Aegis. Desenvolvida como entrega do desafio técnico "Playbook de IA operacional" da pós-graduação em AIOps e Inteligência Artificial com Engenharia Cloud.

A estrutura segue o template [prompt-registry](https://github.com/fabricioveronez/prompt-registry): cada prompt vive em sua própria pasta, com `prompt.md` (frontmatter + texto do prompt, com placeholders `{{variavel}}`) e `README.md` (mesmo frontmatter + documentação). Convenções em [`CLAUDE.md`](./CLAUDE.md).

## Como usar

1. Navegar até a categoria de interesse.
2. Abrir o `README.md` do prompt para entender objetivo, variáveis esperadas e limitações.
3. Copiar o conteúdo do `prompt.md` e substituir os placeholders `{{nome_variavel}}` pelos valores desejados.

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
