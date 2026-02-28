# CLAUDE.md

## Project Overview

Documentação e ferramentas do pipeline de dados do **DestaquesGovBr** — plataforma que agrega notícias de 158+ órgãos do governo federal brasileiro com busca semântica e enriquecimento via IA.

Este repo contém a documentação técnica unificada do pipeline (MkDocs + Material) e futuramente abrigará ferramentas e scripts compartilhados.

## Build & Development Commands

```bash
poetry install                  # Instala dependências
poetry run mkdocs serve         # Dev server com hot-reload (localhost:8000)
poetry run mkdocs build         # Build estático em site/
```

## Architecture

- **Build**: MkDocs + Material theme, gerenciado via Poetry (Python 3.11+)
- **Config**: `mkdocs.yml`
- **Content**: `docs/` (Markdown)
- **Deploy**: GitHub Actions → GitHub Pages (automático em push para main)

## Conventions

- **Language**: Brazilian Portuguese (pt-BR)
- **Commit messages**: Conventional Commits (`feat:`, `fix:`, `docs:`, `chore:`)
- **Navigation**: páginas novas devem ser adicionadas ao `nav:` em `mkdocs.yml`
- **Diagrams**: Mermaid (não imagens estáticas)
- **Links para docs principal**: `https://destaquesgovbr.github.io/docs/`

## Related Repos

| Repo | Papel no Pipeline |
|------|-------------------|
| `scraper` | Coleta de notícias (Cloud Run API + Airflow DAGs) |
| `data-science` | Enriquecimento LLM (classificação + resumo) |
| `embeddings` | Geração de embeddings (sentence-transformer) |
| `data-platform` | Typesense sync worker, PostgresManager, CLI |
| `data-publishing` | Sync PostgreSQL → HuggingFace |
| `activitypub-server` | Federação ActivityPub → Fediverse |
| `infra` | Terraform (Cloud Run, Pub/Sub, IAM) |
