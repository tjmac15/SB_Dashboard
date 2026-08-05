# Vestas SAP Security & Compliance Dashboard

A single-page web dashboard that visualizes weekly SAP security and compliance
scores — sourced from SecurityBridge scan data — across all SAP systems,
broken down by Area of Responsibility (Authorization, SAP Basis, Identity and
Access, Data Protection, Development, Integration, Application Controls,
SoD, Operating System, Database, SB-Admin).

It replaces manual review of static exports with a live, interactive view:
score trends over time, week-over-week deltas, per-system rankings, and
which Areas of Responsibility are currently in a Warning or Critical state.

## How it works

- **Frontend only** — a single `index.html` file (HTML/CSS/vanilla JS). No
  build step, no backend server, no database.
- **Auth** — signs the user in with their own Microsoft 365 identity via
  [MSAL.js](https://github.com/AzureAD/microsoft-authentication-library-for-js)
  (delegated permissions only — no service account, no secrets embedded in
  the code).
- **Data source** — reads the `SB_Scores` SharePoint list through
  [Microsoft Graph](https://learn.microsoft.com/en-us/graph/overview) using
  the signed-in user's own read permissions.
- **Charts** — [Chart.js](https://www.chartjs.org/) (loaded from CDN).

Because it only reads data through supported, delegated Microsoft Graph
calls — no direct SAP connection, no custom ABAP, no database hacks — it
stays outside the SAP "core" entirely (see Clean Core, if that term comes
up in review).

## Pages

| Page | Contents |
|---|---|
| **Overview** | Score Heatmap (by system, with a "Latest rating summary" panel), Overall Posture Trend, Trend Lines by AoR, Weakest AoRs, Critical by System |
| **Week-over-Week & Sparklines** | Week-over-week score deltas and 4-week sparkline trends per Area of Responsibility |

Click any Area of Responsibility row in a table to highlight it.

## Running it locally (no Docker)

Since it's a static file, any local web server works. From the folder
containing `index.html`:

```bash
python3 -m http.server 8080
# or
npx serve .
```

Then open `http://localhost:8080`.

> Opening `index.html` directly via `file://` will **not** work — MSAL's
> redirect flow requires a real `http(s)://` origin.

## Running it with Docker

See [`DOCKER.md`](./DOCKER.md) for the full container build/run/deploy guide.

Quick version:

```bash
docker build -t sb-dashboard:latest .
docker run --rm -p 8080:8080 sb-dashboard:latest
```

## Configuration

App registration and data-source settings live near the top of the
`<script>` block in `index.html`:

| Constant | Purpose |
|---|---|
| `CLIENT_ID` | Azure AD App Registration (public client) ID used for MSAL sign-in |
| `TENANT_ID` | Azure AD tenant ID the app authenticates against |
| `SP_SITE` | SharePoint site path hosting the `SB_Scores` list |
| `SP_LIST` | SharePoint list name (`SB_Scores`) |
| `GRAPH` | Microsoft Graph API base URL |

If you fork/redeploy this under a different Azure AD app registration or a
different SharePoint site, update these four values.

The Azure AD App Registration's **Redirect URI** (under
Authentication → Single-page application) must include whatever URL you're
serving the app from (e.g. `http://localhost:8080` for local dev, plus your
production URL).

## File structure

```
.
├── index.html          # the entire application
├── vestas-logo.jpg      # Vestas wordmark, referenced by index.html — keep alongside it
├── Dockerfile           # container build definition
├── nginx.conf           # nginx server config used inside the container
├── .dockerignore
├── README.md            # this file
└── DOCKER.md            # container-specific documentation
```

## Known limitations

- Single HTML file — fine for its current size, but worth splitting into
  separate CSS/JS files if it keeps growing.
- No automated tests.
- Requires the signed-in user to have read access to the `SB_Scores`
  SharePoint list; there's no fallback/error messaging beyond a generic
  "no data" state if permissions are missing.
