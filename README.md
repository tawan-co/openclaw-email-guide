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

Outbound email → Cloudflare Worker (allowlist) → Resend API → recipient
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

### 6. Outbound Email (Secure Gateway)

To let your agent *send* emails safely, use a **Cloudflare Worker as a gateway** with an allowlist. This ensures your agent can only email addresses you've explicitly authorized — enforced at the infrastructure level.

> ⚠️ **Why a gateway?** If you give your agent direct access to an email API key, it could theoretically email anyone. A jailbreak, prompt injection, or bug could result in spam or worse. The gateway pattern enforces an allowlist server-side — the agent physically cannot bypass it.

#### 6a. Create a Resend Account

1. Sign up at [Resend](https://resend.com)
2. Add your domain and verify DNS records
3. Get an API key — **keep this secret, never share with your agent**

#### 6b. Create the Send Gateway Worker

Create a new Cloudflare Worker for outbound email:

```javascript
export default {
  async fetch(request, env) {
    // Only POST
    if (request.method !== 'POST') {
      return new Response('Method not allowed', { status: 405 });
    }

    // Check auth token
    const authHeader = request.headers.get('Authorization');
    if (authHeader !== `Bearer ${env.AUTH_TOKEN}`) {
      return new Response('Unauthorized', { status: 401 });
    }

    // Parse request
    let body;
    try {
      body = await request.json();
    } catch {
      return new Response('Invalid JSON', { status: 400 });
    }

    const { to, subject, text, replyTo } = body;
    if (!to || !subject || !text) {
      return new Response('Missing required fields: to, subject, text', { status: 400 });
    }

    // ============================================
    // ALLOWLIST — edit this list to control who
    // your agent can email. Add addresses here.
    // ============================================
    const ALLOWLIST = [
      'you@example.com',
      'trusted-contact@example.com',
      // Add more as needed
    ];

    if (!ALLOWLIST.includes(to.toLowerCase())) {
      return Response.json(
        { error: `Recipient ${to} not in allowlist` }, 
        { status: 403 }
      );
    }

    // Send via Resend
    const resendRes = await fetch('https://api.resend.com/emails', {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${env.RESEND_API_KEY}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        from: `Agent <agent@yourdomain.com>`,
        to: [to],
        reply_to: replyTo || 'agent@yourdomain.com',
        subject,
        text,
      }),
    });

    const result = await resendRes.json();
    
    if (result.id) {
      return Response.json({ success: true, id: result.id });
    } else {
      return Response.json({ error: result }, { status: 500 });
    }
  },
};
```

#### 6c. Configure Worker Environment Variables

In the Cloudflare dashboard, add these secrets to your Worker:

| Variable | Value |
|----------|-------|
| `RESEND_API_KEY` | Your Resend API key |
| `AUTH_TOKEN` | Generate a random token (e.g. `openssl rand -hex 32`) |

#### 6d. Create a Local Send Script

On your machine, create a script that calls the gateway:

```javascript
#!/usr/bin/env node
// scripts/send-email.js

const https = require('https');

const args = process.argv.slice(2);
const getArg = (name) => {
  const idx = args.indexOf(`--${name}`);
  return idx > -1 ? args[idx + 1] : null;
};

const to = getArg('to');
const subject = getArg('subject');
const body = getArg('body');

if (!to || !subject || !body) {
  console.error('Usage: node send-email.js --to <email> --subject <subject> --body <body>');
  process.exit(1);
}

// Your gateway URL and auth token
const GATEWAY_URL = 'https://send-email.yourdomain.workers.dev';
const AUTH_TOKEN = 'your-auth-token-here';

const payload = JSON.stringify({ to, subject, text: body });

const url = new URL(GATEWAY_URL);
const options = {
  hostname: url.hostname,
  path: url.pathname,
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${AUTH_TOKEN}`,
    'Content-Length': Buffer.byteLength(payload)
  }
};

const req = https.request(options, (res) => {
  let data = '';
  res.on('data', chunk => data += chunk);
  res.on('end', () => {
    const result = JSON.parse(data);
    if (result.success) {
      console.log(`✅ Email sent to ${to}`);
    } else {
      console.error(`❌ Blocked: ${result.error}`);
      process.exit(1);
    }
  });
});

