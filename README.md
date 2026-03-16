# API via Cloudflare Tunnel (edge-3090-1)

Expose APIs at `https://inference.taixingai.com` and `https://embedding.taixingai.com` using one Cloudflare Tunnel — no Caddy, no open ports. APIs are protected by Cloudflare Access; requests require a valid service token.

| Hostname | Backend |
|----------|---------|
| `inference.taixingai.com` | `http://localhost:8000` |
| `embedding.taixingai.com` | `http://localhost:8001` |

## Prerequisites

- Cloudflare account, domain `taixingai.com` on Cloudflare DNS
- Cloudflare Zero Trust / Access (for API token authentication)
- Inference API on `http://localhost:8000` (e.g. vLLM, chat)
- Embedding API on `http://localhost:8001` (e.g. `/v1/embeddings`)

## One-time setup

### 1. Install cloudflared

- **Linux (deb):**  
  `curl -L --output cloudflared.deb https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb && sudo dpkg -i cloudflared.deb`
- **macOS:**  
  `brew install cloudflared`

### 2. Log in and create tunnel

```bash
cloudflared tunnel login
cloudflared tunnel create edge-3090-1
```

Save the credentials path from the output and set `credentials-file` in the config to that path.

### 3. Config

- Use repo config: `cloudflared/config.yml` (or copy to `~/.cloudflared/config.yml`).
- Ensure `credentials-file` in the config points to your tunnel’s JSON (from step 2).

### 4. Route DNS

```bash
cloudflared tunnel route dns edge-3090-1 inference.taixingai.com
cloudflared tunnel route dns edge-3090-1 embedding.taixingai.com
```

### 5. API authentication (Cloudflare Access)

The APIs are protected by Cloudflare Access. Requests require a valid service token.

**Create a service token:**

1. In [Cloudflare One](https://one.dash.cloudflare.com/), go to **Access controls > Service credentials > Service Tokens**
2. Click **Generate token**, name it (e.g. `taixingai-api-token`), set duration, then **Create Service Token**
3. Copy and store the **Client ID** and **Client Secret** immediately (the secret is shown only once)

**Create Access applications** for `inference.taixingai.com` and `embedding.taixingai.com`:

1. Go to **Access controls > Applications** > **Add an application** > **Self-hosted**
2. Add each domain, enable **401 Response for Service Auth policies** (Advanced settings)
3. Add an **Allow** policy with action **Service Auth** and include your service token

## Run

**1. Start backends:** inference on port 8000, embedding on port 8001.

**2. Start the tunnel:**

```bash
cloudflared tunnel --config /home/hunter/Desktop/app/cloudfare-server/cloudflared/config.yml run edge-3090-1
```

(Or `cloudflared tunnel --config ~/.cloudflared/config.yml run edge-3090-1` if using home-dir config.)

**3. Test:**

Include the service token headers. Without them, requests receive `401 Unauthorized`.

```bash
curl -i -H "CF-Access-Client-Id: <CLIENT_ID>" \
     -H "CF-Access-Client-Secret: <CLIENT_SECRET>" \
     https://inference.taixingai.com/
curl -i -H "CF-Access-Client-Id: <CLIENT_ID>" \
     -H "CF-Access-Client-Secret: <CLIENT_SECRET>" \
     https://embedding.taixingai.com/
```

Store the token in env vars for convenience:

```bash
export CF_ACCESS_CLIENT_ID="<CLIENT_ID>"
export CF_ACCESS_CLIENT_SECRET="<CLIENT_SECRET>"
curl -i -H "CF-Access-Client-Id: $CF_ACCESS_CLIENT_ID" \
     -H "CF-Access-Client-Secret: $CF_ACCESS_CLIENT_SECRET" \
     https://inference.taixingai.com/
```

**Single-header auth (optional):** For clients that only support one custom header (e.g. OpenAI SDK), configure the Access app with `read_service_tokens_from_header: "Authorization"`. Then send:

```bash
curl -H "Authorization: {\"cf-access-client-id\": \"<CLIENT_ID>\", \"cf-access-client-secret\": \"<CLIENT_SECRET>\"}" \
  https://inference.taixingai.com/v1/chat/completions
```

**As a service (Linux):**

```bash
sudo cloudflared service install
```

Use the config path the service expects (often `/etc/cloudflared/config.yml`). Copy or symlink your config and credentials there.

## Result

- `https://inference.taixingai.com` → cloudflared → `http://localhost:8000`
- `https://embedding.taixingai.com` → cloudflared → `http://localhost:8001`

To add more hostnames, add ingress rules in `cloudflared/config.yml` and run `cloudflared tunnel route dns edge-3090-1 <hostname>` for each.
