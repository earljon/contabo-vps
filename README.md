# contabo-vps

A `cloud-init` configuration for bootstrapping a **Contabo VPS** running **Ubuntu 24.04 LTS** with Docker, security hardening, AWS CLI, Cloudflare Tunnel, CrowdSec, Coolify (self-hosted PaaS), and Slack notifications.

---

## Table of Contents

- [What It Does](#what-it-does)
- [Bootstrap Stages](#bootstrap-stages)
- [Required Secrets](#required-secrets)
- [How to Generate Each Secret](#how-to-generate-each-secret)
- [Rendering the Template](#rendering-the-template)
- [Usage with Contabo](#usage-with-contabo)
- [Cloudflare Configuration](#cloudflare-configuration)
- [Post-Bootstrap Verification](#post-bootstrap-verification)
- [Firewall & Ports](#firewall--ports)
- [Troubleshooting](#troubleshooting)

---

## What It Does

| Area | Details |
|---|---|
| **Users** | Creates `contabo` user (sudo + docker), locks password, installs SSH key |
| **SSH** | Moves SSH to port **2219**, key-only auth, no passwords, root = `prohibit-password` |
| **Firewall** | UFW: deny all inbound except port 2219/tcp |
| **Kernel** | sysctl network hardening (SYN flood, ICMP, IP spoofing, IP forwarding for Docker) |
| **Docker** | Installs Docker CE from official apt repo, enables on boot |
| **AWS CLI** | Installs AWS CLI v2, writes credentials for both `root` and `contabo` |
| **Coolify** | Installs Coolify PaaS, registers initial admin account, accessible at `coolify.acacialabs.com` |
| **Cloudflare Tunnel** | Installs `cloudflared`, registers tunnel via token, starts as a system service |
| **CrowdSec** | Installs agent + iptables bouncer for intrusion prevention |
| **Unattended Upgrades** | Auto-patches OS security updates, auto-reboots at 04:00 |
| **Notifications** | Slack webhook alerts at every bootstrap stage (INFO / SUCCESS / ERROR) |
| **MOTD** | Login banner showing SSH command, service status hints |
| **Logrotate** | Catch-all rotation for `/var/log/*.log` (daily, 14-day retention, compressed) |

---

## Bootstrap Stages

The `bootstrap.sh` script runs as a single bash process with per-stage error reporting. Each failed stage sends a Slack `ERROR` alert and halts.

```
1. contabo user      — create user, sudoers, docker group, SSH key
2. AWS CLI           — install v2 binary, write ~/.aws/credentials + config
3. UFW               — default deny in, allow 2219/tcp, enable firewall
4. SSH / sysctl      — apply sshd drop-ins, socket port override, kernel params
5. CrowdSec          — install agent + iptables bouncer, enable services
6. Docker / upgrades — enable docker.service + unattended-upgrades.service
7. Coolify           — install Coolify, wait for healthy, register initial admin via API
8. cloudflared       — install package, register tunnel token, start service
9. Lockout guard     — verify SSH keys exist; revert to port 22 if empty (failsafe)
→  Reboot            — final reboot to apply kernel/network changes
```

---

## Required Secrets

The template uses two kinds of placeholders:

| Placeholder | Type | Description |
|---|---|---|
| `<%= hostname %>` | ERB / template | Hostname to set on the server |
| `${SSH_PUBLIC_KEY}` | Shell variable | Ed25519 or RSA public key content |
| `${AWS_ACCESS_KEY_ID}` | Shell variable | AWS IAM access key ID |
| `${AWS_SECRET_ACCESS_KEY}` | Shell variable | AWS IAM secret access key |
| `${AWS_REGION}` | Shell variable | AWS region (e.g. `ap-southeast-1`) |
| `${CLOUDFLARED_TUNNEL_TOKEN}` | Shell variable | Cloudflare Zero Trust tunnel token |
| `${SLACK_WEBHOOK_URL}` | Shell variable | Slack incoming webhook URL |
| `${COOLIFY_INITIAL_EMAIL}` | Shell variable | Email address for the first Coolify admin account |
| `${COOLIFY_INITIAL_PASSWORD}` | Shell variable | Password for the first Coolify admin account |

---

## How to Generate Each Secret

### `SSH_PUBLIC_KEY`

Generate a new Ed25519 keypair (recommended) or use an existing one.

```bash
# Generate a new keypair
ssh-keygen -t ed25519 -C "contabo-vps" -f ~/.ssh/contabo_ed25519

# Print the public key — this is the value for SSH_PUBLIC_KEY
cat ~/.ssh/contabo_ed25519.pub
```

The value looks like:
```
ssh-ed25519 AAAAC3Nza... contabo-vps
```

Store the **private key** (`~/.ssh/contabo_ed25519`) securely. Never commit it.

---

### `AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY` + `AWS_REGION`

1. Sign in to the [AWS Console](https://console.aws.amazon.com/iam/).
2. Go to **IAM → Users → Create user** (or use an existing service account).
3. Attach the minimum required policies (e.g. `AmazonS3ReadOnlyAccess` or a custom policy).
4. Under the user's **Security credentials** tab, click **Create access key**.
5. Choose **Application running outside AWS**, then click **Create**.
6. Copy the **Access key ID** and **Secret access key** — the secret is shown **once only**.

```
AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
AWS_REGION=ap-southeast-1
```

> **Tip:** Prefer IAM roles or short-lived credentials for production. Long-lived keys written to disk should be scoped to the minimum needed permissions.

---

### `CLOUDFLARED_TUNNEL_TOKEN`

1. Log in to [Cloudflare Zero Trust](https://one.dash.cloudflare.com/).
2. Go to **Networks → Tunnels → Create a tunnel**.
3. Choose **Cloudflared** as the connector type.
4. Name the tunnel (e.g. `contabo-vps`).
5. On the **Install and run a connector** step, select **Docker** or **Debian**, and copy the token from the install command:

```bash
cloudflared service install <TOKEN_HERE>
```

6. The token is the long string after `service install`. Use it as `CLOUDFLARED_TUNNEL_TOKEN`.

> The token is a base64-encoded credential that registers this server as a connector for the tunnel. Treat it like a password — do not commit it.

---

### `SLACK_WEBHOOK_URL`

1. Go to [api.slack.com/apps](https://api.slack.com/apps) and click **Create New App → From scratch**.
2. Name the app (e.g. `VPS Alerts`) and select your workspace.
3. Under **Features**, click **Incoming Webhooks** and toggle it **On**.
4. Click **Add New Webhook to Workspace**, select a channel (e.g. `#vps-alerts`), and authorize.
5. Copy the webhook URL:

```
https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX
```

Use this as `SLACK_WEBHOOK_URL`. The bootstrap will post INFO/SUCCESS/ERROR messages to that channel.

---

### `COOLIFY_INITIAL_EMAIL` + `COOLIFY_INITIAL_PASSWORD`

These are the credentials for the **first admin account** on your Coolify dashboard. The bootstrap registers them automatically via the Coolify API immediately after installation (the registration endpoint is only available while no admin exists).

Choose any email/password combination:

```
COOLIFY_INITIAL_EMAIL=admin@acacialabs.com
COOLIFY_INITIAL_PASSWORD=ChangeMeAfterFirstLogin!
```

> **Security:** Use a strong password (16+ chars, mixed case, symbols). Change it after your first login at `https://coolify.acacialabs.com`. These values are embedded in the rendered cloud-init YAML — treat the rendered file as a secret.

---

## Rendering the Template

The `cloud-init.yaml` file uses two template syntaxes that must be resolved before uploading:

- `<%= hostname %>` — ERB-style (used by some provisioning tools)
- `${VARIABLE}` — shell-style (resolved via `envsubst`)

### Option A — envsubst (recommended for simple use)

```bash
# Set all required variables
export SSH_PUBLIC_KEY="$(cat ~/.ssh/contabo_ed25519.pub)"
export AWS_ACCESS_KEY_ID="AKIAIOSFODNN7EXAMPLE"
export AWS_SECRET_ACCESS_KEY="wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
export AWS_REGION="ap-southeast-1"
export CLOUDFLARED_TUNNEL_TOKEN="eyJ..."
export SLACK_WEBHOOK_URL="https://hooks.slack.com/services/..."
export COOLIFY_INITIAL_EMAIL="admin@acacialabs.com"
export COOLIFY_INITIAL_PASSWORD="ChangeMeAfterFirstLogin!"

# Replace hostname placeholder (ERB-style) manually, then run envsubst
HOSTNAME="my-vps-01"
sed "s/<%= hostname %>/${HOSTNAME}/g" cloud-init.yaml \
  | envsubst > cloud-init-rendered.yaml
```

> **Important:** `envsubst` will also expand `${distro_id}` and `${distro_codename}` in the unattended-upgrades block. To protect those, enumerate only the variables you want substituted:
> ```bash
> envsubst '${SSH_PUBLIC_KEY} ${AWS_ACCESS_KEY_ID} ${AWS_SECRET_ACCESS_KEY} ${AWS_REGION} ${CLOUDFLARED_TUNNEL_TOKEN} ${SLACK_WEBHOOK_URL} ${COOLIFY_INITIAL_EMAIL} ${COOLIFY_INITIAL_PASSWORD}' \
>   < cloud-init.yaml > cloud-init-rendered.yaml
> ```

### Option B — CI/CD secrets (GitHub Actions example)

Store all secrets in **GitHub → Settings → Secrets and variables → Actions**, then render in a workflow:

```yaml
- name: Render cloud-init
  env:
    SSH_PUBLIC_KEY: ${{ secrets.SSH_PUBLIC_KEY }}
    AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
    AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    AWS_REGION: ${{ secrets.AWS_REGION }}
    CLOUDFLARED_TUNNEL_TOKEN: ${{ secrets.CLOUDFLARED_TUNNEL_TOKEN }}
    SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
    COOLIFY_INITIAL_EMAIL: ${{ secrets.COOLIFY_INITIAL_EMAIL }}
    COOLIFY_INITIAL_PASSWORD: ${{ secrets.COOLIFY_INITIAL_PASSWORD }}
    HOSTNAME: my-vps-01
  run: |
    sed "s/<%= hostname %>/${HOSTNAME}/g" cloud-init.yaml \
      | envsubst '${SSH_PUBLIC_KEY} ${AWS_ACCESS_KEY_ID} ${AWS_SECRET_ACCESS_KEY} ${AWS_REGION} ${CLOUDFLARED_TUNNEL_TOKEN} ${SLACK_WEBHOOK_URL} ${COOLIFY_INITIAL_EMAIL} ${COOLIFY_INITIAL_PASSWORD}' \
      > cloud-init-rendered.yaml
```

### Verify the rendered output

Before uploading, confirm no unresolved placeholders remain:

```bash
grep -E '\$\{[A-Z_]+\}|<%=' cloud-init-rendered.yaml
# Should return no output
```

---

## Usage with Contabo

1. Log in to the [Contabo Customer Control Panel](https://my.contabo.com/).
2. Order or reinstall a VPS with **Ubuntu 24.04 LTS**.
3. Look for the **User Data / cloud-init** field during the reinstall wizard.
4. Paste the full contents of `cloud-init-rendered.yaml` into that field.
5. Confirm and start the installation.

Cloud-init runs on first boot. Bootstrap takes approximately **5–10 minutes** depending on package download speed (Coolify pulls several Docker images). You will receive Slack notifications as each stage completes.

---

## Cloudflare Configuration

These steps are performed **once after the VPS bootstrap completes**. They configure the Cloudflare side so that `https://coolify.acacialabs.com` resolves correctly and reaches your Coolify dashboard from any machine (including your Mac).

> **Prerequisites:** `acacialabs.com` must be added to your Cloudflare account as an active zone, and the Cloudflare Tunnel must have been created before bootstrap (the token was used in cloud-init).

---

### Step 1 — Add the Tunnel Public Hostname

This is what routes `coolify.acacialabs.com` through the encrypted tunnel to Coolify on the VPS.

1. Go to [Cloudflare Zero Trust](https://one.dash.cloudflare.com/) → **Networks → Tunnels**.
2. Click your tunnel (the one whose token you used in `CLOUDFLARED_TUNNEL_TOKEN`).
3. Click the **Public Hostnames** tab → **Add a public hostname**.
4. Fill in:

   | Field | Value |
   |---|---|
   | Subdomain | `coolify` |
   | Domain | `acacialabs.com` |
   | Type | `HTTP` |
   | URL | `localhost:8000` |

5. Leave all other fields at their defaults and click **Save hostname**.

Cloudflare automatically creates a DNS CNAME record for `coolify.acacialabs.com` pointing to your tunnel's `.cfargotunnel.com` address. You do not need to create it manually.

---

### Step 2 — Verify the DNS Record

Confirm the CNAME was created:

1. Go to [Cloudflare Dashboard](https://dash.cloudflare.com/) → **acacialabs.com → DNS → Records**.
2. You should see:

   | Type | Name | Content | Proxy |
   |---|---|---|---|
   | CNAME | `coolify` | `<uuid>.cfargotunnel.com` | Proxied (orange cloud) |

3. If the proxy toggle is **grey (DNS only)**, click the cloud icon to enable **Proxied** (orange cloud). This is required — the tunnel only works through Cloudflare's proxy.

---

### Step 3 — Set SSL/TLS Encryption Mode

This prevents `ERR_TOO_MANY_REDIRECTS` and ensures HTTPS works end-to-end.

1. Go to **Cloudflare Dashboard → acacialabs.com → SSL/TLS → Overview**.
2. Set the encryption mode to **Full**.

   | Mode | Result |
   |---|---|
   | Off | HTTP only — do not use |
   | Flexible | HTTPS to Cloudflare, HTTP to origin — can cause redirect loops with Coolify |
   | **Full** ✓ | HTTPS to Cloudflare, tunnel to origin — correct for Cloudflare Tunnel |
   | Full (Strict) | Requires a valid CA cert on the origin — not needed since the tunnel handles encryption |

> **Why Full, not Full (Strict)?** The Cloudflare Tunnel's mTLS connection between `cloudflared` and Cloudflare's edge already provides end-to-end encryption. The origin (`localhost:8000`) only needs to serve HTTP — Full mode covers this without requiring an origin certificate.

---

### Step 4 — Enable "Always Use HTTPS"

This redirects any `http://coolify.acacialabs.com` requests to HTTPS automatically.

1. Go to **SSL/TLS → Edge Certificates**.
2. Toggle **Always Use HTTPS** → **On**.

---

### Step 5 — Verify from Your Mac

Once the tunnel is connected and DNS has propagated (usually instant with Cloudflare):

```bash
# 1. Check DNS resolves to a Cloudflare IP (not your VPS IP)
dig coolify.acacialabs.com +short
# Expected: one or more Cloudflare IPs (e.g. 104.x.x.x)

# 2. Confirm HTTPS responds
curl -I https://coolify.acacialabs.com
# Expected: HTTP/2 200 (or 302 redirect to /login)

# 3. Check the tunnel status on the VPS
ssh -i ~/.ssh/contabo_ed25519 -p 2219 contabo@<your-server-ip> \
  "systemctl is-active cloudflared && curl -sf http://localhost:8000/api/health"
```

Then open your browser:
```
https://coolify.acacialabs.com
```

You should see the Coolify login page. Log in with `COOLIFY_INITIAL_EMAIL` and `COOLIFY_INITIAL_PASSWORD`.

---

### Cloudflare Configuration Checklist

- [ ] Tunnel public hostname added: `coolify.acacialabs.com` → `http://localhost:8000`
- [ ] DNS CNAME exists and proxy is **orange (Proxied)**
- [ ] SSL/TLS mode set to **Full**
- [ ] Always Use HTTPS **On**
- [ ] `dig coolify.acacialabs.com` returns a Cloudflare IP
- [ ] `https://coolify.acacialabs.com` loads in the browser

---

## Post-Bootstrap Verification

### Step 1 — SSH in and verify components

SSH in once the SUCCESS Slack message arrives:

```bash
ssh -i ~/.ssh/contabo_ed25519 -p 2219 contabo@<your-server-ip>
```

Then verify each component:

```bash
# SSH is on port 2219
ss -tlnp | grep 2219

# Firewall
sudo ufw status verbose

# Docker
docker info

# AWS CLI
aws sts get-caller-identity

# Coolify service
systemctl status coolify
curl -sf http://localhost:8000/api/health

# Traefik (started by Coolify automatically)
docker ps | grep traefik

# Cloudflared tunnel
systemctl status cloudflared

# CrowdSec
sudo cscli metrics

# Unattended upgrades
systemctl status unattended-upgrades

# Cloud-init summary
cloud-init status --long
```

---

## Firewall & Ports

| Port | Protocol | Direction | Purpose |
|---|---|---|---|
| 2219 | TCP | Inbound | SSH (custom port) |
| All others | — | Inbound | **Denied** by UFW default |
| All | — | Outbound | Allowed by UFW default |

> Cloudflare Tunnel creates an **outbound-only** connection to Cloudflare's edge — no additional inbound ports are needed for tunneled services.

---

## Troubleshooting

**Bootstrap log:**
```bash
sudo tail -f /var/log/cloud-init-output.log
```

**Cloud-init status:**
```bash
cloud-init status --long
cloud-init analyze show
```

**SSH locked out (no key / wrong key):**

The lockout guard (`stage_lockout_guard`) detects empty `authorized_keys` on both users and automatically reverts SSH to port 22 with the default config, preventing total lockout. If that happens, use the Contabo rescue console to fix the SSH key secret and re-run.

**`envsubst` expands `${distro_id}`:**

Always pass the explicit variable list to `envsubst` (see Option A above) so distro-specific APT variables in the unattended-upgrades config are not accidentally blanked out.

**Coolify did not become healthy (timeout after 120 s):**

The `stage_coolify` health check polls `http://localhost:8000/api/health` every 5 seconds for up to 2 minutes. If it times out, check:
```bash
sudo journalctl -u coolify -n 50
docker ps -a   # look for exited containers
cat /var/log/cloud-init-output.log | grep -A5 "Stage: Coolify"
```

**Coolify admin registration failed:**

The `/api/v1/register` call is fire-and-forget (`|| true`) so bootstrap does not halt if it fails. If you cannot log in:
```bash
# Re-attempt registration manually (only works while no admin exists)
curl -X POST http://localhost:8000/api/v1/register \
  -H 'Content-Type: application/json' \
  -d '{"name":"Admin","email":"you@example.com","password":"yourpassword","password_confirmation":"yourpassword"}'
```

**`https://coolify.acacialabs.com` shows ERR_TOO_MANY_REDIRECTS:**

SSL/TLS mode is set to **Flexible** instead of **Full**. Coolify redirects HTTP → HTTPS internally, Cloudflare redirects HTTPS → HTTP under Flexible mode, creating a loop. Fix: Cloudflare Dashboard → acacialabs.com → SSL/TLS → set to **Full**.

**`https://coolify.acacialabs.com` shows a 522 (Connection Timed Out) or 1033 error:**

The tunnel is not connected. On the VPS:
```bash
systemctl status cloudflared
journalctl -u cloudflared -n 30
```
If the service is down, restart it: `sudo systemctl restart cloudflared`. If it shows `ERR_FAILED_TO_DIAL_TUNNEL`, the tunnel token may be invalid — re-check `CLOUDFLARED_TUNNEL_TOKEN`.

**`https://coolify.acacialabs.com` shows a 502 Bad Gateway:**

The tunnel is connected but Coolify is not running on port 8000. On the VPS:
```bash
systemctl status coolify
curl -sf http://localhost:8000/api/health
docker ps   # all Coolify containers should be Up
```
Restart if needed: `sudo systemctl restart coolify`.

**DNS does not resolve / `dig` returns NXDOMAIN:**

The public hostname was not saved in Cloudflare Zero Trust, or DNS has not propagated. Check:
```bash
dig coolify.acacialabs.com +short
# Expected: Cloudflare IP like 104.x.x.x
```
If empty, re-add the public hostname in Zero Trust → Tunnels → your tunnel → Public Hostnames. Cloudflare DNS propagation is usually under 30 seconds.

**Browser shows "Your connection is not private" (SSL warning):**

The proxy status on the DNS record is **grey (DNS only)** instead of **orange (Proxied)**. Go to Cloudflare Dashboard → acacialabs.com → DNS, find the `coolify` CNAME record, and click the cloud icon to make it orange.

**Slack not notifying:**

The notify script exits silently (`|| true`) if the webhook is unreachable or empty — bootstrap will still complete. Check the cloud-init log for stage output, and verify the webhook URL is correct and the Slack app is still active.
