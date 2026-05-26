# par2proxy

**par2proxy** is a Docker service that sits between Sonarr/Radarr and Decypharr. It acts as a fake SABnzbd download client, intercepts NZB submissions, fetches par2 recovery data, fixes missing or corrupt segments, then forwards the repaired NZB to Decypharr for streaming.

**It never downloads the media files.** Only par2 data (~1–5% of the total NZB size) is fetched.

---

## How it works

```
Sonarr/Radarr
    │  NZB submission (SABnzbd API)
    ▼
par2proxy :8383
    │
    ├─ 1. Fetches par2 index + vol files via NNTP
    ├─ 2. STATs every media segment (no download, just presence check)
    ├─ 3. Rewrites NZB: missing segment IDs → par2 recovery article IDs
    │
    │  Repaired NZB (SABnzbd API)
    ▼
Decypharr :8282
    │
    └─ Streams media via NNTP as normal
       Symlink lives in /mnt/decypharr/sonarr/ (or wherever Decypharr mounts)
```

par2proxy polls Decypharr until the job is fully complete before reporting success to Sonarr, so imports always have the files ready.

---

## Quick start

```bash
git clone https://github.com/yourname/par2proxy
cd par2proxy
cp docker-compose.yml docker-compose.override.yml
# Edit docker-compose.override.yml with your credentials
docker compose up -d --build
```

Open **http://localhost:8383** to see the dashboard.

---

## Sonarr / Radarr setup

**Settings → Download Clients → + → SABnzbd**

| Field    | Value                              |
|----------|------------------------------------|
| Name     | par2proxy                          |
| Host     | `par2proxy` (or your host IP)      |
| Port     | `8383`                             |
| API Key  | value of `API_KEY` env var         |
| Category | `sonarr` (or `radarr`)             |
| URL Base | *(leave empty)*                    |

Click **Test** → **Save**.

Set par2proxy at **higher priority** than any direct Decypharr download client so it gets first pick of Usenet NZBs.

---

## Environment variables

### This service

| Variable        | Default        | Description                                  |
|-----------------|----------------|----------------------------------------------|
| `LISTEN_ADDR`   | `:8383`        | Address par2proxy binds to                   |
| `API_KEY`       | `par2proxy`    | SABnzbd API key that Sonarr uses to auth here |

### Decypharr connection

| Variable                | Default                   | Description                                                      |
|-------------------------|---------------------------|------------------------------------------------------------------|
| `DECYPHARR_URL`         | `http://decypharr:8282`   | Decypharr base URL                                               |
| `DECYPHARR_API_KEY`     | *(empty)*                 | Decypharr Bearer token — from Settings → Auth. Used for REST API (`/api/repair/health`, `/api/browse/`) and for fetching `complete_dir` at startup |
| `DECYPHARR_USERNAME`    | *(empty)*                 | Your Arr host URL, e.g. `http://sonarr:8989` (Decypharr SABnzbd docs) |
| `DECYPHARR_PASSWORD`    | *(empty)*                 | Your Arr API token (Decypharr SABnzbd docs)                      |

> **Note on `DECYPHARR_API_KEY`:** at startup par2proxy calls `GET /sabnzbd/api?mode=config` to fetch the real `complete_dir` from Decypharr, retrying every 5 seconds for up to 50 seconds. This means Docker Compose startup order doesn't matter — Decypharr just needs to be up before the first NZB arrives.

### NNTP providers

**Single provider:**

| Variable           | Default | Description               |
|--------------------|---------|---------------------------|
| `NNTP_HOST`        | —       | NNTP hostname (required)  |
| `NNTP_PORT`        | `563`   | NNTP port                 |
| `NNTP_USER`        | —       | NNTP username (required)  |
| `NNTP_PASS`        | —       | NNTP password (required)  |
| `NNTP_TLS`         | `true`  | Use TLS                   |
| `NNTP_CONNECTIONS` | `8`     | Max connections — keep below Decypharr's limit |

**Multiple providers** (failover, tried in order):

```env
NNTP_0_HOST=news.newshosting.com
NNTP_0_PORT=563
NNTP_0_USER=user1
NNTP_0_PASS=pass1
NNTP_0_CONNECTIONS=8

NNTP_1_HOST=news.eweka.nl
NNTP_1_PORT=563
NNTP_1_USER=user2
NNTP_1_PASS=pass2
NNTP_1_CONNECTIONS=4
```

