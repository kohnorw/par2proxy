# par2proxy

**par2proxy** sits between Sonarr/Radarr and Decypharr. It acts as a fake SABnzbd download client, intercepts NZB submissions, fetches par2 recovery data via NNTP, fixes missing segments, then forwards the repaired NZB to Decypharr for streaming.

**It never downloads the media files.** Only par2 data is fetched (~1–5% of the NZB size).

---

## How it works

```
Sonarr/Radarr
    │  NZB (SABnzbd API)
    ▼
par2proxy :8383
    │  1. Fetches par2 data via NNTP
    │  2. If Decypharr reports missing segments → substitutes them with recovery blocks
    │  3. Forwards repaired NZB
    ▼
Decypharr :8282
    │  Streams media via NNTP
    ▼
/mnt/downloads/sonarr/   (symlink)
```

par2proxy polls Decypharr until the job completes, then reports success to Sonarr so the import always has files ready.

---

## Deploy

```bash
# Extract the tar
tar -xzf par2proxy.tar.gz
cd par2proxy

# Start (uses pre-built binary — no Go needed)
docker-compose up -d --build

# Open the web UI to configure everything
open http://localhost:8383
```

> **Note:** Uses a pre-built `linux/amd64` binary in the tar. The Dockerfile is a single-stage Alpine image — no multi-stage build — which avoids the `ContainerConfig` crash in docker-compose v1.29.2.

---

## First-run setup

On first start, open **http://your-host:8383** and go to **Settings**. Fill in:

### par2proxy
| Field | Description |
|---|---|
| **API key** | Any string — put this same value in Sonarr/Radarr's download client API key field |
| **Max par2 fetch (MB)** | How much par2 data to fetch per NZB. Default 100 MB. |

### Sonarr / Radarr
Add one entry per arr. Each needs:

| Field | Example | Description |
|---|---|---|
| **Name** | `Sonarr` | Display label |
| **URL** | `http://sonarr:8989` | Arr host URL |
| **API key** | `abc123...` | From Sonarr → Settings → General → API Key |
| **Category** | `sonarr` | Subfolder under Decypharr's complete dir. Must match what you set in the arr's download client. |

> These credentials are forwarded to Decypharr's SABnzbd API so it knows which arr sent the NZB.

### Decypharr
| Field | Description |
|---|---|
| **URL** | `http://decypharr:8282` |
| **API token** | From Decypharr → Settings → Auth (Bearer token — only needed for the repair library sweep feature) |

### NNTP providers
Add your Usenet provider(s). Each has host, port, username, password, TLS toggle, and max connections. Click **Test** to verify auth before saving.

Click **Save settings** — config is written to `/docker/par2proxy/config/config.json` and persists across restarts.

---

## Sonarr / Radarr download client settings

**Settings → Download Clients → + → SABnzbd**

| Field | Value |
|---|---|
| **Host** | `par2proxy` (container name) or your host IP |
| **Port** | `8383` |
| **API Key** | Must match what you set in par2proxy Settings → API key |
| **Category** | `sonarr` (or `radarr`) — must match the category in par2proxy's arr config |
| **URL Base** | *(leave empty)* |

Click **Test** → **Save**.

---

## What par2proxy can and can't fix

**Can fix:** NZBs where some media segments are missing but the par2 recovery files are still on Usenet. The par2 data is used to substitute missing segment IDs with recovery block IDs, which Decypharr streams instead.

**Cannot fix:** NZBs where the par2 files themselves have also expired (old releases, typically 5+ years). In this case the error will say `release has missing/expired segments — par2 repair not possible`. Find a fresher re-encode or different indexer source.

---

## Web UI

Open **http://par2proxy:8383/**

| Tab | Description |
|---|---|
| **Queue** | Active NZB jobs — shows phase (Checking par2, Downloading %, etc.), bad/fixed segment counts |
| **History** | Completed and failed jobs |
| **Repair library** | Bulk repair of Decypharr's existing library |
| **NNTP providers** | Live connection status — click Recheck to test now |
| **Settings** | Configure all connections |

### Repair library

| Button | What it does |
|---|---|
| **Scan broken** | Queries Decypharr's repair health API and lists broken entries |
| **Repair all** | Fetches par2 for each broken entry, rewrites NZBs, resubmits |
| **Decypharr sweep** | Triggers Decypharr's own built-in repair run |

---

## Files

```
/docker/par2proxy/config/config.json   ← saved settings (bind-mounted into container)
```

Settings saved from the web UI persist here across container restarts and rebuilds.

---

## Rebuilding from source

If you have Go 1.22+ installed:

```bash
cd par2proxy
GOOS=linux GOARCH=amd64 CGO_ENABLED=0 go build -ldflags="-s -w" -o par2proxy ./cmd/par2proxy
docker-compose up -d --build
```

No external dependencies — pure Go stdlib.

---

## Troubleshooting

**docker-compose `ContainerConfig` error:**
```bash
docker rm -f par2proxy
docker-compose up -d --build
```
This is a docker-compose v1.29.2 bug triggered by recreating an existing container. Removing it first clears the bad metadata.

**Sonarr test connection fails:**
- URL Base must be **empty** in Sonarr's download client settings
- API key must exactly match par2proxy Settings → API key
- Check `docker logs par2proxy | grep sabnzbd` for the actual request

**Files going to `/mnt/complete` instead of `/mnt/complete/sonarr`:**
- Make sure the **Category** in par2proxy's arr config matches the **Category** in Sonarr's download client settings
- Both must be the same string, e.g. `sonarr`

**par2 fetch failing:**
- Check NNTP providers tab — providers must show Online
- Old releases (5+ years) often have expired par2 files — this is unrecoverable

**Decypharr reports 500 / missing segments:**
- par2proxy will automatically retry repair up to 5 times
- If it still fails, the release has expired par2 data

---

## Project layout

```
par2proxy/
├── cmd/par2proxy/       # main entrypoint, HTTP routing
├── internal/
│   ├── config/          # JSON config + env var loading
│   ├── nntp/            # pooled NNTP client, yEnc decode
│   ├── nzb/             # NZB XML parser/marshaller
│   ├── par2/            # PAR2 packet parser
│   ├── repair/          # par2 repair engine
│   ├── sabnzbd/         # fake SABnzbd API + Decypharr forwarding
│   ├── settings/        # settings API (GET/POST + test endpoints)
│   ├── sweeper/         # bulk library repair
│   └── ui/              # embedded web dashboard
├── par2proxy            # pre-built linux/amd64 binary
├── Dockerfile
├── docker-compose.yml
└── README.md
```
