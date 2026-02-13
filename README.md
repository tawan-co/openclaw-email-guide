# Give Your OpenClaw Agent Its Own Email Address

Set up inbound and outbound email for your [OpenClaw](https://github.com/openclaw/openclaw) agent in ~15 minutes. All free tiers.

> **Note:** This guide is for a **local machine** (laptop/desktop/Mac Mini). If you're running OpenClaw on a **VPS** with a public IP, you can skip the Tailscale Funnel step entirely — just point your Cloudflare Worker directly at `https://your-vps-ip:18789/hooks/email` (with proper TLS via reverse proxy like Caddy or nginx).

**Stack:** Cloudflare Email Routing → Cloudflare Worker → Tailscale Funnel → OpenClaw Webhook → Slack

---

## How It Works

```
Inbound email → Cloudflare Email Routing → Cloudflare Worker
    → POST JSON to OpenClaw webhook (via Tailscale Funnel)
    → Agent reads, summarizes, delivers to Slack

Outbound email → Resend API → recipient
```

Your agent gets a real email address (e.g. `agent@yourdomain.com`). When someone emails it, your agent wakes up, reads the message, and notifies you. It can also send emails on your behalf via Resend.

---

## Steps

### 1. Domain Setup (Cloudflare)

- Add your domain to [Cloudflare](https://dash.cloudflare.com/) (if not already)
- Go to **Email Routing** → enable it
- Add a catch-all or specific address (e.g. `agent@yourdomain.com`)
- Route it to a **Cloudflare Worker** (you'll create this next)

### 2. Cloudflare Worker

Create a Worker that receives inbound email and forwards it to your OpenClaw webhook as JSON.

```javascript
export default {
  async email(message, env) {
    const rawEmail = await new Response(message.raw).text();

    // Extract basic fields
    const from = message.from;
    const to = message.to;
    const subject = message.headers.get("subject") || "(no subject)";

    // Parse body (simplified — enhance as needed)
    const body = rawEmail.split("\r\n\r\n").slice(1).join("\r\n\r\n").trim();

    // POST to OpenClaw webhook
    const webhookUrl = `${env.WEBHOOK_URL}?token=${env.WEBHOOK_TOKEN}`;

    await fetch(webhookUrl, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ from, to, subject, body }),
    });
  },
};
```

**Worker environment variables:**
- `WEBHOOK_URL` — Your OpenClaw webhook endpoint (e.g. `https://your-machine.tailnet.ts.net/hooks/email`)
- `WEBHOOK_TOKEN` — The secret token from your OpenClaw hooks config

> **Note:** This is a minimal example. You may want to add proper MIME parsing, attachment handling, HTML-to-text conversion, etc.

### 3. Expose OpenClaw to the Internet (Tailscale Funnel)

> **VPS users:** Skip this step. Your machine is already public. Use a reverse proxy (Caddy, nginx) to terminate TLS and forward to `localhost:18789`.

Your OpenClaw gateway needs to be reachable from Cloudflare. For local machines, [Tailscale Funnel](https://tailscale.com/kb/1223/tailscale-funnel/) is the easiest way.

```bash
# Install Tailscale if you haven't
# https://tailscale.com/download

# Expose your OpenClaw gateway (default port 18789) via HTTPS
tailscale serve --https=443 http://localhost:18789

# Make it publicly accessible
tailscale funnel 443 on

# Verify
tailscale funnel status
```

Your public URL will be something like:
```
https://your-machine.tailnet-name.ts.net
```

### 4. Configure OpenClaw Webhook

Add the `hooks` section to your `openclaw.json`:

```json
{
  "hooks": {
    "enabled": true,
    "path": "/hooks",
    "token": "your-secret-token-here",
    "mappings": [
      {
        "match": {
          "path": "email"
        },
        "action": "agent",
        "wakeMode": "now",
        "name": "Email (agent@yourdomain.com)",
        "messageTemplate": "📧 Email received at agent@yourdomain.com\nFrom: {{from}}\nSubject: {{subject}}\n\n{{body}}",
        "deliver": true,
        "channel": "slack",
        "to": "channel:YOUR_SLACK_CHANNEL_ID"
      }
    ]
  }
}
```

Then restart your gateway:
```bash
openclaw gateway restart
```

### 5. Point the Worker at OpenClaw

In the Cloudflare dashboard, set your Worker's environment variables:

| Variable | Value |
|----------|-------|
| `WEBHOOK_URL` | `https://your-machine.tailnet-name.ts.net/hooks/email` |
| `WEBHOOK_TOKEN` | The token from step 4 |

### 6. Outbound Email (Resend)

To let your agent *send* emails:

1. Create a [Resend](https://resend.com) account
2. Add your domain and verify DNS records
3. Get an API key
4. Store it securely (e.g. 1Password, env var)

Your agent can now send emails via the Resend API.

### 7. Test It

Send an email to `agent@yourdomain.com` and watch it arrive in Slack via your agent. ✨

---

## Cost

| Service | Tier | Limit |
|---------|------|-------|
| Cloudflare Email Routing | Free | Unlimited |
| Cloudflare Workers | Free | 100k requests/day |
| Tailscale Funnel | Free | Included with personal plan |
| Resend | Free | 100 emails/day |
| **Total** | **$0/mo** | |

---

## Architecture

```
                    ┌─────────────────┐
                    │  Someone sends  │
                    │  an email to    │
                    │  agent@domain   │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │   Cloudflare    │
                    │  Email Routing  │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │   Cloudflare    │
                    │    Worker       │
                    │  (parse email,  │
                    │   POST JSON)    │
                    └────────┬────────┘
                             │ HTTPS
                    ┌────────▼────────┐
                    │   Tailscale     │
                    │    Funnel       │
                    └────────┬────────┘
                             │ localhost
                    ┌────────▼────────┐
                    │    OpenClaw     │
                    │    Webhook      │
                    │  /hooks/email   │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │   AI Agent      │
                    │  reads, thinks, │
                    │  summarizes     │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │     Slack       │
                    │  (or any chat)  │
                    └─────────────────┘
```

---

Made with 🐱 by [Tawan](https://tawan.ca)
