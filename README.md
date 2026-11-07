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
