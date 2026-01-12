# PR Preview Deployments

This guide explains how to set up automatic PR preview deployments for RFD content using the `LOCAL_RFD_REPO` mode of rfd-site.

## Overview

When a PR is opened to your RFD content repository, a preview environment is automatically spun up, allowing you to view the rendered RFD before merging.

**Key Benefits:**
- No API, database, or OAuth required
- Uses rfd-site's built-in `LOCAL_RFD_REPO` mode
- Each PR gets its own subdomain (e.g., `pr-123.preview.blake.app`)
- Automatic cleanup when PR is closed

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│  preview.blake.app (DigitalOcean Droplet)               │
│                                                         │
│  Caddy (wildcard *.preview.blake.app)                   │
│    │                                                    │
│    ├── pr-123.preview.blake.app → localhost:3001        │
│    ├── pr-124.preview.blake.app → localhost:3002        │
│    └── pr-125.preview.blake.app → localhost:3003        │
│                                                         │
│  /home/deploy/previews/                                 │
│    ├── pr-123/  (rfd-site + cloned rfd content)         │
│    ├── pr-124/                                          │
│    └── pr-125/                                          │
└─────────────────────────────────────────────────────────┘
```

## Prerequisites

- A fork of [oxidecomputer/rfd-site](https://github.com/oxidecomputer/rfd-site)
- An RFD content repository with your RFDs
- A DigitalOcean account (or similar VPS provider)
- A domain with DNS access

## Setup Instructions

### Step 1: Fork rfd-site

Fork `oxidecomputer/rfd-site` to your GitHub account, then clone locally:

```bash
git clone git@github.com:YOUR_USERNAME/rfd-site.git rfd-site-fork
cd rfd-site-fork

# Add upstream remote for syncing
git remote add upstream https://github.com/oxidecomputer/rfd-site.git

# To sync with upstream later:
git fetch upstream
git merge upstream/main
```

### Step 2: Create Preview Droplet

Create a DigitalOcean droplet:

| Setting | Value |
|---------|-------|
| Image | Ubuntu 24.04 |
| Size | Basic $12/mo (2GB RAM) |
| Region | Your preferred region |
| Hostname | preview.yourdomain.com |

### Step 3: Configure DNS

Add these DNS records pointing to your droplet's IP:

```
A     preview.yourdomain.com      → <droplet-ip>
A     *.preview.yourdomain.com    → <droplet-ip>
```

### Step 4: Set Up the Droplet

SSH into the droplet and run:

```bash
# Update system
apt update && apt upgrade -y

# Install Node.js 20
curl -fsSL https://deb.nodesource.com/setup_20.x | bash -
apt install -y nodejs

# Install Caddy
apt install -y debian-keyring debian-archive-keyring apt-transport-https curl
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | tee /etc/apt/sources.list.d/caddy-stable.list
apt update
apt install -y caddy

# Install jq (for JSON parsing in scripts)
apt install -y jq

# Install pm2 for process management
npm install -g pm2

# Create deploy user
useradd -m -s /bin/bash deploy
mkdir -p /home/deploy/previews
mkdir -p /home/deploy/scripts
chown -R deploy:deploy /home/deploy
```

### Step 5: Create Preview Management Scripts

#### start-preview.sh

Create `/home/deploy/scripts/start-preview.sh`:

```bash
#!/bin/bash
set -e

PR_NUMBER=$1
RFD_REPO_URL=$2
RFD_BRANCH=$3
PORT=$4

PREVIEW_DIR="/home/deploy/previews/pr-${PR_NUMBER}"
SITE_REPO="https://github.com/YOUR_USERNAME/rfd-site.git"  # Update this!

# Clean up existing preview if any
pm2 delete "preview-${PR_NUMBER}" 2>/dev/null || true
rm -rf "${PREVIEW_DIR}"

# Create preview directory
mkdir -p "${PREVIEW_DIR}"
cd "${PREVIEW_DIR}"

# Clone rfd-site
git clone --depth 1 "${SITE_REPO}" site
cd site
npm install

