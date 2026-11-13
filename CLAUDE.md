# CLAUDE.md

Convenções deste repositório. Baseadas no template [prompt-registry](https://github.com/fabricioveronez/prompt-registry), adaptadas ao desafio "Playbook de IA operacional".

## Propósito

Biblioteca de prompts parametrizáveis em Markdown. Não há código de aplicação. Cada prompt tem testes promptfoo (`promptfooconfig.yaml`), e o pipeline do GitHub Actions os executa em pull request (ver `docs/gate-de-qualidade.md`).

## Estrutura

```
<categoria>/
  <nome-do-prompt>/
    prompt.md              # frontmatter + texto do prompt (placeholders {{variavel}})
    README.md              # mesmo frontmatter + documentação humana
    promptfooconfig.yaml   # testes do prompt (caminhos relativos a esta pasta)
    dados/                 # entradas dos testes, quando não são inline
    juiz/                  # rubrica do juiz LLM (só em prompts com juiz, como causa-raiz)
```

- Categoria = pasta na raiz, sem aninhar. Hoje existe apenas `devops/`.
- Pasta do prompt em `kebab-case`, nomeada pelo **resultado**, não pela técnica.
- `prompt.md` contém apenas frontmatter e o texto do prompt, sem explicações.
- O texto do prompt é autocontido: não referencia o `README.md` nem outros arquivos.

## Frontmatter

Idêntico em `prompt.md` e `README.md`:

```yaml
---
nome: Nome humano do prompt
descricao: Uma linha descrevendo o objetivo
versao: 1.0.0
tags: [tag1, tag2]
inputs:
  - nome: variavel
    descricao: O que a variável representa
---
```

- `versao` em semver. Todo prompt nasce em `1.0.0`.
- `tags`: 2 a 5 termos em pt-BR.
- `inputs`: um item por placeholder `{{...}}` do corpo. Sem placeholders, `inputs: []`.

## README do prompt

Após o frontmatter: Objetivo, Quando usar, Exemplo de uso, Limitações conhecidas, e, neste repositório, também **Execução** (modelo e output real) e **Curadoria** (técnica escolhida, justificativa e refinamentos).

## Conteúdo

- Português do Brasil, salvo quando o contexto técnico exigir outro idioma.
- Os dados dos cenários são fictícios, mas tratados como produção: não incluir chaves, segredos nem dados reais. O repositório é público.

## Manutenção da documentação

Ao incluir ou alterar um prompt, atualizar na mesma mudança: o `README.md` do prompt, o `README.md` da categoria e o `README.md` da raiz (índices).

## Git

- Commits semânticos de uma linha, com a categoria como escopo (ex.: `feat(devops): adiciona prompt de triagem de pods`).

## Testes e pipeline

- O workflow `.github/workflows/promptfoo.yml` chama modelos pagos com as chaves do repositório. Ele roda só em pull request (e sob disparo manual). **Não adicionar gatilho de `push` nem de agenda (`cron`)**: cada execução completa custa cerca de US$ 1.
- Alterar `prompt.md`, `promptfooconfig.yaml`, `dados/` ou `juiz/` num pull request executa os testes do prompt alterado. Os demais jobs terminam sem chamar modelo.
- Ao alterar um prompt, reexecutar os testes dele e atualizar a seção **Execução** do `README.md` do prompt.
- Os arquivos `execucao.txt` e `resultado.json` gerados nas execuções locais não são versionados.
