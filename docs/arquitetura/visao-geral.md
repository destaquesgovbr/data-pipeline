# Visão Geral da Arquitetura

O pipeline de dados do DGB combina processamento **event-driven** (near real-time) com **batch** (agendado), usando o PostgreSQL como fonte de verdade central.

## Diagrama Unificado

```mermaid
flowchart TB
    subgraph TRIGGER["Trigger (Batch)"]
        DAGs["Airflow DAGs<br/>~158 DAGs, cada 15min"]
    end

    subgraph COLETA["Coleta"]
        SAPI["Scraper API<br/><i>Cloud Run</i>"]
        GOV["Sites gov.br<br/>+ EBC"]
    end

    subgraph REALTIME["Pipeline Real-Time (Pub/Sub)"]
        T1{{"dgb.news.scraped"}}
        EW["Enrichment Worker<br/><i>Cloud Run</i>"]
        T2{{"dgb.news.enriched"}}
        EAPI["Embeddings API<br/><i>Cloud Run</i>"]
        T3{{"dgb.news.embedded"}}
        TSW["Typesense Sync<br/><i>Cloud Run</i>"]
    end

    subgraph BATCH["Processos Batch"]
        HF_DAG["HuggingFace Sync<br/><i>Airflow, 6AM UTC</i>"]
        FED_DAG["Federation Publish<br/><i>Airflow, cada 10min</i>"]
    end

    subgraph STORES["Armazenamento"]
        PG[("PostgreSQL<br/>Cloud SQL")]
        TS[("Typesense<br/>Compute Engine")]
        HF[("HuggingFace<br/>Datasets")]
    end

    subgraph CONSUMERS["Consumidores"]
        PORTAL["Portal Web<br/><i>Next.js</i>"]
        FEDI["Fediverse<br/><i>Mastodon, Lemmy...</i>"]
        API_PUB["API Pública<br/><i>Datasets</i>"]
    end

    DAGs -->|"POST /scrape"| SAPI
    SAPI -->|HTTP| GOV
    GOV -->|HTML| SAPI
    SAPI -->|INSERT| PG
    SAPI -->|publish| T1

    T1 -->|push| EW
    EW -->|"fetch + UPDATE"| PG
    EW -->|publish| T2

    T2 -->|push| EAPI
    EAPI -->|"fetch + UPDATE"| PG
    EAPI -->|publish| T3

    T2 -->|push| TSW
    T3 -->|push| TSW
    TSW -->|fetch| PG
    TSW -->|upsert| TS

    HF_DAG -->|query| PG
    HF_DAG -->|upload parquet| HF

    FED_DAG -->|query| PG
    FED_DAG -->|enqueue| FED_SVC["Federation Server<br/><i>Cloud Run</i>"]
    FED_SVC -->|deliver| FEDI

    TS --> PORTAL
    HF --> API_PUB
```

## Comparação de Latência

| Caminho | Modo | Latência | Detalhes |
|---------|------|----------|----------|
| Scrape → Portal (busca) | Real-time | **~5 segundos** | Pub/Sub push chain |
| Scrape → Portal (homepage) | Real-time + ISR | ~10 minutos | ISR revalidate=600s |
| Scrape → HuggingFace | Batch | ~24 horas | DAG diária 6AM UTC |
| Scrape → Fediverse | Batch | ~25 minutos | DAG cada 10min + delay |

## Infraestrutura GCP

```mermaid
flowchart LR
    subgraph CR["Cloud Run"]
        S["scraper-api"]
        EW["enrichment-worker"]
        EA["embeddings-api"]
        TW["typesense-sync-worker"]
        FW["federation-web"]
        FF["federation-worker"]
    end

    subgraph COMP["Cloud Composer"]
        AF["Airflow 2.x<br/>~160 DAGs"]
    end

    subgraph PS["Cloud Pub/Sub"]
        T1{{"dgb.news.scraped"}}
        T2{{"dgb.news.enriched"}}
        T3{{"dgb.news.embedded"}}
    end

    SQL[("Cloud SQL<br/>PostgreSQL 15")]
    CE["Compute Engine<br/>Typesense"]
    SM["Secret Manager"]
    AR["Artifact Registry"]

    AF --> S
    T1 --> EW
    T2 --> EA
    T2 --> TW
    T3 --> TW

    S --> SQL
    EW --> SQL
    EA --> SQL
    TW --> SQL
    TW --> CE

    SM -.->|secrets| CR
    AR -.->|images| CR
```

Todos os recursos são provisionados via Terraform no repo [`infra`](https://github.com/destaquesgovbr/infra).

## Serviços Cloud Run

| Serviço | Repo | vCPU | RAM | Min/Max Instâncias | Timeout |
|---------|------|------|-----|---------------------|---------|
| `destaquesgovbr-scraper-api` | scraper | 1 | 1Gi | 0/3 | 900s |
| `destaquesgovbr-enrichment-worker` | data-science | 1 | 1Gi | 0/3 | 900s |
| `destaquesgovbr-embeddings-api` | embeddings | 2 | 4Gi | 0/1 | 600s |
| `destaquesgovbr-typesense-sync-worker` | data-platform | 1 | 512Mi | 0/3 | 300s |
| `destaquesgovbr-federation-web` | activitypub-server | 1 | 512Mi | 0/5 | 300s |
| `destaquesgovbr-federation-worker` | activitypub-server | 1 | 512Mi | 1/2 | — |

Todos usam **scale-to-zero** (min=0), exceto o federation-worker que precisa estar sempre ativo para processar a fila Fedify.

## Repositórios e Responsabilidades

| Repo | Responsabilidade | Serviço Cloud Run | Linguagem |
|------|------------------|-------------------|-----------|
| `scraper` | Coleta de notícias gov.br + EBC | scraper-api | Python (FastAPI) |
| `data-science` | Classificação temática + resumo | enrichment-worker | Python (FastAPI) |
| `embeddings` | Embeddings vetoriais 768-dim | embeddings-api | Python (FastAPI) |
| `data-platform` | Sync Typesense, CLI, managers | typesense-sync-worker | Python (FastAPI) |
| `data-publishing` | Export para HuggingFace | — (DAG Airflow) | Python |
| `activitypub-server` | Federação ActivityPub | federation-web + worker | Node.js (Hono + Fedify) |
| `infra` | Terraform (todos os recursos GCP) | — | HCL |

## Links

- [Pipeline Real-Time](pipeline-realtime.md) — Deep-dive no pipeline event-driven
- [Pipeline Batch](pipeline-batch.md) — Deep-dive nos processos agendados
- [Dados e Armazenamento](dados-e-armazenamento.md) — PostgreSQL, Typesense, HuggingFace
- [Documentação principal do DGB](https://destaquesgovbr.github.io/docs/arquitetura/visao-geral)
