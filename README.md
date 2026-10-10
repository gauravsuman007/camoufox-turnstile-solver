# camoufox-turnstile-solver

A minimal HTTP sidecar that clears a **Cloudflare Turnstile widget embedded in a normal page** and returns the token. Built because FlareSolverr and byparr only engage on Cloudflare's own full-page interstitial — an embedded widget on a 200 OK page is invisible to them.

Uses [camoufox](https://github.com/daijro/camoufox) (patched Firefox, arm64-native). Runs on `linux/amd64` and `linux/arm64`.

## Pull

```bash
docker pull ghcr.io/gauravsuman007/camoufox-turnstile-solver:latest
```

## Run

```bash
docker run -d --name turnstile-solver --restart unless-stopped \
  -p 8193:8000 ghcr.io/gauravsuman007/camoufox-turnstile-solver:latest
```

Health check: `curl http://localhost:8193/healthz` → `{"status":"ok"}`.

## API

```http
POST /solve
Content-Type: application/json

{
  "url":             "https://example.com/register",
  "agree_click":     "a[onclick*=\"go-to-reg\"]",
  "widget_selector": ".cf-turnstile",
  "max_timeout":     90
}
```

Response:

```json
{
  "ok":      true,
  "token":   "0.<base64>...",
  "cookies": [{"name":"...", "value":"...", "domain":"...", ...}],
  "html":    "<the final page HTML>",
  "elapsed": 1.3
}
```

Optional `pre_post` for a form submit before navigating to `url`:

```json
{
  "url": "...",
  "pre_post": {
    "url": "https://example.com/terms",
    "data": "reg_agreed=1"
  },
  ...
}
```

The caller does the subsequent form submit; this sidecar only extracts the token.

## Docker Compose example

```yaml
services:
  your-app:
    image: your-app:latest
    environment:
      TURNSTILE_SIDECAR_URL: http://turnstile-solver:8000

  turnstile-solver:
    image: ghcr.io/gauravsuman007/camoufox-turnstile-solver:latest
    container_name: turnstile-solver
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "python3", "-c",
             "import urllib.request; urllib.request.urlopen('http://localhost:8000/healthz', timeout=3)"]
      interval: 30s
      timeout: 5s
      start_period: 120s
      retries: 3
```

## Notes

- One Firefox process per `/solve` call, serialised through an in-process lock. Scale by replicas rather than concurrent calls to one instance.
- `exclude_addons=["UBO"]` is critical: camoufox ships uBlock Origin, which blocks `challenges.cloudflare.com` by default.
- The reliable click target is the `.cf-turnstile` **div**, not its iframe — Turnstile injects the iframe with no `src` attribute so an iframe-src selector misses it, but the div's `getBoundingClientRect` is always valid. Click 30 px from its left edge, vertical centre.
- `networkidle` never fires while Turnstile is on the page (its own long-poll keeps the network busy) — the sidecar uses `load` wait state then polls the hidden input.
