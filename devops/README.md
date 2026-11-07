# DevOps

Prompts voltados a **operação e confiabilidade da plataforma Aegis**: triagem de incidentes, análise de causa-raiz, decisões de arquitetura sob carga, migração de pipelines e endurecimento de configuração de infraestrutura.

## Escopo

Entram aqui prompts relacionados a:

- Triagem de saúde de workloads em Kubernetes.
- Padronização de notas de triagem de alertas.
- Análise de causa-raiz a partir de configuração, métricas e logs.
- Apoio a decisões de arquitetura (ex.: backpressure em barramento de eventos).
- Planejamento de migrações de pipelines de dados.
- Endurecimento de políticas de rede (NetworkPolicy).

## Fora de escopo

- Escrita de código de aplicação.
- Conteúdo educacional ou de divulgação sobre os temas acima.

## Prompts

- [triagem-de-pods](./triagem-de-pods/) — Triagem rápida da saúde dos pods de um namespace a partir de um snapshot de kubectl colado pelo plantonista, com causa provável e próxima ação.
- [nota-de-triagem](./nota-de-triagem/) — Converte um alerta cru de observabilidade em uma nota de triagem padronizada de cinco linhas para a passagem de turno SRE.
- [causa-raiz](./causa-raiz/) — Analisa um pacote de artefatos de uma degradação em sistemas de busca e indexação sobre Elasticsearch/JVM e produz um diagnóstico de causa-raiz rastreável, com nível de confiança e ações proporcionais.
- [backpressure-no-relay](./backpressure-no-relay/) — Compara estratégias de backpressure para um barramento de eventos a partir do estado atual e das restrições informadas, com aritmética explícita, eliminação por restrições inegociáveis e recomendação justificada.
- [diagnostico-do-forge](./diagnostico-do-forge/) — Produz o diagnóstico estruturado do estado atual de um pipeline de dados em lote antes da migração para arquitetura orientada a eventos.
- [passos-da-migracao-do-forge](./passos-da-migracao-do-forge/) — Converte o diagnóstico de um pipeline em lote em uma sequência ordenada de etapas incrementais de migração para arquitetura orientada a eventos.
- [plano-de-migracao-do-forge](./plano-de-migracao-do-forge/) — Transforma a sequência de etapas de migração de um pipeline para arquitetura orientada a eventos em plano executável, verificável e reversível.
- [networkpolicy-sentinel](./networkpolicy-sentinel/) — Analisa uma NetworkPolicy permissiva e gera manifestos endurecidos e rastreáveis a um padrão de segurança, usando somente identificadores do mapa de serviços.
