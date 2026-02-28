# Adicionando um Novo Componente ao Pipeline

Guia passo-a-passo para adicionar um novo worker event-driven ao pipeline.

## 1. Definir o Topic Pub/Sub

Decida:

- Qual topic existente o worker vai **consumir**?
- O worker precisa **publicar** um novo topic?

Adicione o topic no Terraform (`infra/terraform/pubsub.tf`):

```hcl
resource "google_pubsub_topic" "novo_topic" {
  name                       = "dgb.news.EVENTO"
  message_retention_duration = "604800s"  # 7 dias
}

resource "google_pubsub_topic" "novo_topic_dlq" {
  name                       = "dgb.news.EVENTO-dlq"
  message_retention_duration = "604800s"
}
```

## 2. Criar o Worker (Cloud Run)

Use FastAPI com dois endpoints: `/process` (Pub/Sub handler) e `/health`.

### Template base

```python
"""Worker template para Pub/Sub push subscription."""
import base64
import json
import logging
import os

from fastapi import FastAPI, Request, Response

logger = logging.getLogger(__name__)
app = FastAPI()


@app.get("/health")
async def health():
    return {"status": "ok"}


@app.post("/process")
async def process(request: Request):
    """Handler para push subscription do Pub/Sub."""
    try:
        envelope = await request.json()
        message = envelope.get("message", {})
        data = json.loads(base64.b64decode(message.get("data", "")).decode())
        unique_id = data.get("unique_id")

        if not unique_id:
            logger.warning("Mensagem sem unique_id, ignorando")
            return Response(status_code=200)

        # Verificação de idempotência
        # if already_processed(unique_id):
        #     return Response(status_code=200)

        # Processamento
        logger.info(f"Processando {unique_id}")
        # ... lógica do worker ...

        # Publicar evento de saída (se aplicável)
        # publish_event(unique_id)

        return Response(status_code=200)

    except Exception as e:
        logger.error(f"Erro processando mensagem: {e}", exc_info=True)
        # Retorna 200 para evitar retry infinito (ACK-always)
        return Response(status_code=200)
```

### Pontos importantes

- **Sempre retorne HTTP 200**, mesmo em caso de erro (estratégia ACK-always)
- **Implemente idempotência**: verifique se o artigo já foi processado
- **Logue `unique_id`** em todas as mensagens para rastreabilidade
- **Use `trace_id`** dos atributos da mensagem para correlação

## 3. Criar Dockerfile

```dockerfile
FROM python:3.11-slim

WORKDIR /app
COPY pyproject.toml poetry.lock ./
RUN pip install poetry && poetry install --no-dev --no-interaction

COPY src/ ./src/
CMD ["poetry", "run", "uvicorn", "modulo.worker.app:app", "--host", "0.0.0.0", "--port", "8080"]
```

## 4. Provisionar no Terraform

No repo `infra`, crie um novo arquivo `novo-worker.tf`:

```hcl
# Service Account
resource "google_service_account" "novo_worker" {
  account_id   = "destaquesgovbr-novo-worker"
  display_name = "Novo Worker Service Account"
}

# Cloud Run Service
resource "google_cloud_run_v2_service" "novo_worker" {
  name     = "destaquesgovbr-novo-worker"
  location = var.region

  template {
    service_account = google_service_account.novo_worker.email

    containers {
      image = "hello:latest"  # Placeholder até primeiro deploy

      resources {
        limits = {
          cpu    = "1"
          memory = "512Mi"
        }
      }

      env {
        name  = "DATABASE_URL"
        value_source {
          secret_key_ref {
            secret  = "govbrnews-postgres-connection-string"
            version = "latest"
          }
        }
      }
    }

    scaling {
      min_instance_count = 0
      max_instance_count = 3
    }
  }
}

# Push Subscription
resource "google_pubsub_subscription" "novo_worker_sub" {
  name  = "dgb.news.TOPIC--novo-worker"
  topic = google_pubsub_topic.TOPIC.name

  push_config {
    push_endpoint = "${google_cloud_run_v2_service.novo_worker.uri}/process"
    oidc_token {
      service_account_email = google_service_account.pubsub_push.email
    }
  }

  ack_deadline_seconds = 300

  dead_letter_policy {
    dead_letter_topic     = google_pubsub_topic.TOPIC_dlq.id
    max_delivery_attempts = 5
  }

  retry_policy {
    minimum_backoff = "10s"
    maximum_backoff = "300s"
  }
}

# IAM: Pub/Sub SA pode invocar o Cloud Run
resource "google_cloud_run_v2_service_iam_member" "novo_worker_invoker" {
  name     = google_cloud_run_v2_service.novo_worker.name
  location = var.region
  role     = "roles/run.invoker"
  member   = "serviceAccount:${google_service_account.pubsub_push.email}"
}

# IAM: Worker SA acessa secrets
resource "google_secret_manager_secret_iam_member" "novo_worker_db" {
  secret_id = "govbrnews-postgres-connection-string"
  role      = "roles/secretmanager.secretAccessor"
  member    = "serviceAccount:${google_service_account.novo_worker.email}"
}
```

## 5. Criar Workflow de Deploy

```yaml
# .github/workflows/novo-worker-deploy.yaml
name: Deploy Novo Worker

on:
  push:
    branches: [main]
    paths:
      - "src/modulo/worker/**"
      - "docker/novo-worker/**"
      - "pyproject.toml"
      - "poetry.lock"
  workflow_dispatch:

jobs:
  deploy:
    uses: destaquesgovbr/reusable-workflows/.github/workflows/cloud-run-deploy.yml@v2
    with:
      dockerfile: docker/novo-worker/Dockerfile
      ar_repository: destaquesgovbr-novo-worker
      image_name: novo-worker
      cloud_run_service: destaquesgovbr-novo-worker
```

## 6. Documentar

Adicione uma página em `docs/componentes/novo-worker.md` seguindo o template dos componentes existentes e atualize o `nav` em `mkdocs.yml`.

## Checklist

- [ ] Topic e DLQ topic criados no Terraform
- [ ] Subscription com push config, DLQ policy e retry policy
- [ ] Service Account com IAM mínimo (secret accessor, SQL client, pubsub publisher se necessário)
- [ ] Cloud Run service com health check e scale-to-zero
- [ ] Worker implementa idempotência
- [ ] Worker usa estratégia ACK-always
- [ ] Worker loga `unique_id` e `trace_id`
- [ ] Dockerfile criado e testado localmente
- [ ] Workflow de deploy com path filters corretos
- [ ] Path filters incluem **todos** os diretórios que o worker usa (managers, shared libs, etc.)
- [ ] Documentação atualizada neste repo
