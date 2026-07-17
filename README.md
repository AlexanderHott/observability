# Shared Observability

OpenTelemetry Collector, Prometheus, Grafana, Tempo, and Loki services. Public OTLP/HTTP ingestion requires a bearer token.

## Deployment

Set these environment variables:

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
