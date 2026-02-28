---
title: Fly.io
description: Deploy OpenClaw on Fly.io
summary: "Step-by-step Fly.io deployment for OpenClaw with persistent storage and HTTPS"
read_when:
  - Deploying OpenClaw on Fly.io
  - Setting up Fly volumes, secrets, and first-run config
---

# Fly.io Deployment

**Goal:** OpenClaw Gateway running on a [Fly.io](https://fly.io) machine with persistent storage, automatic HTTPS, and Discord/channel access.

## What you need

- [flyctl CLI](https://fly.io/docs/hands-on/install-flyctl/) installed
- Fly.io account (free tier works)
- Model auth: Anthropic API key (or other provider keys)
- Channel credentials: Discord bot token, Telegram token, etc.

## Beginner quick path

1. Clone repo → customize `fly.toml`
2. Create app + volume → set secrets
3. Deploy with `fly deploy`
4. SSH in to create config or use Control UI

## 1) Create the Fly app

```bash
# Clone the repo
git clone https://github.com/openclaw/openclaw.git
cd openclaw

# Create a new Fly app (pick your own name)
fly apps create my-openclaw

# Create a persistent volume (1GB is usually enough)
fly volumes create openclaw_data --size 1 --region iad
```

**Tip:** Choose a region close to you. Common options: `lhr` (London), `iad` (Virginia), `sjc` (San Jose).

## 2) Configure fly.toml

Edit `fly.toml` to match your app name and requirements.

**Security note:** The default config exposes a public URL. For a hardened deployment with no public IP, see [Private Deployment](#private-deployment-hardened) or use `fly.private.toml`.

```toml
app = "my-openclaw"  # Your app name
primary_region = "iad"

[build]
  dockerfile = "Dockerfile"

[env]
  NODE_ENV = "production"
  OPENCLAW_PREFER_PNPM = "1"
  OPENCLAW_STATE_DIR = "/data"
  NODE_OPTIONS = "--max-old-space-size=1536"

[processes]
  app = "node dist/index.js gateway --allow-unconfigured --port 3000 --bind lan"

[http_service]
  internal_port = 3000
  force_https = true
  auto_stop_machines = false
  auto_start_machines = true
  min_machines_running = 1
  processes = ["app"]

[[vm]]
  size = "shared-cpu-4x"
  memory = "4096mb"

[mounts]
  source = "openclaw_data"
  destination = "/data"
```

**Key settings:**

| Setting                        | Why                                                                         |
| ------------------------------ | --------------------------------------------------------------------------- |
| `--bind lan`                   | Binds to `0.0.0.0` so Fly's proxy can reach the gateway                     |
| `--allow-unconfigured`         | Starts without a config file (you'll create one after)                      |
| `internal_port = 3000`         | Must match `--port 3000` (or `OPENCLAW_GATEWAY_PORT`) for Fly health checks |
| `memory = "4096mb"`            | 512MB is too small; 4GB recommended for reliable cold starts                |
| `OPENCLAW_STATE_DIR = "/data"` | Persists state on the volume                                                |

## 3) Set secrets

```bash
# Required: Gateway token (for non-loopback binding)
fly secrets set OPENCLAW_GATEWAY_TOKEN=$(openssl rand -hex 32)

# Model provider API keys
fly secrets set ANTHROPIC_API_KEY=sk-ant-...

# Optional: Other providers
fly secrets set OPENAI_API_KEY=sk-...
fly secrets set GOOGLE_API_KEY=...

# Channel tokens
fly secrets set DISCORD_BOT_TOKEN=MTQ...
```

**Notes:**

- Non-loopback binds (`--bind lan`) require `OPENCLAW_GATEWAY_TOKEN` for security.
- Treat these tokens like passwords.
- **Prefer env vars over config file** for all API keys and tokens. This keeps secrets out of `openclaw.json` where they could be accidentally exposed or logged.

## 4) Deploy

```bash
fly deploy
```

First deploy builds the Docker image (~2-3 minutes). Subsequent deploys are faster.

After deployment, verify:

```bash
fly status
fly logs
```

You should see:

```
[gateway] listening on ws://0.0.0.0:3000 (PID xxx)
```

## 5) Create config file

The gateway requires a config file with `gateway.controlUi.allowedOrigins` when binding to non-loopback (`--bind lan`). Without it, the gateway will refuse to start.

SSH into the machine and create a minimal config:

```bash
echo '{"gateway":{"mode":"local","controlUi":{"allowedOrigins":["https://my-openclaw.fly.dev"]}}}' \
  | fly ssh console -C "tee /data/openclaw.json"
```

Replace `my-openclaw` with your actual Fly app name.

Then restart:

```bash
fly machines list                    # find your machine ID
fly machine restart <machine-id>
```

Verify the gateway starts:

```bash
fly logs
```

You should see `[gateway] listening on ws://0.0.0.0:3000`.

### Full config example

For a more complete setup with agents and channels, SSH in and write a full config:

```bash
fly ssh console
```

```bash
cat > /data/openclaw.json << 'EOF'
{
  "agents": {
    "defaults": {
      "model": {
        "primary": "anthropic/claude-opus-4-6",
        "fallbacks": ["anthropic/claude-sonnet-4-5", "openai/gpt-4o"]
      },
      "maxConcurrent": 4
    },
    "list": [
      {
        "id": "main",
        "default": true
      }
    ]
  },
  "auth": {
    "profiles": {
      "anthropic:default": { "mode": "token", "provider": "anthropic" },
      "openai:default": { "mode": "token", "provider": "openai" }
    }
  },
  "bindings": [
    {
      "agentId": "main",
      "match": { "channel": "discord" }
    }
  ],
  "channels": {
    "discord": {
      "enabled": true,
      "groupPolicy": "allowlist",
      "guilds": {
        "YOUR_GUILD_ID": {
          "channels": { "general": { "allow": true } },
          "requireMention": false
        }
      }
    }
  },
  "gateway": {
    "mode": "local",
    "controlUi": {
      "allowedOrigins": ["https://my-openclaw.fly.dev"]
    }
  },
  "meta": {
    "lastTouchedVersion": "2026.1.29"
  }
}
EOF
```

**Important:** Replace `my-openclaw` in `allowedOrigins` with your actual Fly app name.

**Note:** With `OPENCLAW_STATE_DIR=/data`, the config path is `/data/openclaw.json`.

**Note:** The Discord token can come from either:

- Environment variable: `DISCORD_BOT_TOKEN` (recommended for secrets)
- Config file: `channels.discord.token`

If using env var, no need to add token to config. The gateway reads `DISCORD_BOT_TOKEN` automatically.

Restart to apply:

```bash
exit
fly machine restart <machine-id>
```

## 6) Access the Gateway

### Control UI

Open in browser:

```bash
fly open
```

Or visit `https://my-openclaw.fly.dev/`

**Step 1: Enter the gateway token**

Click **Overview** in the left sidebar. Paste your `OPENCLAW_GATEWAY_TOKEN` value into the **Gateway Token** field and click **Connect**.

**Step 2: Approve device pairing**

On first connection from a new browser, the Control UI will show a "pairing required" message. Approve the device via SSH:

```bash
# List pending pairing requests
fly ssh console -C "sh -c 'OPENCLAW_GATEWAY_PORT=3000 node /app/openclaw.mjs devices list'"

# Approve the pending request (copy the Request ID from the output)
fly ssh console -C "sh -c 'OPENCLAW_GATEWAY_PORT=3000 node /app/openclaw.mjs devices approve <request-id>'"
```

The browser will reconnect automatically after approval. You only need to do this once per browser/device.

### Logs

```bash
fly logs              # Live logs
fly logs --no-tail    # Recent logs
```

### SSH Console

```bash
fly ssh console
```

## Troubleshooting

### "App is not listening on expected address"

The gateway is binding to `127.0.0.1` instead of `0.0.0.0`.

**Fix:** Add `--bind lan` to your process command in `fly.toml`.

### "non-loopback Control UI requires gateway.controlUi.allowedOrigins"

The gateway crashes on startup because `--bind lan` requires a config file with `gateway.controlUi.allowedOrigins`.

**Fix:** Create a config file with your app's origin before starting the gateway. See [Step 5](#5-create-config-file).

### Health checks failing / connection refused

Fly can't reach the gateway on the configured port.

**Fix:** Ensure `internal_port` matches the gateway port (set `--port 3000` or `OPENCLAW_GATEWAY_PORT=3000`).

### OOM / Memory Issues

Container keeps restarting or getting killed. Signs: `SIGABRT`, `v8::internal::Runtime_AllocateInYoungGeneration`, or silent restarts.

**Fix:** Increase memory in `fly.toml`:

```toml
[[vm]]
  size = "shared-cpu-4x"
  memory = "4096mb"
```

Or update an existing machine:

```bash
fly machine update <machine-id> --vm-memory 4096 --vm-size shared-cpu-4x -y
```

**Note:** 512MB is too small. 2GB may work for warm restarts but cold starts can hang on `shared-cpu-2x` due to module initialization. **`shared-cpu-4x` with 4GB is recommended.**

### Gateway process running but never listens (cold start hang)

The gateway process is alive (`ps aux` shows `openclaw-gateway`) but never binds to port 3000. Fly proxy reports "instance refused connection".

This happens on `shared-cpu-2x` (2 vCPUs) because module initialization is too slow. The gateway can take 5+ minutes to start on a small VM, exceeding Fly's patience.

**Fix:** Upgrade to `shared-cpu-4x`:

```bash
fly machine update <machine-id> --vm-memory 4096 --vm-size shared-cpu-4x -y
```

After upgrade, cold starts complete in ~50 seconds.

### Gateway Lock Issues

Gateway refuses to start with "already running" errors.

This happens when the container restarts but the PID lock file persists on the volume.

**Fix:** Delete the lock file:

```bash
fly ssh console --command "rm -f /data/gateway.*.lock"
fly machine restart <machine-id>
```

The lock file is at `/data/gateway.*.lock` (not in a subdirectory).

### Config Not Being Read

If using `--allow-unconfigured`, the gateway creates a minimal config. Your custom config at `/data/openclaw.json` should be read on restart.

Verify the config exists:

```bash
fly ssh console --command "cat /data/openclaw.json"
```

### Writing Config via SSH

The `fly ssh console -C` command doesn't support shell redirection. To write a config file:

```bash
# Use echo + tee (pipe from local to remote)
echo '{"your":"config"}' | fly ssh console -C "tee /data/openclaw.json"

# Or use sftp
fly sftp shell
> put /local/path/config.json /data/openclaw.json
```

**Note:** `fly sftp` may fail if the file already exists. Delete first:

```bash
fly ssh console --command "rm /data/openclaw.json"
```

### State Not Persisting

If you lose credentials or sessions after a restart, the state dir is writing to the container filesystem.

**Fix:** Ensure `OPENCLAW_STATE_DIR=/data` is set in `fly.toml` and redeploy.

## Updates

```bash
# Pull latest changes
git pull

# Redeploy
fly deploy

# Check health
fly status
fly logs
```

### Updating Machine Command

If you need to change the startup command without a full redeploy:

```bash
# Get machine ID
fly machines list

# Update command
fly machine update <machine-id> --command "node dist/index.js gateway --port 3000 --bind lan" -y

# Or with memory increase
fly machine update <machine-id> --vm-memory 4096 --vm-size shared-cpu-4x --command "node dist/index.js gateway --port 3000 --bind lan" -y
```

**Note:** After `fly deploy`, the machine command may reset to what's in `fly.toml`. If you made manual changes, re-apply them after deploy.

## Private Deployment (Hardened)

By default, Fly allocates public IPs, making your gateway accessible at `https://your-app.fly.dev`. This is convenient but means your deployment is discoverable by internet scanners (Shodan, Censys, etc.).

For a hardened deployment with **no public exposure**, use the private template.

### When to use private deployment

- You only make **outbound** calls/messages (no inbound webhooks)
- You use **ngrok or Tailscale** tunnels for any webhook callbacks
- You access the gateway via **SSH, proxy, or WireGuard** instead of browser
- You want the deployment **hidden from internet scanners**

### Setup

Use `fly.private.toml` instead of the standard config:

```bash
# Deploy with private config
fly deploy -c fly.private.toml
```

Or convert an existing deployment:

```bash
# List current IPs
fly ips list -a my-openclaw

# Release public IPs
fly ips release <public-ipv4> -a my-openclaw
fly ips release <public-ipv6> -a my-openclaw

# Switch to private config so future deploys don't re-allocate public IPs
# (remove [http_service] or deploy with the private template)
fly deploy -c fly.private.toml

# Allocate private-only IPv6
fly ips allocate-v6 --private -a my-openclaw
```

After this, `fly ips list` should show only a `private` type IP:

```
VERSION  IP                   TYPE             REGION
v6       fdaa:x:x:x:x::x      private          global
```

### Accessing a private deployment

Since there's no public URL, use one of these methods:

**Option 1: Local proxy (simplest)**

```bash
# Forward local port 3000 to the app
fly proxy 3000:3000 -a my-openclaw

# Then open http://localhost:3000 in browser
```

**Option 2: WireGuard VPN**

```bash
# Create WireGuard config (one-time)
fly wireguard create

# Import to WireGuard client, then access via internal IPv6
# Example: http://[fdaa:x:x:x:x::x]:3000
```

**Option 3: SSH only**

```bash
fly ssh console -a my-openclaw
```

### Webhooks with private deployment

If you need webhook callbacks (Twilio, Telnyx, etc.) without public exposure:

1. **ngrok tunnel** - Run ngrok inside the container or as a sidecar
2. **Tailscale Funnel** - Expose specific paths via Tailscale
3. **Outbound-only** - Some providers (Twilio) work fine for outbound calls without webhooks

Example voice-call config with ngrok:

```json
{
  "plugins": {
    "entries": {
      "voice-call": {
        "enabled": true,
        "config": {
          "provider": "twilio",
          "tunnel": { "provider": "ngrok" },
          "webhookSecurity": {
            "allowedHosts": ["example.ngrok.app"]
          }
        }
      }
    }
  }
}
```

The ngrok tunnel runs inside the container and provides a public webhook URL without exposing the Fly app itself. Set `webhookSecurity.allowedHosts` to the public tunnel hostname so forwarded host headers are accepted.

### Security benefits

| Aspect            | Public       | Private    |
| ----------------- | ------------ | ---------- |
| Internet scanners | Discoverable | Hidden     |
| Direct attacks    | Possible     | Blocked    |
| Control UI access | Browser      | Proxy/VPN  |
| Webhook delivery  | Direct       | Via tunnel |

## Notes

- Fly.io uses **x86 architecture** (not ARM)
- The Dockerfile is compatible with both architectures
- For WhatsApp/Telegram onboarding, use `fly ssh console`
- Persistent data lives on the volume at `/data`
- Signal requires Java + signal-cli; use a custom image and keep memory at 2GB+.

## Cost

With the recommended config (`shared-cpu-4x`, 4GB RAM):

- ~$25-30/month depending on usage
- Free tier includes some allowance
- `shared-cpu-2x` (2GB) is cheaper (~$10-15/month) but cold starts may hang due to slow module initialization

See [Fly.io pricing](https://fly.io/docs/about/pricing/) for details.
