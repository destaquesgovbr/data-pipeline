# Data Pipeline — DestaquesGovBr

Documentação técnica unificada do pipeline de dados do [DestaquesGovBr](https://github.com/destaquesgovbr).

## Documentação

Acesse em: **https://destaquesgovbr.github.io/data-pipeline/**

### Desenvolvimento local

```bash
poetry install
poetry run mkdocs serve
```

O site estará disponível em `http://localhost:8000`.

## Sobre o Pipeline

O pipeline processa notícias de 158+ órgãos do governo federal brasileiro, desde a coleta até a disponibilização em múltiplos canais:

- **Portal** — busca full-text e semântica via Typesense
- **HuggingFace** — datasets abertos em formato Parquet
- **Fediverse** — publicação via ActivityPub (Mastodon, Lemmy, etc.)

Dois modos de operação:

| Modo | Latência | Tecnologia |
|------|----------|------------|
| Real-time | ~5 segundos | Pub/Sub + Cloud Run |
| Batch | 15min a 24h | Airflow (Cloud Composer) |
