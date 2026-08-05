# Docker guide

This app is a static site (`index.html` + `vestas-logo.jpg`), so the
container is just a web server — no application runtime, no build step.

## The image

- **Base:** [`nginxinc/nginx-unprivileged`](https://hub.docker.com/r/nginxinc/nginx-unprivileged)
  (Alpine variant), not the standard `nginx` image.
- **Why unprivileged:** it runs as a **non-root user** and listens on
  **port 8080** instead of port 80. Most corporate Kubernetes clusters
  (VKS included) enforce a security context that disallows running as root
  and disallows binding to privileged ports (<1024) — this image satisfies
  that out of the box, so you don't need to fight `runAsNonRoot` /
  `runAsUser` pod security policies later.
- **What's copied in:** `index.html`, `vestas-logo.jpg`, and a custom
  `nginx.conf` that replaces the image's default server block.

## Build

From the folder containing `Dockerfile`, `nginx.conf`, `index.html`, and
`vestas-logo.jpg`:

```bash
docker build -t sb-dashboard:latest .
```

## Run locally

```bash
docker run --rm -p 8080:8080 sb-dashboard:latest
```

Open `http://localhost:8080`. Stop with `Ctrl+C` (the `--rm` flag cleans up
the container automatically).

To run it detached (in the background):

```bash
docker run -d --name sb-dashboard -p 8080:8080 sb-dashboard:latest
docker logs -f sb-dashboard      # view logs
docker stop sb-dashboard         # stop it
```

## Health check

The image defines a `HEALTHCHECK` that curls `/` every 30s. There's also a
dedicated lightweight endpoint at **`/healthz`** (returns `200 ok`) — use
this one for Kubernetes liveness/readiness probes instead of `/`, since it
doesn't require loading the full page or its CDN dependencies (MSAL,
Chart.js) to succeed.

## Tag and push to a registry

```bash
docker tag sb-dashboard:latest <your-registry>/sb-dashboard:<tag>
docker push <your-registry>/sb-dashboard:<tag>
```

Replace `<your-registry>` with wherever this needs to land (Azure Container
Registry, an internal VKS-associated registry, GitHub Container Registry,
etc.) and `<tag>` with a version or `latest`.

## Deploying to Kubernetes (VKS)

A minimal Deployment + Service would look like:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sb-dashboard
spec:
  replicas: 2
  selector:
    matchLabels:
      app: sb-dashboard
  template:
    metadata:
      labels:
        app: sb-dashboard
    spec:
      containers:
        - name: sb-dashboard
          image: <your-registry>/sb-dashboard:<tag>
          ports:
            - containerPort: 8080
          securityContext:
            runAsNonRoot: true
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 15
          readinessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 3
            periodSeconds: 10
          resources:
            requests:
              cpu: "50m"
              memory: "32Mi"
            limits:
              cpu: "200m"
              memory: "64Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: sb-dashboard
spec:
  selector:
    app: sb-dashboard
  ports:
    - port: 80
      targetPort: 8080
```

> `readOnlyRootFilesystem: true` works here because nginx only needs to
> *read* the static files and its own config — it doesn't write to disk.
> If your cluster's nginx image needs writable `/var/cache/nginx` or
> `/var/run`, mount those as `emptyDir` volumes; test this before relying
> on it in production, since exact behavior can vary by base image version.

You'd still need an Ingress (or whatever VKS uses — check with your
platform team) in front of the Service to expose it outside the cluster
with a real hostname and TLS.

## Environment-specific redirect URI

MSAL's sign-in flow needs the app's **exact runtime URL** registered as a
Redirect URI in Azure AD (Authentication → Single-page application). If you
deploy to a new hostname (e.g. a new Ingress URL), add that URL there —
otherwise sign-in will fail with a redirect URI mismatch error, not a
container/Docker problem.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| Container exits immediately | Check `docker logs <container>` — usually a syntax error in `nginx.conf` |
| `403 Forbidden` on load | File permissions issue copying into the image, or `index.html` missing from the build context |
| Blank page / MSAL errors in browser console | Redirect URI not registered for this URL in Azure AD, or the site isn't served over the URL MSAL expects |
| Logo doesn't load | `vestas-logo.jpg` wasn't in the build context, or wasn't copied — check the `COPY` lines in `Dockerfile` |
| `/healthz` returns 404 | `nginx.conf` wasn't picked up — confirm it's copied to `/etc/nginx/conf.d/default.conf` and the image was rebuilt after any edits |

I haven't been able to build or run this image myself (no Docker available
in this environment) — if you hit an error not listed above, paste it over
and I'll help debug it.
