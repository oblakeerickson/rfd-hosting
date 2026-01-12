# PR Preview Deployments for rfd-site

This guide explains how to set up automatic PR preview deployments for **rfd-site code changes** using the `LOCAL_RFD_REPO` mode.

## Overview

When a PR is opened to your rfd-site fork, a preview environment is automatically spun up, allowing you to test UI/styling changes before merging.

**Key Benefits:**
- No rfd-api or database required
- Uses rfd-site's built-in `LOCAL_RFD_REPO` mode
- Each PR gets its own subdomain (e.g., `pr-123.preview.blake.app`)
- All previews share the same static test RFD content
- Automatic cleanup when PR is closed

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│  preview.blake.app (DigitalOcean Droplet)               │
│                                                         │
│  /home/deploy/test-rfds/        (static test content)   │
│                                                         │
│  Caddy (wildcard *.preview.blake.app)                   │
│    │                                                    │
│    ├── pr-123.preview.blake.app → localhost:3001        │
│    ├── pr-124.preview.blake.app → localhost:3002        │
│    └── pr-125.preview.blake.app → localhost:3003        │
│                                                         │
│  /home/deploy/previews/                                 │
│    ├── pr-123/site/  (rfd-site from PR branch)          │
│    ├── pr-124/site/                                     │
│    └── pr-125/site/                                     │
└─────────────────────────────────────────────────────────┘
```

## Prerequisites

- A fork of [oxidecomputer/rfd-site](https://github.com/oxidecomputer/rfd-site)
- A DigitalOcean account (or similar VPS provider)
- A domain with DNS access

## Setup Instructions

### Step 1: Fork rfd-site

```bash
# Fork oxidecomputer/rfd-site to your GitHub account, then clone:
git clone git@github.com:YOUR_USERNAME/rfd-site.git
cd rfd-site

# Add upstream remote for syncing
git remote add upstream https://github.com/oxidecomputer/rfd-site.git
```

### Step 2: Create a Test RFD Repository

Create a repository with sample RFD content for previews:

```bash
# Create a new repo for test RFDs
mkdir test-rfds
cd test-rfds
git init

# Create the rfd directory structure
mkdir -p rfd/0001 rfd/0002 rfd/0003

# Create sample RFDs
cat > rfd/0001/README.adoc << 'EOF'
= RFD 1 Sample Published RFD
Test Author <test@example.com>
:state: published

== Introduction

This is a sample RFD for testing the rfd-site preview environment.

== Background

Lorem ipsum dolor sit amet, consectetur adipiscing elit.

== Proposal

This RFD proposes a test proposal for preview purposes.
EOF

cat > rfd/0002/README.adoc << 'EOF'
= RFD 2 Sample Discussion RFD
Test Author <test@example.com>
:state: discussion

== Introduction

This RFD is in discussion state.

== Details

Some details about the proposal.
EOF

cat > rfd/0003/README.adoc << 'EOF'
= RFD 3 Sample Prediscussion RFD
Test Author <test@example.com>
:state: prediscussion

== Introduction

This RFD is in prediscussion state.

== Ideas

Some early ideas being explored.
EOF

git add .
git commit -m "Add sample RFDs for preview testing"
```

Push to GitHub:

```bash
gh repo create test-rfds --private --source=. --push
```

### Step 3: Create Preview Droplet

Create a DigitalOcean droplet:

| Setting | Value |
|---------|-------|
| Image | Ubuntu 24.04 |
| Size | Basic $12/mo (2GB RAM) |
| Region | Your preferred region |
| Hostname | preview.yourdomain.com |

### Step 4: Configure DNS

Add these DNS records pointing to your droplet's IP:

```
A     preview.yourdomain.com      → <droplet-ip>
A     *.preview.yourdomain.com    → <droplet-ip>
```

### Step 5: Set Up the Droplet

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

# Clone the test RFDs repo (one-time setup)
su - deploy -c "git clone https://github.com/YOUR_USERNAME/test-rfds.git /home/deploy/test-rfds"
```

### Step 6: Create Preview Management Scripts

#### start-preview.sh

Create `/home/deploy/scripts/start-preview.sh`:

```bash
#!/bin/bash
set -e

PR_NUMBER=$1
SITE_BRANCH=$2
PORT=$3

PREVIEW_DIR="/home/deploy/previews/pr-${PR_NUMBER}"
SITE_REPO="https://github.com/YOUR_USERNAME/rfd-site.git"
TEST_RFDS="/home/deploy/test-rfds"

echo "Starting preview for PR ${PR_NUMBER} on port ${PORT}..."

# Clean up existing preview if any
pm2 delete "preview-${PR_NUMBER}" 2>/dev/null || true
rm -rf "${PREVIEW_DIR}"

# Create preview directory
mkdir -p "${PREVIEW_DIR}"
cd "${PREVIEW_DIR}"

# Clone rfd-site at the PR branch
echo "Cloning rfd-site branch: ${SITE_BRANCH}..."
git clone --depth 1 --branch "${SITE_BRANCH}" "${SITE_REPO}" site
cd site
npm install

# Start the dev server with pm2
echo "Starting dev server..."
LOCAL_RFD_REPO="${TEST_RFDS}" PORT="${PORT}" pm2 start npm --name "preview-${PR_NUMBER}" -- run dev

echo "Preview PR-${PR_NUMBER} started on port ${PORT}"
```

#### stop-preview.sh

Create `/home/deploy/scripts/stop-preview.sh`:

