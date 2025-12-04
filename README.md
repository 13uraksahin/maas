# MAAS - Remote Water Meter Reading Platform

A high-performance IoT application for remote water meter reading with complex multi-tenancy support.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              MAAS Architecture                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐                │
│  │   Sigfox     │     │    LoRa      │     │   NB-IoT     │                │
│  │   Devices    │     │   Devices    │     │   Devices    │                │
│  └──────┬───────┘     └──────┬───────┘     └──────┬───────┘                │
│         │                    │                    │                        │
│         └────────────────────┼────────────────────┘                        │
│                              │                                             │
│                              ▼                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │                    INGESTION LAYER (NestJS)                           │ │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                   │ │
│  │  │  Sigfox     │  │    LoRa     │  │   Generic   │  Device Adapters  │ │
│  │  │  Adapter    │  │   Adapter   │  │   Adapter   │  (Strategy)       │ │
│  │  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘                   │ │
│  │         └────────────────┼────────────────┘                          │ │
│  │                          ▼                                           │ │
│  │              ┌─────────────────────┐                                 │ │
│  │              │   Webhook Handler   │─────▶ Return 200 OK (fast!)     │ │
│  │              └──────────┬──────────┘                                 │ │
│  └─────────────────────────┼─────────────────────────────────────────────┘ │
│                            │                                               │
│                            ▼                                               │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │                      QUEUE LAYER (BullMQ)                             │ │
│  │  ┌─────────────────────────────────────────────────────────────────┐ │ │
│  │  │                    Redis Queue                                   │ │ │
│  │  │  readings: [job1, job2, job3, ...]                              │ │ │
│  │  └──────────────────────────┬──────────────────────────────────────┘ │ │
│  └─────────────────────────────┼─────────────────────────────────────────┘ │
│                                │                                           │
│                                ▼                                           │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │                    PROCESSING LAYER (Worker)                          │ │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐       │ │
│  │  │ Resolve Device  │─▶│ Calculate Delta │─▶│  Batch Insert   │       │ │
│  │  │ → Asset Mapping │  │                 │  │    (Kysely)     │       │ │
│  │  └─────────────────┘  └─────────────────┘  └────────┬────────┘       │ │
│  │                                                      │               │ │
│  │                                                      ▼               │ │
│  │                                            ┌─────────────────┐       │ │
│  │                                            │  Socket.io Emit │       │ │
│  │                                            │  (Real-time)    │       │ │
│  │                                            └────────┬────────┘       │ │
│  └─────────────────────────────────────────────────────┼─────────────────┘ │
│                                                        │                   │
│         ┌──────────────────────────────────────────────┼───────────┐       │
│         │                                              │           │       │
│         ▼                                              ▼           │       │
│  ┌──────────────┐                             ┌──────────────┐     │       │
│  │  PostgreSQL  │                             │   Nuxt 3     │     │       │
│  │  TimescaleDB │                             │   Frontend   │     │       │
│  │    + ltree   │                             └──────────────┘     │       │
│  └──────────────┘                                                  │       │
│                                                                    │       │
└────────────────────────────────────────────────────────────────────┘       │
                                                                             │
```

## Tech Stack

### Backend
- **NestJS** - Node.js framework for scalable server-side applications
- **Socket.io** - Real-time bidirectional event-based communication
- **BullMQ** - Redis-based queue for job processing
- **Prisma 6** - Next-generation ORM for type-safe database access
- **Kysely** - Type-safe SQL query builder for batch operations

### Database
- **PostgreSQL 16** - Primary database
- **ltree** extension - Hierarchical tenant tree queries
- **TimescaleDB** - High-performance time-series data

### Frontend
- **Nuxt 3** - Vue.js meta-framework
- **Tailwind CSS** - Utility-first CSS framework
- **Pinia** - State management with Vue 3 Composition API
- **ApexCharts** - Modern charting library
- **Socket.io-client** - Real-time WebSocket client

### Infrastructure
- **Docker** - Containerization
- **Dokploy** - Deployment platform
- **Redis** - In-memory data structure store

## Project Structure

```
maas/
├── api/                          # NestJS Backend
│   ├── prisma/
│   │   ├── schema.prisma         # Database schema
│   │   └── migrations/           # SQL migrations
│   ├── src/
│   │   ├── ingestion/            # IoT data ingestion module
│   │   │   ├── adapters/         # Device adapters (Strategy Pattern)
│   │   │   │   ├── device-adapter.interface.ts
│   │   │   │   ├── device-adapter.factory.ts
│   │   │   │   ├── sigfox.adapter.ts
│   │   │   │   └── lora.adapter.ts
│   │   │   ├── processors/       # BullMQ workers
│   │   │   │   └── readings.processor.ts
│   │   │   ├── dto/
│   │   │   ├── ingestion.controller.ts
│   │   │   ├── ingestion.service.ts
│   │   │   └── ingestion.module.ts
│   │   ├── socket/               # Socket.io gateway
│   │   ├── kysely/               # Kysely query builder setup
│   │   ├── prisma/               # Prisma service
│   │   └── tenant/               # Tenant management (ltree)
│   └── Dockerfile
│
├── app/                          # Nuxt 3 Frontend
│   ├── assets/css/
│   ├── components/
│   │   └── charts/
│   │       └── RealTimeConsumptionChart.vue
│   ├── composables/
│   │   └── useSocket.ts          # Socket.io composable
│   ├── layouts/
│   ├── pages/
│   ├── plugins/
│   │   └── apexcharts.client.ts
│   ├── stores/
│   │   └── liveReadings.ts       # Pinia store with throttling
│   └── Dockerfile
│
├── docker-compose.yml
└── README.md
```

## Database Schema

### Entity Relationships

```
┌─────────────────────────────────────────────────────────────────┐
│                        HIERARCHY                                │
│  ┌──────────┐                                                   │
│  │  Tenant  │◄──── ltree path for hierarchy                     │
│  │  (root)  │      "manufacturer.municipality.district"         │
│  └────┬─────┘                                                   │
│       │ 1:N                                                     │
├───────┼─────────────────────────────────────────────────────────┤
│       │              COMMERCIAL                                 │
│       ▼                                                         │
│  ┌──────────┐                                                   │
│  │ Customer │ ◄──── End user / Billing entity                   │
│  └────┬─────┘                                                   │
│       │ 1:N (optional)                                          │
├───────┼─────────────────────────────────────────────────────────┤
│       │               PHYSICAL                                  │
│       ▼                                                         │
│  ┌──────────┐         ┌───────────────────┐      ┌──────────┐  │
│  │  Asset   │◄────────│ DeviceAllocation  │─────▶│  Device  │  │
│  │ (meter)  │         │ (start/end date)  │      │ (modem)  │  │
│  └────┬─────┘         └───────────────────┘      └──────────┘  │
│       │                                                         │
├───────┼─────────────────────────────────────────────────────────┤
│       │              TIME-SERIES                                │
│       ▼                                                         │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                    Reading (Hypertable)                   │  │
│  │  time | asset_id | device_id | value | delta | signal    │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

