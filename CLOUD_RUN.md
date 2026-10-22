# Cloud Run Environment Configuration Mapping

This document outlines how the containerized application's configuration and secrets map to Google Cloud Run runtime deployment parameters.

## Architectural Principles
1. **Never bake secrets into the container image**: Secrets must be stored outside images and injected securely at runtime.
2. **Distinguish Configuration from Secrets**:
   - **Configuration** (non-sensitive parameters like log level, hostnames, ports) is passed directly via environment variables (`--set-env-vars`).
   - **Secrets** (API tokens, passwords, encryption keys) are managed in Google Cloud Secret Manager and securely mounted into the container environment via Secret Manager integration (`--set-secrets`).

---

## Variable Mapping Table

| Variable Name | Type | Local Docker Mechanism | Cloud Run Mapping | Cloud Run Flag / Syntax |
| :--- | :--- | :--- | :--- | :--- |
| `DATABASE_URL` | Configuration / Secret | `--env-file .env` | Cloud Run Environment Variable or Secret | `--set-env-vars DATABASE_URL="postgres://user:password@localhost:5432/mydb"`<br>*(or `--set-secrets DATABASE_URL=db-url-secret:latest` if password is sensitive)* |
| `LOG_LEVEL` | Configuration | `--env-file .env` | Cloud Run Environment Variable | `--set-env-vars LOG_LEVEL="info"` |
| `PORT` | Configuration | `-p 3000:3000` | Cloud Run Environment Variable | Cloud Run automatically injects `PORT=8080` (or `--set-env-vars PORT="8080"`) |
| `API_KEY` | Secret | `-e API_KEY=$API_KEY` | Google Cloud Secret Manager | `--set-secrets API_KEY=api-key-secret:latest` |

---

## Detailed Mapping

### 1. Configuration Variables (`--set-env-vars`)
Values that vary across environments (Development, Staging, Production) but do not present a security vulnerability if exposed in deployment manifests:
- **`LOG_LEVEL`**: Injected via `--set-env-vars LOG_LEVEL=info`. Sets log filtering level for the application.
- **`DATABASE_URL`**: Injected via `--set-env-vars DATABASE_URL="postgres://..."`. In standard non-sensitive connection string setups.

### 2. Secrets (`--set-secrets`)
Values requiring strict access controls, audit logs, and encryption at rest:
- **`API_KEY`**: Stored in Google Cloud Secret Manager:
  ```bash
  echo -n "super-secret-key-value" | gcloud secrets create api-key-secret --data-file=-
  ```
  And injected into Cloud Run:
  ```bash
  --set-secrets API_KEY=api-key-secret:latest
  ```

---

## Example Cloud Run Deployment Command

```bash
gcloud run deploy envlab-service \
  --image gcr.io/PROJECT_ID/envlab:latest \
  --region us-central1 \
  --platform managed \
  --allow-unauthenticated \
  --set-env-vars LOG_LEVEL="info",DATABASE_URL="postgres://user:password@localhost:5432/mydb" \
  --set-secrets API_KEY="api-key-secret:latest"
```
