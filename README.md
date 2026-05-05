# Observability Stack

Stack self-hosted de observabilidade para VPS com ElysiaJS + React.

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

## Integração com ElysiaJS

### Instalar dependências

```bash
bun add @elysiajs/opentelemetry @opentelemetry/sdk-node
bun add @opentelemetry/exporter-trace-otlp-grpc
bun add @opentelemetry/exporter-metrics-otlp-grpc
bun add @opentelemetry/sdk-metrics
```

### Configurar (src/telemetry.ts)

```ts
import { NodeSDK } from '@opentelemetry/sdk-node'
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-grpc'
import { OTLPMetricExporter } from '@opentelemetry/exporter-metrics-otlp-grpc'
import { PeriodicExportingMetricReader } from '@opentelemetry/sdk-metrics'
import { BatchSpanProcessor } from '@opentelemetry/sdk-trace-base'

const sdk = new NodeSDK({
  serviceName: 'meu-crm-backend',
  spanProcessors: [
    new BatchSpanProcessor(
      new OTLPTraceExporter({ url: 'http://localhost:4317' })
    ),
  ],
  metricReader: new PeriodicExportingMetricReader({
    exporter: new OTLPMetricExporter({ url: 'http://localhost:4317' }),
    exportIntervalMillis: 15_000,
  }),
})

sdk.start()
```

### Usar no app (src/index.ts)

```ts
// Importar ANTES de qualquer outra coisa
import './telemetry'

import { Elysia } from 'elysia'
import { opentelemetry } from '@elysiajs/opentelemetry'

const app = new Elysia()
  .use(opentelemetry())
  .get('/', () => 'ok')
  .listen(3000)
```

O plugin `opentelemetry()` do ElysiaJS automaticamente cria spans para cada rota,
captura erros e propaga o trace context nos headers.

## Integração com React (frontend)

```bash
bun add @opentelemetry/sdk-web @opentelemetry/instrumentation-fetch
```

```ts
// src/telemetry.ts
import { WebTracerProvider } from '@opentelemetry/sdk-web'
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http'
import { FetchInstrumentation } from '@opentelemetry/instrumentation-fetch'
import { registerInstrumentations } from '@opentelemetry/instrumentation'
import { BatchSpanProcessor } from '@opentelemetry/sdk-trace-base'

const provider = new WebTracerProvider()

provider.addSpanProcessor(
  new BatchSpanProcessor(
    new OTLPTraceExporter({
      url: 'http://localhost:4318/v1/traces', // HTTP (não gRPC no browser)
    })
  )
)

provider.register()

registerInstrumentations({
  instrumentations: [
    new FetchInstrumentation({
      propagateTraceHeaderCorsUrls: [/localhost/], // ajuste para seu domínio
    }),
  ],
})
```

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