### Behaviour

| Variable       | Default | Description                                          |
|----------------|---------|------------------------------------------------------|
| `MAX_PAR2_MB`  | `100`   | Max MB of par2 data to fetch per NZB                 |
| `VERBOSE`      | `false` | Verbose logging                                      |

---

## Web UI

Open **http://par2proxy:8383/** to see:

- **Queue** — active NZB jobs with repair progress, bad/fixed segment counts
- **History** — completed and failed jobs
- **Repair library** — bulk repair of Decypharr's existing usenet library
- **NNTP providers** — live connection status per provider

### Repair library tab

Three actions:

| Button | What it does |
|--------|-------------|
| **Scan broken** | Calls `GET /api/repair/health?status=broken` on Decypharr and lists what it already knows is broken — no repair yet |
| **Repair all** | Starts a full sweep: fetches par2 data for every broken entry, rewrites its NZB, resubmits through par2proxy → Decypharr |
| **Decypharr sweep** | Triggers `POST /api/repair/run` on Decypharr directly — its own built-in health checker (faster, no par2 fetch) |

---

## API endpoints

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/sabnzbd/api?mode=addfile` | Submit NZB (Sonarr/Radarr) |
| `GET`  | `/sabnzbd/api?mode=queue`   | Queue status |
| `GET`  | `/sabnzbd/api?mode=history` | Completed jobs |
| `GET`  | `/api/status`               | JSON status for web UI |
| `POST` | `/api/sweep/start`          | Start library sweep |
| `POST` | `/api/sweep/stop`           | Cancel sweep |
| `GET`  | `/api/sweep/status`         | Live sweep progress |
| `GET`  | `/api/sweep/broken`         | Broken entries from Decypharr |
| `POST` | `/api/sweep/decypharr`      | Trigger Decypharr's built-in repair |
| `GET`  | `/health`                   | Health check → `ok` |

---

## Docker Compose example

```yaml
services:
  par2proxy:
    build: .
    container_name: par2proxy
    restart: unless-stopped
    ports:
      - "8383:8383"
    environment:
      LISTEN_ADDR: ":8383"
      API_KEY: "changeme"

      DECYPHARR_URL: "http://decypharr:8282"
      DECYPHARR_API_KEY: "your-decypharr-api-token"
      DECYPHARR_USERNAME: "http://sonarr:8989"
      DECYPHARR_PASSWORD: "your-sonarr-api-token"

      NNTP_HOST: "news.newshosting.com"
      NNTP_PORT: "563"
      NNTP_USER: "your-username"
      NNTP_PASS: "your-password"
      NNTP_TLS: "true"
      NNTP_CONNECTIONS: "8"

      MAX_PAR2_MB: "100"
      VERBOSE: "false"
    networks:
      - arr-net

networks:
  arr-net:
    external: true
```

---

## Decypharr config note

Keep `skip_repair: false` in Decypharr's usenet config. par2proxy ensures recovery article IDs are present in the NZB; Decypharr's native repair layer does the Reed-Solomon reconstruction during streaming.

```json
{
  "usenet": {
    "skip_repair": false
  }
}
```

---

## Building from source

```bash
go build -o par2proxy ./cmd/par2proxy
./par2proxy
```

Requires Go 1.22+. No external dependencies — pure stdlib.

---

## Health check

```bash
curl http://localhost:8383/health
# → ok
```

---

## Project layout

```
par2proxy/
├── cmd/par2proxy/       # main entrypoint
├── internal/
│   ├── config/          # environment + JSON config loading
│   ├── nntp/            # pooled NNTP client with yEnc decode
│   ├── nzb/             # NZB XML parser and marshaller
│   ├── par2/            # PAR2 packet parser (IFSC checksums, recovery blocks)
│   ├── repair/          # par2 repair engine: STAT segments, rewrite NZB
│   ├── sabnzbd/         # fake SABnzbd HTTP API + Decypharr status polling
│   ├── sweeper/         # bulk library repair via Decypharr REST API
│   └── ui/              # embedded web dashboard (dashboard.html + JSON API)
├── Dockerfile
├── docker-compose.yml
└── README.md
```

---

## License

MIT
