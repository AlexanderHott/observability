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

### Fail deployments on startup errors

In Dokploy, set **Advanced → Command** to the full Compose command below for this deployment. Dokploy prefixes it with `docker`, so do not include that word:

```text
compose -p observability-compose-puthfp --env-file .env -f ./docker-compose.yml up -d --build --remove-orphans --wait --wait-timeout 120
```

This preserves the current project name, environment file, and Compose path. For another deployment, copy its default command and append `--wait --wait-timeout 120`. This setting lives in Dokploy and is not applied by merging this repository.

[Compose `--wait`](https://docs.docker.com/reference/cli/docker/compose/up/) waits for services to be running, or healthy when they have Docker healthchecks. [Dokploy accepts a complete custom command](https://docs.dokploy.com/docs/core/docker-compose) and propagates its failure. A failed startup should fail the deployment; keeping containers running with `restart: unless-stopped` is a separate recovery policy. A failed deployment does not automatically restore the previous images.

These upstream images do not all provide Docker healthchecks, so `--wait` alone is a startup check, not proof of readiness or protection against a crash after the check finishes. The collector exposes its health extension at `http://otel-collector:13133/` on the internal network. Do not publish it through Traefik. The official collector image has no shell or `curl`, so an in-container `curl` healthcheck would itself fail. Use an external readiness probe for runtime monitoring; CI probes the collector and all four backends over HTTP after startup.

### Image updates

The deployment targets `linux/arm64`, matching the Dokploy host. Every service declares that platform. The collector uses an explicit `-arm64` release tag and matching ARM64 digest. Other services use multi-platform digests containing ARM64 builds. Tempo retains `latest` with an immutable digest because the deployed build reports `3.0.0` but differs from the published `3.0.0` release; switching it to a release image requires separate compatibility validation.

Dependabot checks Docker Compose images weekly and opens update PRs, with at most five open at once. Keep the collector's `-arm64` suffix: Dependabot's Docker tag comparison preserves alphabetic variants. An unsuffixed tag previously updated to `-386`, which its version parser can treat as a numeric version component. Dependabot does not expose a global architecture allowlist; do not add unsupported version-glob rules to simulate one.

The **ARM64 images and startup** CI job runs natively on ARM64. It checks the collector tag, pulls every pinned image, verifies each image's actual platform, starts the stack, checks all five readiness endpoints and OTLP authentication, and rejects container restarts. It uses disposable volumes and test credentials; it does not validate migrations against production data. Make this check required in the GitHub ruleset for `main` so incompatible updates cannot be merged. Adding the workflow does not itself change branch protection. Dependabot also updates the CI action pins.

Review and merge update PRs before deploying; digest changes also require review. Dokploy deploys merged changes automatically only when auto-deploy is enabled.

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
- Collector health: http://localhost:13133/, bound to loopback only

Local OTLP/HTTP requests still require the bearer token. Local OTLP/gRPC on port `4317` does not require it.
