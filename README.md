# Triagem agentica de issues

Este repositorio mantem a logica central de um GitHub Agentic Workflow que
analisa issues abertas ou reabertas, procura duplicidades, sugere classificacao
e prioridade e publica um relatorio de triagem em portugues.

## Arquitetura

Eventos de issue nao atravessam repositorios. Cada repositorio monitorado recebe
somente um relay pequeno, que reage ao evento e chama o reusable workflow
`.github/workflows/issue-triage.lock.yml` deste repositorio. Toda a logica de
analise permanece centralizada aqui.

O reusable workflow executa no contexto do repositorio chamador. O agente recebe
acesso somente de leitura, enquanto labels e comentarios sao aplicados por
`safe-outputs` usando o `GITHUB_TOKEN` do repositorio de origem.

## Pre-requisitos

- GitHub CLI autenticado com acesso aos repositorios e permissao de workflow.
- Extensao `gh-aw`: `gh extension install github/gh-aw`.
- GitHub Actions habilitado nos repositorios de destino.
- GitHub Copilot com cobranca centralizada na organizacao para usar
  `copilot-requests: write`.
- Em Settings > Actions > General deste repositorio, acesso aos reusable
  workflows liberado para os repositorios da organizacao.
- Secret `ORG_WORKFLOW_TOKEN` neste repositorio, usando um fine-grained PAT ou
  token equivalente com Contents, Pull requests e Workflows: Read and write nos
  destinos. Um PAT classic precisa dos escopos `repo` e `workflow`.
- Labels desejadas criadas nos repositorios de destino. O agente ignora labels
  ausentes, mas recomenda-se padroniza-las antes do rollout.

Labels configuradas: `bug`, `enhancement`, `question`, `documentation`,
`needs-info`, `priority/p0`, `priority/p1`, `priority/p2`, `duplicate`, `invalid`
e `spam`.

## Preparacao local

```bash
gh aw doctor
gh aw compile --strict
gh aw validate --strict
```

Versione tanto `.github/workflows/issue-triage.md` quanto o arquivo gerado
`.github/workflows/issue-triage.lock.yml`.

## Relay

O arquivo [templates/issue-triage-relay.yml](templates/issue-triage-relay.yml)
mostra o unico workflow que precisa existir nos repositorios de origem. Substitua
`ORGANIZATION/issue-triage` pelo slug deste repositorio central e prefira uma tag
ou SHA no lugar de `main` para rollout de producao.

## Instalacao nos destinos

O workflow `Install issue triage relay` recebe nomes de repositorios, cria ou
atualiza `.github/workflows/issue-triage.yml` e abre um pull request em cada um.
Execute primeiro com `dry_run: true` pela aba Actions. Depois execute com
`dry_run: false` para criar os PRs.

Tambem e possivel disparar pelo CLI:

```bash
gh workflow run install-triage.yml \
  -f repositories='api-service,web-app' \
  -f central_ref='v1' \
  -f dry_run=false
```

O input aceita nomes separados por virgula, espaco ou quebra de linha. Use `*`
sozinho para selecionar todos os repositorios nao arquivados, que nao sejam forks
e tenham Issues habilitadas. Por seguranca, todos sao resolvidos dentro da mesma
organizacao deste repositorio. Repositorios sem Actions devem ser omitidos da
lista explicita.

Cada PR precisa ser revisado e mesclado para ativar a triagem. Novos repositorios
tambem precisam passar pelo instalador.

## Operacao

```bash
gh run list --repo ORGANIZACAO/REPOSITORIO --workflow issue-triage.yml
gh aw logs issue-triage --repo ORGANIZACAO/issue-triage
gh aw health issue-triage --repo ORGANIZACAO/issue-triage --days 30
```

As definicoes foram baseadas nos exemplos oficiais de
[AI issue triage](https://github.github.com/gh-aw/gallery/ai-issue-triage/) e
[cross-repository issue tracking](https://github.github.com/gh-aw/gallery/multi-repo/issue-tracking/).