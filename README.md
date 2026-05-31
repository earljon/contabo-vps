# contabo-vps

A `cloud-init` configuration for bootstrapping a **Contabo VPS** running **Ubuntu 24.04 LTS** with Docker, security hardening, AWS CLI, Cloudflare Tunnel, CrowdSec, and Slack notifications.

---

## Table of Contents

- [What It Does](#what-it-does)
- [Bootstrap Stages](#bootstrap-stages)
- [Required Secrets](#required-secrets)
- [How to Generate Each Secret](#how-to-generate-each-secret)
- [Rendering the Template](#rendering-the-template)
- [Usage with Contabo](#usage-with-contabo)
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
7. cloudflared       — install package, register tunnel token, start service
8. Lockout guard     — verify SSH keys exist; revert to port 22 if empty (failsafe)
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

# Replace hostname placeholder (ERB-style) manually, then run envsubst
HOSTNAME="my-vps-01"
sed "s/<%= hostname %>/${HOSTNAME}/g" cloud-init.yaml \
  | envsubst > cloud-init-rendered.yaml
```

> **Important:** `envsubst` will also expand `${distro_id}` and `${distro_codename}` in the unattended-upgrades block. To protect those, enumerate only the variables you want substituted:
> ```bash
> envsubst '${SSH_PUBLIC_KEY} ${AWS_ACCESS_KEY_ID} ${AWS_SECRET_ACCESS_KEY} ${AWS_REGION} ${CLOUDFLARED_TUNNEL_TOKEN} ${SLACK_WEBHOOK_URL}' \
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
    HOSTNAME: my-vps-01
  run: |
    sed "s/<%= hostname %>/${HOSTNAME}/g" cloud-init.yaml \
      | envsubst '${SSH_PUBLIC_KEY} ${AWS_ACCESS_KEY_ID} ${AWS_SECRET_ACCESS_KEY} ${AWS_REGION} ${CLOUDFLARED_TUNNEL_TOKEN} ${SLACK_WEBHOOK_URL}' \
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

Cloud-init runs on first boot. Bootstrap takes approximately **3–6 minutes** depending on package download speed. You will receive Slack notifications as each stage completes.

---

## Post-Bootstrap Verification

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

**Slack not notifying:**

The notify script exits silently (`|| true`) if the webhook is unreachable or empty — bootstrap will still complete. Check the cloud-init log for stage output, and verify the webhook URL is correct and the Slack app is still active.
