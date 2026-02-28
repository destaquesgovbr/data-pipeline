# Monitoramento e Observabilidade

## Métricas por Serviço

### Cloud Run

Cada serviço Cloud Run expõe métricas automáticas no Cloud Monitoring:

| Métrica | Descrição | Alerta Sugerido |
|---------|-----------|-----------------|
| `request_count` | Total de requisições | Error rate > 5% por 10min |
| `request_latencies` | Latência por request | p95 > 30s |
| `instance_count` | Instâncias ativas | — (informacional) |
| `container/cpu/utilization` | Uso de CPU | > 80% sustained |
| `container/memory/utilization` | Uso de memória | > 90% |

**Filtro por serviço**:
```
resource.type="cloud_run_revision"
resource.labels.service_name="destaquesgovbr-enrichment-worker"
```

### Pub/Sub

| Métrica | Descrição | Alerta Sugerido |
|---------|-----------|-----------------|
| `num_undelivered_messages` | Backlog por subscription | > 100 por 30min |
| `oldest_unacked_message_age` | Idade da msg mais antiga | > 1h |
| `dead_letter_message_count` | Mensagens na DLQ | > 0 por 15min |

!!! warning "DLQ > 0 requer ação"
    Qualquer mensagem na DLQ indica falha persistente. Veja [Troubleshooting](troubleshooting.md) e [Runbook: Limpar DLQ](runbooks.md#limpar-dlq).

### Cloud Composer (Airflow)

| Métrica | Descrição | Alerta Sugerido |
|---------|-----------|-----------------|
| DAG run success/failure | Status por DAG | Failure count > 3 consecutivos |
| Task duration | Duração por task | > 2x média histórica |
| Scheduler heartbeat | Saúde do scheduler | Ausente por > 5min |

Acesse a UI do Airflow para visualizar DAG runs, logs e grafos de dependência.

### Cloud SQL (PostgreSQL)

| Métrica | Descrição | Alerta Sugerido |
|---------|-----------|-----------------|
| `database/cpu/utilization` | Uso de CPU | > 80% |
| `database/memory/utilization` | Uso de memória | > 90% |
| `database/num_backends` | Conexões ativas | > 80% do max |
| `database/disk/bytes_used` | Disco usado | > 80% capacity |

## Logging

### Correlação com trace_id

Todas as mensagens Pub/Sub incluem um atributo `trace_id` (UUID). Use-o para rastrear um artigo por todo o pipeline:

```
jsonPayload.trace_id="abc123-def456-..."
```

### Filtros úteis no Cloud Logging

**Erros do enrichment worker**:
```
resource.type="cloud_run_revision"
resource.labels.service_name="destaquesgovbr-enrichment-worker"
severity>=ERROR
```

**Artigos processados pelo pipeline (por unique_id)**:
```
jsonPayload.unique_id="mec-2026-02-28-titulo"
```

**Cold starts do Cloud Run**:
```
resource.type="cloud_run_revision"
textPayload:"Starting"
```

## Dashboard Recomendado

### Pipeline Health

Painel com 4 seções:

1. **Throughput**: Mensagens publicadas por topic (scraped, enriched, embedded) — últimas 24h
2. **Backlog**: `num_undelivered_messages` por subscription — tempo real
3. **Latência**: Request latency por Cloud Run service — p50/p95
4. **Erros**: Error rate por serviço + DLQ count — últimas 24h

### Backlog Monitor

Tabela com status de processamento:

```sql
-- Artigos pendentes de enriquecimento
SELECT COUNT(*) FROM news
WHERE most_specific_theme_id IS NULL
  AND published_at > NOW() - INTERVAL '7 days';

-- Artigos pendentes de embedding
SELECT COUNT(*) FROM news
WHERE content_embedding IS NULL
  AND most_specific_theme_id IS NOT NULL
  AND published_at > NOW() - INTERVAL '7 days';
```

## Health Checks

Todos os serviços Cloud Run expõem `GET /health`:

```bash
# Verificar todos os serviços
for svc in scraper-api enrichment-worker embeddings-api typesense-sync-worker; do
  URL=$(gcloud run services describe "destaquesgovbr-$svc" \
    --region southamerica-east1 --format 'value(status.url)')
  echo "$svc: $(curl -s "$URL/health")"
done
```
