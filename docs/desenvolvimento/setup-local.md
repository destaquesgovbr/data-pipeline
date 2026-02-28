# Setup de Desenvolvimento Local

## Pré-requisitos

| Ferramenta | Versão | Uso |
|-----------|--------|-----|
| Python | 3.11+ | Repos Python (scraper, data-science, embeddings, data-platform, data-publishing) |
| Poetry | 1.7+ | Gerenciamento de dependências |
| Node.js | 22+ | activitypub-server |
| Docker | 24+ | Containers locais |
| gcloud CLI | latest | Acesso ao GCP |
| Cloud SQL Proxy | latest | Acesso ao PostgreSQL de produção |

## Acesso ao PostgreSQL

O PostgreSQL roda no Cloud SQL (IP privado). Para acesso local, use o Cloud SQL Proxy:

```bash
# Instalar (macOS)
brew install cloud-sql-proxy

# Conectar
cloud-sql-proxy inspire-7-finep:southamerica-east1:destaquesgovbr-db \
  --port 5432
```

A connection string local fica:
```
postgresql://USER:PASSWORD@localhost:5432/govbrnews
```

## Setup por Repo

=== "Scraper"

    ```bash
    cd scraper
    poetry install

    # Variáveis de ambiente
    export DATABASE_URL="postgresql://..."
    export PUBSUB_TOPIC_NEWS_SCRAPED="projects/inspire-7-finep/topics/dgb.news.scraped"

    # Rodar API localmente
    poetry run uvicorn govbr_scraper.api:app --reload --port 8001

    # Testar
    curl -X POST http://localhost:8001/scrape/agencies \
      -H "Content-Type: application/json" \
      -d '{"agencies": ["mec"], "start_date": "2026-02-28"}'
    ```

=== "Data Science"

    ```bash
    cd data-science
    poetry install

    # Variáveis
    export DATABASE_URL="postgresql://..."
    export AWS_BEDROCK_CONNECTION_URI="aws://KEY:SECRET@/?region_name=us-east-1"
    export PUBSUB_TOPIC_NEWS_ENRICHED="projects/inspire-7-finep/topics/dgb.news.enriched"

    # Rodar worker localmente
    poetry run uvicorn news_enrichment.worker.app:app --reload --port 8002
    ```

=== "Embeddings"

    ```bash
    cd embeddings
    poetry install  # Demora: baixa modelo ML (~400MB)

    # Variáveis
    export DATABASE_URL="postgresql://..."
    export PUBSUB_TOPIC_NEWS_EMBEDDED="projects/inspire-7-finep/topics/dgb.news.embedded"
    export EMBEDDINGS_API_KEY="dev-key"

    # Rodar API localmente
    poetry run uvicorn embeddings_api.main:app --reload --port 8003
    ```

=== "Data Platform"

    ```bash
    cd data-platform
    poetry install

    # Variáveis
    export DATABASE_URL="postgresql://..."
    export TYPESENSE_WRITE_CONN='{"host":"localhost","port":"8108","apiKey":"KEY","protocol":"http"}'

    # CLI
    poetry run data-platform sync-typesense --start-date 2026-02-01

    # Worker
    poetry run uvicorn data_platform.workers.typesense_sync.app:app --reload --port 8004
    ```

=== "Data Publishing"

    ```bash
    cd data-publishing
    poetry install

    # Requer Astro CLI para rodar DAGs localmente
    # Veja: https://docs.astronomer.io/astro/cli/install-cli
    astro dev start
    ```

=== "ActivityPub Server"

    ```bash
    cd activitypub-server
    npm install

    # Variáveis
    export DATABASE_URL="postgresql://..."
    export FEDERATION_TOKEN="dev-token"

    # Rodar
    npm run dev
    ```

## Typesense Local

Para desenvolvimento com busca, rode um Typesense local via Docker:

```bash
docker run -d --name typesense \
  -p 8108:8108 \
  -v /tmp/typesense-data:/data \
  typesense/typesense:27.1 \
  --data-dir /data \
  --api-key=dev_api_key \
  --enable-cors
```

Veja [Typesense Local (docs principal)](https://destaquesgovbr.github.io/docs/modulos/typesense-local) para mais detalhes.

## Links

- [Setup Backend (docs principal)](https://destaquesgovbr.github.io/docs/onboarding/setup-backend) — Guia completo de onboarding
- [Airflow DAGs (docs principal)](https://destaquesgovbr.github.io/docs/workflows/airflow-dags) — Composer e Astro CLI
