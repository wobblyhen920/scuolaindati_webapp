# scuolaindati-web

[![React 18](https://img.shields.io/badge/React-18.2-61DAFB.svg?logo=react)](https://react.dev/)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.4-3178C6.svg?logo=typescript)](https://www.typescriptlang.org/)
[![Go 1.22](https://img.shields.io/badge/Go-1.22-00ADD8.svg?logo=go)](https://go.dev/)
[![DuckDB](https://img.shields.io/badge/DuckDB-embedded-FFF000.svg)](https://duckdb.org/)
[![Vite](https://img.shields.io/badge/Vite-5-646CFF.svg?logo=vite)](https://vitejs.dev/)
[![Chakra UI](https://img.shields.io/badge/Chakra_UI-2.10-319795.svg)](https://chakra-ui.com/)

**Webapp e API per esplorare i dati sulle scuole italiane.**

🔗 **Live:** [scuolaindati.it](https://scuolaindati.it)

# scuolaindati-scraper
🔗 [github.com/wobblyhen920/scuolaindati_scraper/tree/main](https://github.com/wobblyhen920/scuolaindati_scraper/tree/main) 

---

## Architettura

```
┌─────────────────────────────────────────────────────────────────────┐
│                           BROWSER                                   │
│  React + Chakra UI + Plotly + Zustand                               │
└───────────────────────────────┬─────────────────────────────────────┘
                                │ HTTP
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                          NGINX                                      │
│  • SSL termination                                                  │
│  • Rate limiting (10 req/s API, 30 req/s general)                   │
│  • Static files + SPA fallback                                      │
│  • /releases/* → file server (CSV/Parquet)                          │
│  • /api/* → reverse proxy                                           │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      GO API SERVER                                  │
│  Chi router + DuckDB embedded + LRU cache                           │
│  Query DSL → SQL translation                                        │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    PARQUET FILES                                    │
│  releases/latest/ALL/wide.parquet                                   │
│  releases/latest/areas/{NORD,CENTRO,SUD,...}/wide.parquet           │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Struttura del repository

```
scuolaindati-web/
│
├── web/                       # Frontend React
│   ├── src/
│   │   ├── app/
│   │   │   ├── api/           # Client API
│   │   │   ├── shell/         # Layout (Sidebar, Topbar)
│   │   │   ├── state/         # Zustand store
│   │   │   ├── theme/         # Chakra theme + Plotly theme
│   │   │   └── routes.tsx
│   │   ├── pages/
│   │   │   ├── Explore/       # Explorer con filtri e grafici
│   │   │   ├── Releases/      # Download release
│   │   │   ├── ApiDocs/       # Swagger UI
│   │   │   ├── Info/          # Homepage
│   │   │   └── Wiki/          # Documentazione
│   │   └── components/
│   ├── public/
│   │   ├── content/           # Markdown e CSV
│   │   └── openapi.yaml
│   ├── package.json
│   └── vite.config.ts
│
├── api/                       # Backend Go
│   ├── cmd/server/main.go
│   ├── internal/
│   │   ├── server/            # HTTP handlers
│   │   ├── query/             # DSL → SQL
│   │   ├── duck/              # DuckDB manager
│   │   ├── cache/             # LRU cache
│   │   └── config/
│   ├── openapi.yaml
│   ├── go.mod
│   └── go.sum
│
├── deploy/                    # Configurazioni deploy
│   ├── nginx-scuolaindati.conf
│   ├── deploy.sh
│   └── *.md                   # Documentazione deploy
```

---

## Quick start

### Requisiti

- **Node.js** >= 18
- **Go** >= 1.22
- **DuckDB** (incluso via go-duckdb)

### Frontend

```bash
cd web
npm install
npm run dev        # Dev server su http://localhost:5173
npm run build      # Build produzione in dist/
```

### Backend

```bash
cd api

# Configurazione (variabili d'ambiente)
export RELEASE_ROOT=/path/to/releases
export LISTEN_ADDR=:8080

# Build e run
go build -o scuolaindati-api ./cmd/server
./scuolaindati-api
```

### Variabili d'ambiente API

| Variabile | Default | Descrizione |
|-----------|---------|-------------|
| `LISTEN_ADDR` | `:8080` | Indirizzo di ascolto |
| `RELEASE_ROOT` | `./releases` | Directory con le release |
| `QUERY_TIMEOUT` | `30s` | Timeout query DuckDB |
| `MAX_ROWS` | `10000` | Limite righe per query |
| `CACHE_SIZE` | `100` | Dimensione cache LRU |

---

## API

### Endpoints

| Metodo | Path | Descrizione |
|--------|------|-------------|
| `GET` | `/api/health` | Healthcheck |
| `GET` | `/api/releases` | Lista release e pacchetti |
| `GET` | `/api/schema?pkg=ALL` | Schema colonne di un pacchetto |
| `GET` | `/api/preview?pkg=ALL&limit=100` | Anteprima righe |
| `POST` | `/api/query` | Query DSL completa |

### Query DSL

```json
{
  "pkg": "ALL",
  "release": "latest",
  "select": ["CODICE_SCUOLA", "denominazione", "comune"],
  "where": [
    { "col": "provincia", "op": "=", "val": "RM" },
    { "col": "tipoDiIstruzione", "op": "in", "val": ["PC", "PS"] }
  ],
  "order_by": [{ "col": "denominazione", "dir": "asc" }],
  "limit": 100,
  "offset": 0
}
```

**Operatori where supportati:**
- `=`, `!=`, `<`, `>`, `<=`, `>=`
- `in`, `not_in`
- `like`, `ilike`
- `is_null`, `is_not_null`

**Aggregazioni:**
```json
{
  "pkg": "ALL",
  "select": ["provincia"],
  "group_by": "provincia",
  "aggs": [
    { "fn": "count", "as": "n_scuole" },
    { "fn": "avg", "col": "10_studenti_iscritti", "as": "media_iscritti" }
  ]
}
```

### Esempio curl

```bash
# Schema
curl "https://scuolaindati.it/api/schema?pkg=ALL"

# Query
curl -X POST "https://scuolaindati.it/api/query" \
  -H "Content-Type: application/json" \
  -d '{"pkg":"ALL","select":["CODICE_SCUOLA","denominazione"],"limit":10}'
```

---

## Frontend

### Stack

- **React 18** + **TypeScript**
- **Chakra UI** - Componenti e theming
- **Zustand** - State management
- **Plotly.js** - Grafici
- **React Router** - Routing
- **Vite** - Build tool

### Pagine

| Path | Componente | Descrizione |
|------|------------|-------------|
| `/` | `InfoPage` | Homepage con overview progetto |
| `/explore` | `ExplorePage` | Explorer interattivo (filtri, variabili, grafici) |
| `/releases` | `ReleasesPage` | Download release complete |
| `/api` | `ApiDocsPage` | Documentazione API (Swagger UI) |
| `/wiki/*` | `WikiPage` | Documentazione markdown |

### State management

```typescript
// store.ts
interface AppState {
  release: string      // "latest" o timestamp
  pkg: string          // "ALL" o "areas/NORD" etc
  setRelease: (r: string) => void
  setPkg: (p: string) => void
}
```

---

## Deploy

### Produzione

```bash
# 1. Build frontend
cd web && npm run build

# 2. Build API
cd api && go build -o scuolaindati-api ./cmd/server

# 3. Deploy files
rsync -avz web/dist/ server:/var/www/html/scuolaindati.it/
rsync -avz api/scuolaindati-api server:/opt/scuolaindati/

# 4. Systemd service per API
# 5. Nginx config (vedi deploy/nginx-scuolaindati.conf)
```

### Rate limiting

Configurato già in Nginx:
- **API:** 10 req/s per IP (burst 20)
- **Generale:** 30 req/s per IP (burst 50)

### SSL

Let's Encrypt con certbot:
```bash
certbot --nginx -d scuolaindati.it -d www.scuolaindati.it
```

---

## Sviluppo

### Struttura Explorer (pagina principale)

L'explorer funziona in 3 step:

1. **Filtra scuole** - Tipo istruzione, provincia, comune, CAP, ricerca testo
2. **Seleziona variabili** - Checkbox con 120+ variabili disponibili
3. **Consulta risultati** - Anteprima tabella, grafici automatici, download CSV

### Aggiungere una pagina

```tsx
// 1. Crea src/pages/NuovaPagina/NuovaPagina.tsx
export default function NuovaPagina() {
  return <Box>...</Box>
}

// 2. Aggiungi route in src/app/routes.tsx
{ path: '/nuova', element: <NuovaPagina /> }

// 3. Aggiungi link in src/app/shell/SidebarNav.tsx
```

### API client

```typescript
import { apiQuery, apiGetSchema } from './app/api/client'

// Query
const result = await apiQuery({
  pkg: 'ALL',
  select: ['CODICE_SCUOLA', 'denominazione'],
  where: [{ col: 'provincia', op: '=', val: 'RM' }],
  limit: 100
})

// Schema
const schema = await apiGetSchema({ pkg: 'ALL', release: 'latest' })
```

---

## File statici

### Releases (download)

I file Parquet/CSV delle release sono serviti direttamente da Nginx:

```
/releases/latest/ALL/wide.csv
/releases/latest/ALL/wide.parquet
/releases/latest/areas/NORD/wide.csv
...
```

### Contenuti markdown

```
/content/wiki/index.md
/content/wiki/variabili.md
/content/mappatura_variabili.csv
```

---

## Licenza

**[CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/deed.it)**

---

## Contatti

📧 **scuolaindati@proton.me**