## Getting Started

### Prerequisites

- Node.js 20+
- Docker & Docker Compose
- pnpm (recommended) or npm

### Development Setup

1. **Clone and install dependencies:**

```bash
# Clone repository
cd maas

# Install API dependencies
cd api && npm install

# Install App dependencies
cd ../app && npm install
```

2. **Start infrastructure services:**

```bash
# Start PostgreSQL and Redis
docker-compose up -d postgres redis
```

3. **Setup database:**

```bash
cd api

# Generate Prisma client
npm run db:generate

# Run migrations
npm run db:migrate

# (Optional) Open Prisma Studio
npm run db:studio
```

4. **Start development servers:**

```bash
# Terminal 1 - API
cd api && npm run start:dev

# Terminal 2 - App
cd app && npm run dev
```

5. **Access the application:**

- Frontend: http://localhost:3000
- API: http://localhost:3001
- API Status: http://localhost:3001/api/ingestion/status

### Production Deployment

```bash
# Build and start all services
docker-compose up -d --build

# View logs
docker-compose logs -f
```

## API Endpoints

### Ingestion

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/ingestion/webhook` | Generic webhook (auto-detect) |
| POST | `/api/ingestion/sigfox` | Sigfox callback |
| POST | `/api/ingestion/lora` | LoRaWAN callback |
| GET | `/api/ingestion/status` | Queue status & health |

### Tenants

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/tenants` | List all tenants |
| POST | `/api/tenants` | Create tenant |
| GET | `/api/tenants/:id` | Get tenant details |
| GET | `/api/tenants/:id/subtree` | Get tenant subtree (ltree) |
| GET | `/api/tenants/:id/children` | Get direct children |

## Webhook Examples

### Sigfox Payload

```json
{
  "device": "1A2B3C",
  "data": "0001e24000550012",
  "time": 1701619200,
  "snr": 12.5,
  "rssi": -95,
  "seqNumber": 42
}
```

### LoRaWAN Payload (TTN v3)

```json
{
  "devEUI": "0004A30B001F6E8A",
  "frmPayload": "AAHiQABVABI=",
  "fPort": 1,
  "fCnt": 123,
  "rxInfo": [
    {
      "gatewayId": "eui-b827ebfffe123456",
      "rssi": -85,
      "snr": 8.5,
      "time": "2024-12-03T10:00:00.000Z"
    }
  ]
}
```

## Real-time Socket Events

### Client → Server

| Event | Data | Description |
|-------|------|-------------|
| `subscribe:tenant` | `{ tenantId }` | Join tenant room |
| `subscribe:asset` | `{ assetId }` | Subscribe to asset updates |
| `unsubscribe:asset` | `{ assetId }` | Unsubscribe from asset |

### Server → Client

| Event | Data | Description |
|-------|------|-------------|
| `reading:new` | `Reading` | New meter reading |
| `reading:batch` | `Reading[]` | Batch of readings |
| `alert:new` | `Alert` | New alert |
| `device:status` | `DeviceStatus` | Device status change |

## Performance Considerations

### Ingestion Pipeline

1. **No synchronous DB writes** - Webhooks immediately queue jobs
2. **BullMQ concurrency** - 10 concurrent job processors
3. **Batch inserts** - Readings buffered and inserted in batches
4. **TimescaleDB hypertables** - Optimized for time-series queries

### Frontend Optimization

1. **Throttled updates** - UI updates limited to 1/second
2. **Event buffering** - Socket events buffered before processing
3. **Circular buffers** - Limited data points per asset (100)
4. **Client-side filtering** - Minimize re-renders

## Environment Variables

### API (.env)

```env
DATABASE_URL="postgresql://postgres:postgres@localhost:5432/maas"
REDIS_HOST="localhost"
REDIS_PORT=6379
PORT=3001
NODE_ENV="development"
FRONTEND_URL="http://localhost:3000"
```

### App (.env)

```env
NUXT_PUBLIC_API_BASE_URL="http://localhost:3001"
NUXT_PUBLIC_SOCKET_URL="http://localhost:3001"
```

## License

MIT

