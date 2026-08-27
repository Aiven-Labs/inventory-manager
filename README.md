# Inventory Manager

A multi-service demo application with a Node.js web backend, a React 
frontend, a PostgreSQL database, a Valkey cache, a Valkey queue, and a Go-based 
background job service.

## Architecture

```mermaid
flowchart LR
    User --> Web["Web (Node.js/React)"]
    Web -- CRUD --> PostgreSQL
    Web -- cache search results --> Cache["Cache (Valkey)"]
    Web -- LPUSH job --> Queue["Queue (Valkey)"]
    Queue --> Analyzer["emissions-tracker (Go)"]
    Analyzer --> PostgreSQL
```

- **Web**: Node.js + Express backend serving a React frontend. Handles 
  inventory item CRUD, caches search results in Valkey, and enqueues cost 
  analysis jobs to Valkey.
- **emissions-tracker**: Go worker that polls a Valkey list for jobs, analyzes 
  item components to estimate cost metrics (unit cost, weight, volume, shipping, handling time), and writes results to PostgreSQL.
- **Valkey**: Used as a job queue (between web and analyzer) and a cache (for 
  item search results).
- **PostgreSQL**: Primary data store for inventory items and cost metrics.

## Features

- Inventory item search and browsing
- Dashboard with KPI cards and interactive charts (Recharts)
- Responsive design with Tailwind CSS
- Real-time search functionality
- Detailed item pages with components and handling procedures
- Pin/unpin items for quick access
- Cost analysis (async, powered by Go worker via Valkey queue)
- Search result caching via Valkey
- Docker/Podman support

## Tech stack

- **Web backend**: Node.js, Express, PostgreSQL, ioredis
- **Frontend**: React, TypeScript, Vite, Tailwind CSS, Recharts
- **Cost analyzer**: Go, go-redis, pgx
- **Database**: PostgreSQL
- **Cache/Queue**: Valkey
- **Container**: Docker/Podman

## Running locally with Docker or Podman Compose

The easiest way to run the application locally is with Docker Compose or Podman 
Compose. This starts all services in one command.

1. **Clone the repository**

   ```bash
   git clone <repository-url>
   cd inventory-manager
   ```

2. **Start all services**

   Using Docker Compose:

   ```bash
   docker compose -f apps/docker-compose.yml up --build -d
   ```

   Using Podman Compose:

   ```bash
   podman-compose -f apps/docker-compose.yml up --build -d
   ```

   This starts:

   - **PostgreSQL** database on port 5432
   - **Valkey** on port 6379
   - **Web app** (backend + frontend) on http://localhost:3000
   - **Cost analyzer** (Go worker)

   The database is automatically seeded with sample inventory items on first launch.

3. **View logs**

   ```bash
   docker compose -f apps/docker-compose.yml logs -f
   ```

   Or with Podman:

   ```bash
   podman-compose -f apps/docker-compose.yml logs -f
   ```

4. **Stop all services**

   ```bash
   docker compose -f apps/docker-compose.yml down
   ```

   To also remove the database volume (resets all data):

   ```bash
   docker compose -f apps/docker-compose.yml down -v
   ```
   
## Running as an Aiven Runtime application

The inventory manager demo can be deployed using Aiven Runtime.

This will scan the `docker-compose.yml` file, recognise the necessary 
PostgreSQL and Valkey services and deploy them as Aiven services, and deploy 
the user applications as Runtime applications.

### Fork the repository.

If you just want to run the demo as-is, then that's all you need to do.

If you want to alter what the code does, this is the time to do that as 
well.

If you do make alterations to the code, remember to push to your forks
upstream, so that Aiven Runtime will be able to see your changes.

### Deploy to Aiven Runtime

The following is a summary - check
[the documentation](https://aiven.io/docs/products/apps/deploy-apps)
for the most up-to-date  information.

1. In the [Aiven Console](https://console.aiven.io/) go to your project and
   click **Applications**.
2. Click **Deploy app**.
3. If you haven’t already done so, use **Connect another account** to connect 
   your GitHub account to your Aiven organization.
4. Select your **Account**, your forked repository, and the appropriate branch.
5. Click **Next**.
6. Select the manifest file `apps/docker-compose.yaml` and click **Scan**.
7. On the card for the Kafka service, click the paired arrows icon
   ![paired arrows](images/paired-arrows-icon.png), and choose the Kafka
   service you created earlier.
8. On the cards for the `inventory-manager-web` and 
   `inventory-manager-emissions-tracker`, click the pen icon
   ![pen icon](images/pen-icon.png) to edit each configuration.

    - Check that the application **Name** is recognisable, and edit it if
      necessary. The aim is to have a name that you’ll recognise and be able
      to find later.
    - Alter the service settings to choose an appropriate plan. You 
      probably want all of the application services in the same cloud and 
      region.

9. On the cards for the `inventory-manager-db`, `inventory-manager-queue` and
   `inventory-manager-web-cache` services, either

   - click the pen icon ![pen icon](images/pen-icon.png) to edit each
     configuration.

     - Check that the service **Name** is recognisable, and edit it if
       necessary. The aim is to have a name that you’ll recognise and be able
       to find later.
     - Alter the service settings to choose an appropriate plan. You
       probably want all of the application services in the same cloud and
       region.

   - **or** click the paired arrows icon
    ![paired arrows](images/paired-arrows-icon.png), and choose an existing 
    Aiven service.


10. To deploy the app services, click **Deploy**.

The applications and Aiven services will start to build.

For the manager and emissions tracker services, see their progress in the 
appropriate **Build logs** tab. 
Once each is running, their runtime logs can be seen in the **Runtime log**.

## Database seeding

The database is automatically seeded with sample inventory items on first
launch. To re-seed it manually:

1. Set the `DATABASE_URL` environment variable to the correct values - for instance, if you're running PostgreSQL locally, this would be:
   ```bash
   expost DATABASE_URL postgresql://mgr_user:mgr_pass@localhost:5432/mgr_db
   ```

2. Then run:
   ```bash
   cd apps/web/backend
   npm run db:seed
   ```

## API endpoints

- `GET /health` - Health check
- `GET /api/items` - Get all items (supports `search`, `limit`, `offset` query parameters)
- `GET /api/items/:id` - Get item by ID
- `POST /api/items/:id/pin` - Pin an item
- `POST /api/items/:id/unpin` - Unpin an item
- `POST /api/items/:id/analyze` - Enqueue cost analysis (via queue -> Go worker)
- `GET /api/items/:id/metrics` - Get cost metrics for an item

## Environment variables

These are configured automatically when using Docker/Podman Compose, or when deploying with Aiven Runtime. Only set them manually for local development.

| Variable | Description                             | Default (Compose) |
|---|-----------------------------------------|---|
| `DATABASE_URL` | PostgreSQL connection string (required) | `postgresql://mgr_user:mgr_pass@mgr-db:5432/mgr_db` |
| `CACHE_REDIS_URL` | Valkey URL for caching (web only)       | `redis://mgr-web-cache:6379` |
| `QUEUE_REDIS_URL` | Valkey URL for job queue                | `redis://mgr-queue:6379` |
| `SERVER_PORT` | Backend server port                     | `3000` |
| `FRONTEND_DEV_PORT` | Frontend dev server port                | `5000` |
| `NODE_ENV` | Environment mode                        | `production` |

## License

MIT License
