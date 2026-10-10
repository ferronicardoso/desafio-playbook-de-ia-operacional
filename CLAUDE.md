# CLAUDE.md

Convenções deste repositório. Baseadas no template [prompt-registry](https://github.com/fabricioveronez/prompt-registry), adaptadas ao desafio "Playbook de IA operacional".

## Propósito

Biblioteca de prompts parametrizáveis em Markdown. Não há código de aplicação. Os testes dos prompts (promptfoo) e o pipeline (GitHub Actions) são adicionados nos Checkpoints 08 a 10.

## Estrutura

```
<categoria>/
  <nome-do-prompt>/
    prompt.md              # frontmatter + texto do prompt (placeholders {{variavel}})
    README.md              # mesmo frontmatter + documentação humana
    promptfooconfig.yaml   # testes do prompt (a partir do Checkpoint 08)
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
