# Cloud Run Mapping

## Configuration Variables (`--set-env-vars`)
Non-sensitive configuration values that control application behavior and vary across environments.

- **`LOG_LEVEL`**: Application log verbosity (`info`, `debug`, `warn`, `error`). Injected via `--set-env-vars LOG_LEVEL="info"`.
- **`DATABASE_URL`**: Database connection URI (host, database name, non-secret parameters). Injected via `--set-env-vars DATABASE_URL="postgres://user:password@localhost:5432/mydb"` (or referenced via Secret Manager if credentials are sensitive).
- **`PORT`**: Port number for the HTTP server. Cloud Run sets `PORT=8080` automatically, or custom values can be set via `--set-env-vars PORT="8080"`.

## Secrets (`--set-secrets`)
Sensitive credentials, access tokens, and passwords that must never be baked into container images or committed to Git. Stored in Google Cloud Secret Manager.

- **`API_KEY`**: Private API authentication key. Mounted at runtime via `--set-secrets API_KEY="api-key-secret:latest"`.
- **`DATABASE_URL`** *(Best Practice for Production)*: Database connection URI with embedded passwords can be stored in Secret Manager and injected via `--set-secrets DATABASE_URL="db-connection-string:latest"`.

## Example Cloud Run CLI Command

```bash
gcloud run deploy envlab-service \
  --image gcr.io/PROJECT_ID/envlab:latest \
  --region us-central1 \
  --platform managed \
  --allow-unauthenticated \
  --set-env-vars LOG_LEVEL="info",DATABASE_URL="postgres://user:password@localhost:5432/mydb" \
  --set-secrets API_KEY="api-key-secret:latest"
```