# Clone RFD content repo (the PR branch)
cd "${PREVIEW_DIR}"
git clone --depth 1 --branch "${RFD_BRANCH}" "${RFD_REPO_URL}" rfd-content

# Start the dev server with pm2
cd "${PREVIEW_DIR}/site"
LOCAL_RFD_REPO="${PREVIEW_DIR}/rfd-content" PORT="${PORT}" pm2 start npm --name "preview-${PR_NUMBER}" -- run dev

echo "Preview started on port ${PORT}"
```

#### stop-preview.sh

Create `/home/deploy/scripts/stop-preview.sh`:

```bash
#!/bin/bash
set -e

PR_NUMBER=$1
PREVIEW_DIR="/home/deploy/previews/pr-${PR_NUMBER}"

# Stop pm2 process
pm2 delete "preview-${PR_NUMBER}" 2>/dev/null || true

# Remove preview directory
rm -rf "${PREVIEW_DIR}"

echo "Preview ${PR_NUMBER} stopped and cleaned up"
```

#### update-caddy.sh

Create `/home/deploy/scripts/update-caddy.sh`:

```bash
#!/bin/bash
# Regenerate Caddyfile based on active previews

CADDYFILE="/etc/caddy/Caddyfile"
DOMAIN="preview.yourdomain.com"  # Update this!

cat > "${CADDYFILE}" << EOF
# Auto-generated Caddyfile for PR previews
# Generated at: $(date)

EOF

# Find all running previews and add routes
for proc in $(pm2 jlist 2>/dev/null | jq -r '.[] | select(.name | startswith("preview-")) | "\(.name):\(.pm2_env.PORT // empty)"' 2>/dev/null); do
    NAME=$(echo $proc | cut -d: -f1)
    PORT=$(echo $proc | cut -d: -f2)

    if [ -n "$PORT" ]; then
        PR_NUM=$(echo $NAME | sed 's/preview-//')

        cat >> "${CADDYFILE}" << EOF
pr-${PR_NUM}.${DOMAIN} {
    reverse_proxy localhost:${PORT}
}

EOF
    fi
done