```bash
#!/bin/bash
set -e

PR_NUMBER=$1
PREVIEW_DIR="/home/deploy/previews/pr-${PR_NUMBER}"

echo "Stopping preview for PR ${PR_NUMBER}..."

# Stop pm2 process
pm2 delete "preview-${PR_NUMBER}" 2>/dev/null || true

# Remove preview directory
rm -rf "${PREVIEW_DIR}"

echo "Preview PR-${PR_NUMBER} stopped and cleaned up"
```

#### update-caddy.sh

Create `/home/deploy/scripts/update-caddy.sh`:

```bash
#!/bin/bash
set -e

CADDYFILE="/etc/caddy/Caddyfile"
DOMAIN="preview.yourdomain.com"  # Update this!

echo "Updating Caddyfile..."

cat > "${CADDYFILE}" << EOF
# Auto-generated Caddyfile for PR previews
# Generated at: $(date)

EOF

# Find all running previews and add routes
pm2 jlist 2>/dev/null | jq -r '.[] | select(.name | startswith("preview-")) | "\(.name) \(.pm2_env.PORT // "unknown")"' | while read NAME PORT; do
    if [ "$PORT" != "unknown" ] && [ -n "$PORT" ]; then
        PR_NUM=$(echo $NAME | sed 's/preview-//')
        echo "Adding route: pr-${PR_NUM}.${DOMAIN} -> localhost:${PORT}"
        cat >> "${CADDYFILE}" << EOF
pr-${PR_NUM}.${DOMAIN} {
    reverse_proxy localhost:${PORT}
}

EOF
    fi
done

# Reload Caddy
sudo systemctl reload caddy
echo "Caddy reloaded"
```

#### list-previews.sh

Create `/home/deploy/scripts/list-previews.sh`:

```bash
#!/bin/bash
echo "Active previews:"
pm2 list | grep preview || echo "No active previews"
```

Make scripts executable:

```bash
chmod +x /home/deploy/scripts/*.sh
chown deploy:deploy /home/deploy/scripts/*.sh
```

### Step 7: Set Up SSH Key for GitHub Actions

As the deploy user:

```bash
su - deploy
ssh-keygen -t ed25519 -C "github-actions-preview" -f ~/.ssh/github_actions -N ""
cat ~/.ssh/github_actions.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys

# Display the private key (add this to GitHub secrets)
cat ~/.ssh/github_actions
```

### Step 8: Configure Sudoers for Caddy Reload

As root:

```bash
echo "deploy ALL=(ALL) NOPASSWD: /bin/systemctl reload caddy" >> /etc/sudoers.d/deploy-caddy
chmod 440 /etc/sudoers.d/deploy-caddy
```

### Step 9: Initial Caddy Config

Create `/etc/caddy/Caddyfile`:

```
# Initial config - will be populated by update-caddy.sh
```

Start Caddy:

```bash
systemctl enable caddy
systemctl start caddy
```

### Step 10: Add GitHub Action to Your rfd-site Fork

In your rfd-site fork, create `.github/workflows/preview.yml`:

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
              "${{ github.head_ref }}" \
              ${{ steps.port.outputs.port }}

            /home/deploy/scripts/update-caddy.sh

      - name: Comment PR with preview URL
        uses: actions/github-script@v7
        with:
          script: |
            const prNumber = context.payload.pull_request.number;
            const previewUrl = `https://pr-${prNumber}.${{ env.PREVIEW_HOST }}`;

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
            /home/deploy/scripts/update-caddy.sh
```

### Step 11: Add GitHub Secret

In your rfd-site fork:

1. Go to **Settings** → **Secrets and variables** → **Actions**
2. Click **New repository secret**
3. Name: `PREVIEW_SSH_KEY`
4. Value: The private key from Step 7

## Usage

1. Create a PR to your rfd-site fork
2. GitHub Actions automatically deploys a preview
3. A comment is added to the PR with the preview URL
4. The preview shows your UI changes with static test RFD content
5. When the PR is closed/merged, the preview is automatically cleaned up

## Syncing Your rfd-site Fork

To pull in updates from upstream oxidecomputer/rfd-site:

```bash
cd rfd-site
git fetch upstream
git checkout main
git merge upstream/main
git push origin main
```

Your `.github/workflows/preview.yml` won't conflict since it doesn't exist upstream.

## Troubleshooting

### Check pm2 processes

```bash
ssh deploy@preview.yourdomain.com
pm2 list
pm2 logs preview-123
```

### Check Caddy status

```bash
systemctl status caddy
cat /etc/caddy/Caddyfile
```

### Manual preview management

```bash
# Start a preview manually
/home/deploy/scripts/start-preview.sh 123 main 3001
/home/deploy/scripts/update-caddy.sh

# List active previews
/home/deploy/scripts/list-previews.sh

# Stop a preview manually
/home/deploy/scripts/stop-preview.sh 123
/home/deploy/scripts/update-caddy.sh

# Stop all previews
pm2 delete all
```

### Update test RFDs

```bash
cd /home/deploy/test-rfds
git pull
```

## Resource Considerations

Each preview runs a Node.js dev server consuming approximately:
- **RAM**: 200-400MB per preview
- **CPU**: Minimal when idle, spikes during page loads

A 2GB droplet can comfortably run 3-4 concurrent previews.

## Limitations

- **Dev mode only**: `LOCAL_RFD_REPO` only works in development mode
- **Static content**: All previews show the same test RFD content
- **No authentication**: Previews don't have login functionality (no rfd-api)