req.on('error', (e) => console.error('Request failed:', e.message));
req.write(payload);
req.end();
```

#### 6e. Teach Your Agent

Add to your agent's instructions (e.g., `TOOLS.md`):

```markdown
### Email Sending (ENFORCED)
- **ALWAYS use:** `scripts/send-email.js`
- **NEVER call any email API directly**
- Allowlist is server-side — you cannot modify it
- Usage: `node scripts/send-email.js --to <email> --subject <subject> --body <body>`
```

Now your agent can only email addresses in your allowlist, and you control that list entirely.

### 7. Test It

Send an email to `agent@yourdomain.com` and watch it arrive in Slack via your agent. ✨

---

## Plus Addressing (Route Emails to Different Channels)

Want different emails to go to different Slack channels? Use **plus addressing** with a catch-all rule.

### How It Works

```
agent+project-a@yourdomain.com → #project-a channel
agent+support@yourdomain.com   → #support channel
agent@yourdomain.com           → default channel (DM or general)
```

### Setup

#### 1. Enable Catch-All in Cloudflare

In **Email Routing**, set up a **Catch-all address** that routes to your worker. This captures `agent+anything@yourdomain.com` addresses that don't have explicit rules.

#### 2. Update the Worker to Parse Plus Tags

```javascript
export default {
  async email(message, env) {
    const to = message.to;
    const from = message.from;
    const subject = message.headers.get("subject") || "(no subject)";

    // Parse plus tag: agent+project-a@domain.com → "project-a"
    const match = to.match(/^([^+]+)\+([^@]+)@/);
    const tag = match ? match[2] : null;
    const baseAddress = match ? `${match[1]}@${to.split("@")[1]}` : to;

    // Read body
    const rawEmail = await new Response(message.raw).text();
    const bodyStart = rawEmail.indexOf("\r\n\r\n");
    const body = bodyStart > -1 ? rawEmail.slice(bodyStart + 4, bodyStart + 4000) : "";

    // POST to OpenClaw webhook
    await fetch(env.WEBHOOK_URL, {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        "Authorization": `Bearer ${env.WEBHOOK_TOKEN}`,
      },
      body: JSON.stringify({
        to,
        from,
        subject,
        tag,           // null if no plus tag
        baseAddress,   // agent@domain.com (without the +tag)
        body: body.trim().slice(0, 4000),
      }),
    });
  },
};
```

#### 3. Update OpenClaw Webhook Config

Include `{{tag}}` in your message template:

```json
{
  "hooks": {
    "enabled": true,
    "path": "/hooks",
    "token": "your-secret-token-here",
    "mappings": [
      {
        "match": { "path": "email" },
        "action": "agent",
        "wakeMode": "now",
        "name": "Email (agent@yourdomain.com)",
        "messageTemplate": "📧 Email received\nTo: {{to}}\nTag: {{tag}}\nFrom: {{from}}\nSubject: {{subject}}\n\n{{body}}",
        "deliver": true,
        "channel": "slack",
        "to": "channel:YOUR_DEFAULT_CHANNEL_ID"
      }
    ]
  }
}
```

#### 4. Teach Your Agent the Routing Rules

Add to your agent's memory (e.g., `MEMORY.md` or `SOUL.md`):

```markdown
## Email Routing Rules

**Inbound:** When I receive an email with a plus tag (e.g., agent+support@domain.com), 
post a summary to the Slack channel matching that tag name. No tag = DM to owner.

**Outbound:** When sending emails from a specific channel context, use the plus tag 
for that channel (e.g., from #support → send as agent+support@domain.com). 
This ensures replies route back to the right channel.

**Tag → Channel Mapping:**
- `support` → C0ABC123 (#support)
- `sales` → C0DEF456 (#sales)
- `project-a` → C0GHI789 (#project-a)
```

### Benefits

- **Automatic routing** — Emails tagged with `+channel-name` go to that channel
- **Reply threading** — When your agent sends FROM a channel using the plus tag, replies come back to the same channel
- **One email address** — No need to create multiple email addresses or workers
- **Zero config for new channels** — Just use `agent+new-channel@domain.com` and add to the mapping

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
