# API via Cloudflare Tunnel (edge-3090-1)

Expose APIs at `https://inference.taixingai.com` and `https://embedding.taixingai.com` using one Cloudflare Tunnel — no Caddy, no open ports.

| Hostname | Backend |
|----------|---------|
| `inference.taixingai.com` | `http://localhost:8000` |
| `embedding.taixingai.com` | `http://localhost:8001` |

## Prerequisites

- Cloudflare account, domain `taixingai.com` on Cloudflare DNS
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

## Run

**1. Start backends:** inference on port 8000, embedding on port 8001.

**2. Start the tunnel:**

```bash
cloudflared tunnel --config /home/hunter/Desktop/app/cloudfare-server/cloudflared/config.yml run edge-3090-1
```

(Or `cloudflared tunnel --config ~/.cloudflared/config.yml run edge-3090-1` if using home-dir config.)

**3. Test:**

```bash
curl -i https://inference.taixingai.com/
curl -i https://embedding.taixingai.com/
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
