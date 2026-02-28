# Contribuindo

## Convenções de Código

### Commits

Usamos [Conventional Commits](https://www.conventionalcommits.org/):

| Prefixo | Uso |
|---------|-----|
| `feat:` | Nova funcionalidade |
| `fix:` | Correção de bug |
| `docs:` | Documentação |
| `chore:` | Manutenção (deps, CI, etc.) |
| `refactor:` | Refatoração sem mudança de comportamento |
| `test:` | Testes |

Exemplos:
```
feat: add enrichment worker Pub/Sub handler
fix: URL-decode AWS credentials from Airflow URI
docs: add pipeline real-time architecture page
```

### Branches

| Padrão | Uso |
|--------|-----|
| `feat/descricao` | Nova feature |
| `fix/descricao` | Bug fix |
| `docs/descricao` | Documentação |
| `chore/descricao` | Manutenção |

### Pull Requests

1. Crie uma branch a partir de `main`
2. Faça commits seguindo a convenção
3. Abra PR com descrição clara do que mudou e por quê
4. Aguarde CI passar (build, testes, lint)
5. Merge via squash ou merge commit

## Documentação

Ao modificar o pipeline (novo componente, mudança de fluxo, etc.), atualize esta documentação:

1. Edite/crie páginas em `docs/`
2. Adicione ao `nav:` em `mkdocs.yml` se for página nova
3. Use Mermaid para diagramas (não imagens)
4. Rode `poetry run mkdocs serve` para verificar
5. Inclua link para a [docs principal](https://destaquesgovbr.github.io/docs/) quando relevante

## Testes

Cada repo tem sua própria estratégia de testes:

| Repo | Framework | Comando |
|------|-----------|---------|
| scraper | pytest | `poetry run pytest` |
| data-science | pytest | `poetry run pytest` |
| embeddings | pytest | `poetry run pytest` |
| data-platform | pytest | `poetry run pytest` |
| activitypub-server | vitest | `npm test` |

## CI/CD

### Deploy de Cloud Run

Workers usam o [reusable workflow](https://github.com/destaquesgovbr/reusable-workflows) `cloud-run-deploy.yml`:

- Build Docker image
- Push para Artifact Registry
- Deploy no Cloud Run
- Trigger automático em push para `main` com path filters

### Deploy de DAGs

DAGs Airflow usam o reusable workflow `composer-deploy-dags.yml`:

- `gsutil rsync` do diretório `dags/` para o bucket do Composer
- Cada repo usa um subdiretório próprio

### Deploy desta documentação

Push para `main` → GitHub Actions → `mkdocs build` → GitHub Pages.
