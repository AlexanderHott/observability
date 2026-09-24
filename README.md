# Shared Observability

OpenTelemetry Collector, Prometheus, Grafana, Tempo, and Loki services. Public OTLP/HTTP ingestion requires a bearer token.

## Deployment

Set these environment variables in the Compose service's **Environment** tab in Dokploy:

```env
GRAFANA_ADMIN_USER=admin
GRAFANA_ADMIN_PASSWORD=<strong-password>
OTEL_BEARER_TOKEN=<long-random-token>
```

Generate a token with:

```bash
openssl rand -hex 32
```

Configure the existing HTTPS domain for the `otel-collector` service on container port `4318`. Configure the Grafana domain on container port `3000`. Do not create public routes for ports `4317`, `9464`, `9090`, `3100`, or `3200`.

### Startup and recovery

Save the environment variables and deploy the service. [Dokploy writes them to a `.env` file beside the Compose file](https://docs.dokploy.com/docs/core/docker-compose#environment). The Compose file explicitly maps the required variables into Grafana and the collector; keep the missing-value checks so startup fails when secrets are absent.

If Dokploy's **Start** action reports missing `GRAFANA_ADMIN_PASSWORD` or `OTEL_BEARER_TOKEN` despite saved values, verify that `.env` exists in the deployed checkout alongside `docker-compose.yml`. From that directory on the server, validate interpolation without printing secrets:

```bash
docker compose --env-file .env -f docker-compose.yml config --quiet
```

If the file is missing or stale, save the Environment settings and redeploy, then verify the generated file again. Adding a service-level `env_file` does not replace the `.env` file needed for Compose interpolation. Do not commit secrets to Git.

All five services use `restart: unless-stopped` to recover after process exits or Docker restarts. An intentionally stopped container stays stopped until started manually. Check the actual containers and Grafana's `/api/health` endpoint after deployment; Dokploy's last successful deployment status alone does not show runtime health.

### Image updates

Images are pinned to the multi-platform digests deployed on 2026-09-24. Four also use matching release tags. Tempo retains `latest` with an immutable digest because the deployed build reports `3.0.0` but differs from the published `3.0.0` release; switching it to a release image requires separate compatibility validation.

Dependabot checks Docker Compose images weekly and opens update PRs, with at most five open at once. Review and merge those PRs before deploying; digest changes also require review. Dokploy deploys merged changes automatically only when auto-deploy is enabled.

### Grafana admin password

Grafana initializes the admin credentials from the environment when creating a new database. Existing credentials persist in `grafana-data`; changing `GRAFANA_ADMIN_PASSWORD` does not rotate an existing account's password. Startup no longer resets that password.

Use Grafana's UI for password changes, or run this explicit recovery command from the deployed Compose directory to reset the default admin account to the configured container secret:

```bash
docker compose exec -T grafana sh -c 'printf "%s" "$GF_SECURITY_ADMIN_PASSWORD" | /usr/share/grafana/bin/grafana cli --homepath /usr/share/grafana --config /etc/grafana/grafana.ini admin reset-admin-password --password-from-stdin'
```

## Configure Projects

Set these variables on every application that exports telemetry through the HTTPS endpoint:

```env
OTEL_EXPORTER_OTLP_ENDPOINT=https://otel.example.com
OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
OTEL_EXPORTER_OTLP_HEADERS=authorization=Bearer%20<your-token>
OTEL_SERVICE_NAME=my-api
```

Replace `otel.example.com` and the token with the domain and secret. Use a unique `OTEL_SERVICE_NAME` for every application.

## Local Development

Create `.env` from `.env.example`, set both secrets, and publish the local ports with the overlay:

```bash
docker compose -f docker-compose.yml -f docker-compose.local.yml up -d
```

The local services are available at:

- Grafana: http://localhost:3000
- Prometheus: http://localhost:9090
- Tempo: http://localhost:3200
- Loki: http://localhost:3100
- OTLP/gRPC: http://localhost:4317
- OTLP/HTTP: http://localhost:4318

Local OTLP/HTTP requests still require the bearer token. Local OTLP/gRPC on port `4317` does not require it.
