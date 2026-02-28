# Plano: Repo `data-pipeline` com Documentação MkDocs

## Contexto

O pipeline de dados do DGB está distribuído em 6 repos (scraper, data-science, embeddings, data-platform, data-publishing, activitypub-server). Recentemente migramos o pipeline batch (Airflow) para event-driven (Pub/Sub + Cloud Run), resultando em latência de ~5s. Falta uma documentação unificada que mostre o pipeline como sistema coerente — tanto o modo real-time quanto o batch.

O novo repo `data-pipeline` será criado para abrigar essa documentação (e futuro código/ferramentas).

## Estrutura do Repo

```
data-pipeline/
├── .github/workflows/docs.yml
├── .gitignore
├── CLAUDE.md
├── README.md
├── pyproject.toml
├── mkdocs.yml
└── docs/
    ├── index.md
    ├── stylesheets/extra.css
    ├── arquitetura/
    │   ├── visao-geral.md            # Diagrama unificado batch + real-time
    │   ├── pipeline-realtime.md      # Deep-dive Pub/Sub
    │   ├── pipeline-batch.md         # Deep-dive Airflow DAGs
    │   └── dados-e-armazenamento.md  # PostgreSQL, Typesense, HuggingFace
    ├── componentes/
    │   ├── scraper.md
    │   ├── enrichment.md
    │   ├── embeddings.md
    │   ├── typesense-sync.md
    │   ├── data-publishing.md
    │   └── federation.md
    ├── operacoes/
    │   ├── monitoramento.md
    │   ├── troubleshooting.md
    │   └── runbooks.md
    └── desenvolvimento/
        ├── setup-local.md
        ├── contribuindo.md
        └── adicionando-componente.md
```

## Configuração

### `pyproject.toml`
Mesma estrutura do repo `docs`: Poetry, Python ^3.11, mkdocs ^1.6.0, mkdocs-material ^9.6.0. Sem blog plugin.

### `mkdocs.yml`
- **Theme**: Material, pt-BR, paleta **indigo/blue** (diferencia do `docs` que é green)
- **Features**: navigation.tabs, navigation.sections, navigation.top, navigation.expand, search, content.code.copy, content.tabs.link
- **Extensions**: admonition, pymdownx.superfences (mermaid), pymdownx.highlight, pymdownx.tabbed, pymdownx.details, tables, toc
- **Nav**: 4 seções (Arquitetura, Componentes, Operações, Desenvolvimento)

### `.github/workflows/docs.yml`
Cópia do workflow do repo `docs` (Poetry + mkdocs build + GitHub Pages deploy).

## Conteúdo das Páginas

### `index.md` — Homepage
- O que é o pipeline DGB (158+ órgãos, AI enrichment, 4 saídas)
- Diagrama Mermaid simplificado end-to-end
- Tabela: Real-time (~5s) vs Batch (15min/10min/diário)

### `arquitetura/visao-geral.md` — Página Central
- Grande diagrama Mermaid unificado batch + real-time
- Infra diagram: Cloud Run, Composer, Cloud SQL, Pub/Sub, Typesense
- Tabela de repos: nome, propósito, serviço Cloud Run, linguagem

### `arquitetura/pipeline-realtime.md`
- Baseado em `pubsub-workers.md` existente (referência cruzada)
- Topics/subscriptions, timeline de latência, idempotência, DLQs

### `arquitetura/pipeline-batch.md`
- 3 pipelines: Scraper trigger (15min), HF sync (6AM), Federation (10min)
- Diagramas de sequência por pipeline

### `arquitetura/dados-e-armazenamento.md`
- PostgreSQL (fonte de verdade), Typesense (busca), HuggingFace (dados abertos)
- Ciclo de vida de um artigo pelos 3 stores

### Componentes (6 páginas, template padrão)
Cada página: O que faz → Como funciona → Onde mora → Conexões (Mermaid) → Configuração → Specs → Idempotência

| Componente | Repo | Input → Output |
|---|---|---|
| Scraper | `scraper` | gov.br → PG + `dgb.news.scraped` |
| Enrichment | `data-science` | `dgb.news.scraped` → PG + `dgb.news.enriched` |
| Embeddings | `embeddings` | `dgb.news.enriched` → PG + `dgb.news.embedded` |
| Typesense Sync | `data-platform` | enriched + embedded → Typesense |
| Data Publishing | `data-publishing` | PG → HuggingFace |
| Federation | `activitypub-server` | PG → Fediverse |

### `operacoes/monitoramento.md`
Métricas Cloud Run, Pub/Sub, Composer, Cloud SQL. Logging com `trace_id`. Alertas recomendados.

### `operacoes/troubleshooting.md`
Por sintoma: artigos não aparecem, enrichment falhou, embeddings não gerados, Typesense desatualizado, HF não atualizado, federation não publica. Inclui bugs reais encontrados na sessão de validação.

### `operacoes/runbooks.md`
Procedimentos: re-scrape, forçar enriquecimento, rebuild Typesense, deploy emergência, limpar DLQ.

### `desenvolvimento/setup-local.md`
Pré-requisitos + setup por repo (tabs). Cloud SQL Proxy, env vars.

### `desenvolvimento/contribuindo.md`
Conventional Commits, branch naming, PR process.

### `desenvolvimento/adicionando-componente.md`
Guia passo-a-passo + template FastAPI `/process` + checklist.

## Fontes de Conteúdo

| Fonte | Uso |
|---|---|
| `docs/arquitetura/pubsub-workers.md` | Pipeline real-time + componentes |
| `docs/arquitetura/fluxo-de-dados.md` | Pipeline batch |
| `docs/blog/posts/2026-02-26-desmembrando-monolito.md` | Contexto, decisões |
| CLAUDE.md de cada repo | Specs, env vars, conexões |
| Sessão de validação (28/fev) | Troubleshooting (7 bugs reais) |

## Sequência de Implementação

1. Scaffold: diretório, pyproject.toml, mkdocs.yml, .gitignore, CLAUDE.md, README.md
2. `poetry install` + verificar `mkdocs serve`
3. Arquitetura: index.md, visao-geral.md, pipeline-realtime.md, pipeline-batch.md, dados-e-armazenamento.md
4. Componentes: 6 páginas
5. Operações: monitoramento.md, troubleshooting.md, runbooks.md
6. Desenvolvimento: setup-local.md, contribuindo.md, adicionando-componente.md
7. CI/CD: .github/workflows/docs.yml
8. Git init + push para `destaquesgovbr/data-pipeline`

## Verificação

1. `poetry run mkdocs serve` — site renderiza sem erros
2. Diagramas Mermaid renderizam corretamente
3. Links internos e externos funcionam
4. `poetry run mkdocs build` — build sem warnings
