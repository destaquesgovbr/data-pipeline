# Runbooks Operacionais

Procedimentos passo-a-passo para operações comuns do pipeline.

## Re-scrape de artigo específico

Quando um artigo precisa ser re-coletado (conteúdo atualizado, erro no parse).

```bash
# 1. Identificar o órgão
AGENCY="mec"

# 2. Chamar a API com allow_update=true
URL=$(gcloud run services describe destaquesgovbr-scraper-api \
  --region southamerica-east1 --format 'value(status.url)')

TOKEN=$(gcloud auth print-identity-token)

curl -X POST "$URL/scrape/agencies" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{
    \"agencies\": [\"$AGENCY\"],
    \"start_date\": \"2026-02-28\",
    \"end_date\": \"2026-02-28\",
    \"allow_update\": true
  }"
```

O scraper vai re-coletar e publicar no Pub/Sub, acionando todo o pipeline real-time.

## Forçar enriquecimento de um artigo

Quando um artigo precisa ser re-classificado.

```bash
# 1. Limpar classificação no PostgreSQL
psql "$DATABASE_URL" -c "
  UPDATE news
  SET most_specific_theme_id = NULL,
      theme_l1_id = NULL,
      theme_l2_id = NULL,
      theme_l3_id = NULL,
      summary = NULL
  WHERE unique_id = 'UNIQUE_ID_AQUI';
"

# 2. Publicar mensagem no topic para acionar o enrichment
gcloud pubsub topics publish dgb.news.scraped \
  --message='{"unique_id": "UNIQUE_ID_AQUI"}' \
  --attribute="trace_id=$(uuidgen),event_version=1.0"
```

## Forçar geração de embedding

```bash
# 1. Limpar embedding
psql "$DATABASE_URL" -c "
  UPDATE news
  SET content_embedding = NULL,
      embedding_generated_at = NULL
  WHERE unique_id = 'UNIQUE_ID_AQUI';
"

# 2. Publicar no topic enriched (o embeddings API escuta esse topic)
gcloud pubsub topics publish dgb.news.enriched \
  --message='{"unique_id": "UNIQUE_ID_AQUI"}' \
  --attribute="trace_id=$(uuidgen),event_version=1.0"
```

## Sync incremental do Typesense

Para preencher lacunas ou sincronizar um período específico.

```bash
# Via CLI (requer acesso ao PostgreSQL e Typesense)
data-platform sync-typesense --start-date 2026-02-01

# Via GitHub Actions
gh workflow run typesense-maintenance-sync.yaml \
  -R destaquesgovbr/data-platform
```

## Rebuild completo do Typesense

!!! danger "Operação destrutiva"
    Este procedimento **deleta todos os documentos** da collection e recarrega do PostgreSQL. O portal fica sem resultados de busca durante o reload.

```bash
# Via GitHub Actions (requer confirmação explícita)
gh workflow run typesense-full-reload.yaml \
  -R destaquesgovbr/data-platform \
  -f confirm=DELETE
```

Duração estimada: ~30 minutos para ~300k documentos.

## Deploy manual de emergência

Quando o deploy automático via GitHub Actions não funciona.

```bash
# Variáveis
PROJECT="inspire-7-finep"
REGION="southamerica-east1"
SERVICE="destaquesgovbr-enrichment-worker"  # ajustar conforme o serviço
IMAGE="southamerica-east1-docker.pkg.dev/$PROJECT/destaquesgovbr-enrichment-worker/enrichment-worker:latest"

# Build e push
docker build -t $IMAGE -f docker/enrichment-worker/Dockerfile .
docker push $IMAGE

# Deploy
gcloud run services update $SERVICE \
  --image $IMAGE \
  --region $REGION
```

## Verificar saúde dos serviços

```bash
# Health check de todos os serviços
SERVICES=(scraper-api enrichment-worker embeddings-api typesense-sync-worker)
REGION="southamerica-east1"

for svc in "${SERVICES[@]}"; do
  URL=$(gcloud run services describe "destaquesgovbr-$svc" \
    --region $REGION --format 'value(status.url)')
  STATUS=$(curl -s -o /dev/null -w "%{http_code}" "$URL/health" \
    -H "Authorization: Bearer $(gcloud auth print-identity-token)")
  echo "$svc: $STATUS"
done
```

## Limpar DLQ

Inspecionar e processar mensagens que foram para a Dead-Letter Queue.

```bash
# 1. Listar DLQ topics
gcloud pubsub topics list --filter="name:dlq" --format="value(name)"

# 2. Criar subscription temporária para ler mensagens
DLQ_TOPIC="projects/inspire-7-finep/topics/dgb.news.scraped--enrichment-dlq"
gcloud pubsub subscriptions create temp-dlq-reader \
  --topic=$DLQ_TOPIC

# 3. Pull mensagens para inspecionar
gcloud pubsub subscriptions pull temp-dlq-reader \
  --limit=10 --auto-ack=false

# 4. Após resolver o problema, re-publicar no topic original
# (ou ACK se o artigo não precisa ser reprocessado)

# 5. Limpar subscription temporária
gcloud pubsub subscriptions delete temp-dlq-reader
```

## Trigger manual da DAG HuggingFace

```bash
# Via gcloud (requer acesso ao Composer)
gcloud composer environments run destaquesgovbr-composer \
  --location us-central1 \
  dags trigger -- sync_postgres_to_huggingface

# Via Airflow UI
# Acesse a UI do Composer → DAGs → sync_postgres_to_huggingface → Trigger
```

## Trigger manual da DAG Federation

```bash
gcloud composer environments run destaquesgovbr-composer \
  --location us-central1 \
  dags trigger -- federation_publish
```

## Verificar artigo no Typesense

```bash
# Buscar por unique_id
curl "http://TYPESENSE_HOST:8108/collections/news/documents/search" \
  -H "X-TYPESENSE-API-KEY: $API_KEY" \
  -G \
  --data-urlencode "q=UNIQUE_ID" \
  --data-urlencode "query_by=unique_id" \
  --data-urlencode "filter_by=unique_id:=UNIQUE_ID"
```
