# Observability Stack

Stack self-hosted de observabilidade para VPS.

## Serviços

| Serviço            | Porta | Função                          |
|--------------------|-------|---------------------------------|
| Grafana            | 3000  | Dashboard e visualização        |
| Prometheus         | 9090  | Armazenamento de métricas       |
| Loki               | 3100  | Armazenamento de logs           |
| Tempo              | 3200  | Armazenamento de traces         |
| OTEL Collector     | 4317  | Receptor gRPC (principal)       |
| OTEL Collector     | 4318  | Receptor HTTP (frontend/fallback)|

## Subir a stack

```bash
docker compose up -d
```

Acesse o Grafana em http://localhost:3000
Login padrão: **admin / admin** (troque na primeira vez)

## Importar dashboards prontos

No Grafana: **Dashboards → Import** e cole os IDs abaixo:

| ID    | Dashboard                          |
|-------|------------------------------------|
| 1860  | Node Exporter Full (VPS: CPU/RAM)  |
| 19792 | OpenTelemetry Collector            |
| 13639 | Loki Logs Explorer                 |

## Parar / remover

```bash
# Parar sem remover dados
docker compose stop

# Remover tudo incluindo volumes
docker compose down -v
```

## Estrutura de arquivos

```
observability/
├── docker-compose.yml
├── otel-collector/
│   └── config.yaml
├── prometheus/
│   └── prometheus.yml
└── grafana/
    └── provisioning/
        ├── datasources/
        │   └── datasources.yaml   # Prometheus + Loki + Tempo pré-configurados
        └── dashboards/
            └── dashboards.yaml
```