# Reload Caddy
systemctl reload caddy
```

Make scripts executable:

```bash
chmod +x /home/deploy/scripts/*.sh
chown deploy:deploy /home/deploy/scripts/*.sh
```

### Step 6: Set Up SSH Key for GitHub Actions

As the deploy user:

```bash
su - deploy
ssh-keygen -t ed25519 -C "github-actions-preview" -f ~/.ssh/github_actions -N ""
cat ~/.ssh/github_actions.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys

# Display the private key (add this to GitHub secrets)
cat ~/.ssh/github_actions
```

### Step 7: Configure Sudoers for Caddy Reload

As root:

```bash
echo "deploy ALL=(ALL) NOPASSWD: /bin/systemctl reload caddy" >> /etc/sudoers.d/deploy-caddy
chmod 440 /etc/sudoers.d/deploy-caddy
```

### Step 8: Initial Caddy Config

Create `/etc/caddy/Caddyfile`:

```
# Initial config - will be populated by update-caddy.sh
```

Start Caddy:

```bash
systemctl enable caddy
systemctl start caddy
```

### Step 9: Add GitHub Action to Your RFD Content Repo

In your RFD content repository (e.g., `your-username/rfd`), create `.github/workflows/preview.yml`:

```yaml
name: PR Preview

on:
  pull_request:
    types: [opened, synchronize, reopened, closed]

env:
  PREVIEW_HOST: preview.yourdomain.com  # Update this!
  PREVIEW_USER: deploy
  BASE_PORT: 3001

jobs:
  deploy-preview:
    if: github.event.action != 'closed'
    runs-on: ubuntu-latest
    steps:
      - name: Calculate port
        id: port
        run: echo "port=$(( ${{ env.BASE_PORT }} + (${{ github.event.pull_request.number }} % 100) ))" >> $GITHUB_OUTPUT

      - name: Deploy preview
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ env.PREVIEW_HOST }}
          username: ${{ env.PREVIEW_USER }}
          key: ${{ secrets.PREVIEW_SSH_KEY }}
          script: |
            /home/deploy/scripts/start-preview.sh \
              ${{ github.event.pull_request.number }} \
              "https://github.com/${{ github.repository }}.git" \
              "${{ github.head_ref }}" \
              ${{ steps.port.outputs.port }}

            sudo /home/deploy/scripts/update-caddy.sh

      - name: Comment PR with preview URL
        uses: actions/github-script@v7
        with:
          script: |
            const prNumber = context.payload.pull_request.number;
            const previewUrl = `https://pr-${prNumber}.${{ env.PREVIEW_HOST }}`;

            // Check if we already commented
            const comments = await github.rest.issues.listComments({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: prNumber,
            });

            const botComment = comments.data.find(c =>
              c.user.type === 'Bot' && c.body.includes('Preview deployment')
            );

            const body = `### Preview deployment

            Your preview is ready at: ${previewUrl}

            _Updated: ${new Date().toISOString()}_`;

            if (botComment) {
              await github.rest.issues.updateComment({
                owner: context.repo.owner,
                repo: context.repo.repo,
                comment_id: botComment.id,
                body,
              });
            } else {
              await github.rest.issues.createComment({
                owner: context.repo.owner,
                repo: context.repo.repo,
                issue_number: prNumber,
                body,
              });
            }

  cleanup-preview:
    if: github.event.action == 'closed'
    runs-on: ubuntu-latest
    steps:
      - name: Cleanup preview
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ env.PREVIEW_HOST }}
          username: ${{ env.PREVIEW_USER }}
          key: ${{ secrets.PREVIEW_SSH_KEY }}
          script: |
            /home/deploy/scripts/stop-preview.sh ${{ github.event.pull_request.number }}
            sudo /home/deploy/scripts/update-caddy.sh
```

### Step 10: Add GitHub Secret

In your RFD content repository:

1. Go to **Settings** → **Secrets and variables** → **Actions**
2. Click **New repository secret**
3. Name: `PREVIEW_SSH_KEY`
4. Value: The private key from Step 6

## Usage

1. Create a PR to your RFD content repository
2. GitHub Actions automatically deploys a preview
3. A comment is added to the PR with the preview URL
4. When the PR is closed/merged, the preview is automatically cleaned up

## Syncing Your rfd-site Fork

To pull in updates from the upstream oxidecomputer/rfd-site:

```bash
cd rfd-site-fork
git fetch upstream
git checkout main
git merge upstream/main
git push origin main
```

Your custom workflows in `.github/workflows/` won't conflict since they don't exist in the upstream repo.

## Troubleshooting

### Check pm2 processes

```bash
pm2 list
pm2 logs preview-123
```

### Check Caddy status

```bash
systemctl status caddy
cat /etc/caddy/Caddyfile
```

### Manual cleanup

```bash
# Stop all previews
pm2 delete all

# Remove all preview directories
rm -rf /home/deploy/previews/*
```

### Port conflicts

The port calculation uses `BASE_PORT + (PR_NUMBER % 100)`, so PRs 1, 101, 201 would all use the same port. For most use cases this is fine, but if you have many concurrent PRs, consider a different port allocation strategy.

## Resource Considerations

Each preview runs a Node.js dev server consuming approximately:
- **RAM**: 200-400MB per preview
- **CPU**: Minimal when idle, spikes during page loads

A 2GB droplet can comfortably run 3-4 concurrent previews. Scale up if you need more.

## Limitations

- **Dev mode only**: The `LOCAL_RFD_REPO` feature only works in development mode, not production builds
- **Single branch**: Each preview shows RFDs from one branch only
- **No authentication**: Previews are publicly accessible (consider HTTP basic auth if needed)

## Security Considerations

- Preview URLs are predictable (`pr-NUMBER.preview.domain.com`)
- Anyone with the URL can view the preview
- Consider adding HTTP basic auth in Caddy if your RFDs contain sensitive information:

```
pr-123.preview.yourdomain.com {
    basicauth {
        preview $2a$14$... # Generate with: caddy hash-password
    }
    reverse_proxy localhost:3001
}
```
