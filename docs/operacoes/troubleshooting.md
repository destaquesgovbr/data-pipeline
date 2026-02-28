# Troubleshooting

Guia organizado por sintoma. Inclui problemas reais encontrados durante a validação do pipeline em produção (fevereiro/2026).

## Artigos não aparecem no portal

### Sintoma
Artigo foi coletado pelo scraper mas não aparece na busca do portal.

### Diagnóstico

1. **Verificar se o artigo está no PostgreSQL**:
    ```sql
    SELECT unique_id, most_specific_theme_id, content_embedding IS NOT NULL as has_embedding
    FROM news WHERE unique_id = 'ID_DO_ARTIGO';
    ```

2. **Verificar enriquecimento** (`most_specific_theme_id IS NULL` = não processado):
    - Checar logs do enrichment-worker
    - Checar backlog da subscription `dgb.news.scraped--enrichment`

3. **Verificar Typesense**:
    ```bash
    curl "http://TYPESENSE_HOST:8108/collections/news/documents/search?q=UNIQUE_ID&query_by=unique_id" \
      -H "X-TYPESENSE-API-KEY: KEY"
    ```

4. **Cache do portal**: Homepage tem ISR com revalidate=600s (10min). A busca direta é real-time.

## Enrichment falhou

### "Unable to locate credentials" (AWS Bedrock)

**Causa**: O worker lê credenciais do `AWS_BEDROCK_CONNECTION_URI` (formato Airflow: `aws://KEY:SECRET@/?region_name=us-east-1`) mas tentava usar variáveis individuais `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY`.

**Fix**: O handler agora faz parse automático do URI via `_parse_aws_credentials()`.

### AWS Signature mismatch (String-to-Sign error)

**Causa**: O secret key no URI contém caracteres URL-encoded (`%2F` para `/`, `%2B` para `+`). O `urlparse` retorna valores encoded, mas o boto3 precisa dos valores decoded.

**Fix**: Adicionado `unquote()` para decodificar username e password do URI.

### TypeError: 'str' object does not support item assignment

**Causa**: `classifier.classify_single(article)` retornava JSON string (default `return_format="json"`) em vez de dict.

**Fix**: Alterado para `classify_single(article, return_format="dict")`.

## Embeddings não gerados

### Timeout ou OOM

**Causa**: O modelo sentence-transformer consome ~1.5Gi de RAM. Se o container estiver com pouca memória, o carregamento falha.

**Verificar**: Logs do Cloud Run para erros de memória. O serviço tem 4Gi — suficiente, mas se houver múltiplas requests concorrentes, pode ser problema.

### Artigo sem content

Se `title`, `summary` e `content` forem todos NULL/vazios, o embedding não é gerado. O worker loga e faz ACK (não vai para DLQ).

## Typesense desatualizado

### "localhost:8108 Connection refused"

**Causa**: O client.py lia `TYPESENSE_HOST` (default `localhost`), mas o Cloud Run injeta credenciais via `TYPESENSE_WRITE_CONN` (JSON).

**Fix**: Adicionado `_parse_write_conn()` que faz parse do JSON `{"host": "...", "port": "8108", "apiKey": "...", "protocol": "http"}`.

### "'PostgresManager' object has no attribute 'engine'"

**Causa**: O handler usava `pg.engine` mas o atributo era privado (`self._engine`).

**Fix**: Adicionada property pública `engine` ao PostgresManager.

### "List argument must consist only of tuples or dictionaries"

**Causa**: `pd.read_sql_query(query, pg.engine, params=[unique_id])` — psycopg2 requer tuple, não list.

**Fix**: Alterado para `params=(unique_id,)`.

### Deploy não trigou após merge

**Causa**: O path filter do workflow `typesense-sync-worker-deploy.yaml` não incluía `src/data_platform/managers/` nem `src/data_platform/typesense/`.

**Workaround**: Trigger manual via `gh workflow run typesense-sync-worker-deploy.yaml --ref main`.

**Fix permanente**: Adicionar esses paths ao trigger do workflow.

## HuggingFace não atualizado

### DAG falha com "Token expired"

**Verificar**: O token HuggingFace na connection `huggingface_default` do Airflow. Tokens HF podem expirar.

### Dados duplicados

**Não deveria acontecer**: A DAG faz dedup via Dataset Viewer API. Se acontecer, verificar se a API do HF está respondendo corretamente.

## Federation não publica

### ap_publish_queue vazia mas artigos novos existem

**Verificar**: O watermark em `ap_sync_watermark`. Se o `last_created_at` estiver muito à frente, a query não encontra artigos.

**Fix**: Atualizar manualmente o watermark:
```sql
UPDATE ap_sync_watermark SET last_created_at = 'TIMESTAMP_CORRETO';
```

### POST /trigger-publish retorna erro

**Verificar**: Token de autenticação e conectividade entre o Composer e o Cloud Run. A DAG usa IAM para autenticação.

## Cloud Run cold start lento

**Esperado**: Com `min_instance_count=0`, a primeira request após período de inatividade tem cold start (~2-5s para workers Python, ~1-2s para Node.js).

**Mitigação**: Se a latência for crítica, configurar `min_instance_count=1` no Terraform. Custo: ~$25/mês por instância sempre ativa.

## Erros de conexão com banco

### "connection refused" ou timeout

**Verificar**:

1. VPC Connector configurado no Cloud Run
2. Cloud SQL IP privado acessível
3. Pool de conexões não esgotado (checar `num_backends` no Cloud SQL)

### "too many connections"

**Causa**: Cada instância Cloud Run abre conexões. Com scale-up agressivo, pode exceder o limite.

**Fix**: Usar connection pooling (PgBouncer) ou reduzir `max_instance_count`.
