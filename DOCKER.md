# RFD Docker Deployment Guide

This guide covers deploying the RFD stack (rfd-api, rfd-processor, rfd-site, PostgreSQL) using Docker on a DigitalOcean droplet.

## Overview

**What you'll deploy:**
- PostgreSQL database
- rfd-api (REST API server)
- rfd-processor (syncs RFDs from GitHub)
- rfd-site (web frontend)
- Caddy (reverse proxy with automatic HTTPS)

**Architecture:**
```
                    ┌─────────────────────────────────────────┐
                    │           DigitalOcean Droplet          │
                    │                                         │
Internet ──────────►│  Caddy (80/443)                        │
                    │    ├── rfd-api.yourdomain.com ──► rfd-api:8080
                    │    └── rfd.yourdomain.com ──────► rfd-site:3000
                    │                                         │
                    │  rfd-processor ◄──► GitHub              │
                    │         │                               │
                    │         ▼                               │
                    │     PostgreSQL                          │
                    └─────────────────────────────────────────┘
```

## Prerequisites

- macOS with Docker Desktop installed
- DigitalOcean account
- GitHub account
- Domain with DNS access
- `rfd-cli` installed locally: `cargo install --git https://github.com/oxidecomputer/rfd-api rfd-cli`

## Part 1: Build Docker Images (macOS)

Images must be built for `linux/amd64` architecture since DigitalOcean droplets run x86 Linux.

### 1.1 Clone rfd-hosting

```bash
git clone https://github.com/oblakeerickson/rfd-hosting.git
cd rfd-hosting/docker
```

### 1.2 Build rfd-api Image

This image contains rfd-api, rfd-processor, and rfd-installer.

```bash
docker build --platform linux/amd64 -t rfd-api:latest -f Dockerfile .
```

This takes 15-20 minutes on first build.

### 1.3 Build rfd-site Image

```bash
docker build --platform linux/amd64 -t rfd-site:latest -f Dockerfile.site .
```

### 1.4 Save Images to Tarballs

```bash
docker save rfd-api:latest | gzip > /tmp/rfd-api.tar.gz
docker save rfd-site:latest | gzip > /tmp/rfd-site.tar.gz
```

## Part 2: Create DigitalOcean Droplet

### 2.1 Create Droplet

1. Go to https://cloud.digitalocean.com/droplets/new
2. Choose **Ubuntu 24.04 LTS**
3. Select **Premium AMD** ($7/mo or higher)
4. Add your SSH key
5. Name it (e.g., `rfd-server`)
6. Create droplet

### 2.2 Configure DNS

Add A records pointing to your droplet's IP address:

| Type | Hostname | Value |
|------|----------|-------|
| A | rfd-api | `<droplet IP>` |
| A | rfd | `<droplet IP>` |

Wait a few minutes for DNS propagation.

## Part 3: Set Up Server

### 3.1 SSH into Droplet

```bash
ssh root@rfd-api.yourdomain.com
```

### 3.2 Install Docker

```bash
curl -fsSL https://get.docker.com | sh
```

Verify installation:

```bash
docker --version
docker compose version
```

### 3.3 Create Directory Structure

```bash
mkdir -p /opt/rfd
cd /opt/rfd
```

## Part 4: Upload Images and Config Files

### 4.1 Upload Docker Images (from macOS)

```bash
scp /tmp/rfd-api.tar.gz root@rfd-api.yourdomain.com:/tmp/
scp /tmp/rfd-site.tar.gz root@rfd-api.yourdomain.com:/tmp/
```

### 4.2 Upload Config Files (from macOS)

From the `rfd-hosting/docker` directory:

```bash
scp docker-compose.yml root@rfd-api.yourdomain.com:/opt/rfd/
scp .env.example root@rfd-api.yourdomain.com:/opt/rfd/.env
scp -r config root@rfd-api.yourdomain.com:/opt/rfd/
scp -r caddy root@rfd-api.yourdomain.com:/opt/rfd/
```

### 4.3 Load Docker Images (on server)

```bash
ssh root@rfd-api.yourdomain.com
cd /opt/rfd

gunzip -c /tmp/rfd-api.tar.gz | docker load
gunzip -c /tmp/rfd-site.tar.gz | docker load

# Verify
docker images
```

## Part 5: Configure for Pre-built Images

### 5.1 Edit docker-compose.yml

Since you're using pre-built images (not building on the server), edit `docker-compose.yml`:

```bash
vim docker-compose.yml
```

For `rfd-api`, `rfd-processor`, and `rfd-site`, comment out the `build:` sections and add `image:`:

```yaml
  rfd-api:
    # build:
    #   context: .
    #   dockerfile: Dockerfile
    image: rfd-api:latest
    ...

  rfd-processor:
    # build:
    #   context: .
    #   dockerfile: Dockerfile
    image: rfd-api:latest
    ...

  rfd-site:
    # build:
    #   context: .
    #   dockerfile: Dockerfile.site
    image: rfd-site:latest
    ...
```

## Part 6: Create GitHub Resources

Before configuring the RFD stack, you need to create several GitHub resources.

### 6.1 Create RFD Repository

Create a private GitHub repository to store your RFDs:

```bash
# On your local machine
gh repo create my-rfds --private
cd my-rfds
mkdir -p rfd/0001

cat > rfd/0001/README.adoc << 'EOF'
= RFD 1 Hello World
Your Name <your-email@example.com>
:state: published

== Introduction

This is my first RFD.
EOF

git add .
git commit -m "Add first RFD"
git push -u origin main
```

### 6.2 Create GitHub OAuth App for Web

1. Go to https://github.com/settings/developers
2. Click **New OAuth App**
3. Fill in:
   - **Application name:** `RFD Site`
   - **Homepage URL:** `https://rfd.yourdomain.com`
   - **Authorization callback URL:** `https://rfd-api.yourdomain.com/login/oauth/github/code/callback`
4. Click **Register application**
5. Note the **Client ID**
6. Click **Generate a new client secret** and save it

### 6.3 Create GitHub OAuth App for CLI

1. Go to https://github.com/settings/developers
2. Click **New OAuth App**
3. Fill in:
   - **Application name:** `RFD CLI`
   - **Homepage URL:** `https://rfd.yourdomain.com`
   - **Authorization callback URL:** `https://rfd-api.yourdomain.com/login/oauth/github/code/callback`
   - **Enable Device Flow:** ✅ Checked
4. Click **Register application**
5. Note the **Client ID**
6. Click **Generate a new client secret** and save it

### 6.4 Create GitHub App

1. Go to https://github.com/settings/apps/new
2. Fill in:
   - **GitHub App name:** `RFD API` (must be unique across GitHub)
   - **Homepage URL:** `https://rfd-api.yourdomain.com`
3. **Webhook:**
   - **Active:** ✅ Checked
   - **Webhook URL:** `https://rfd-api.yourdomain.com/github`
   - **Webhook secret:** Generate with `openssl rand -hex 32` and save it
4. **Repository permissions:**
   - **Contents:** Read and write
   - **Metadata:** Read-only (auto-selected)
   - **Pull requests:** Read and write
5. **Subscribe to events:**
   - ✅ Push
6. **Where can this GitHub App be installed?**
   - Select **Only on this account**
7. Click **Create GitHub App**
8. Note the **App ID** at the top of the page
9. Scroll to **Private keys** and click **Generate a private key**
10. Save the downloaded `.pem` file

### 6.5 Install GitHub App

1. In your GitHub App settings, click **Install App** in the left sidebar
2. Click **Install** next to your account
3. Select **Only select repositories** and choose your RFD repository
4. Click **Install**
5. Note the **Installation ID** from the URL: `https://github.com/settings/installations/INSTALLATION_ID`

## Part 7: Configure RFD Stack

### 7.1 Generate Secrets

On the server:

```bash
cd /opt/rfd

# Generate secrets
echo "DB_PASSWORD: $(openssl rand -hex 32)"
echo "GITHUB_WEBHOOK_KEY: <use the webhook secret from step 6.4>"
echo "SESSION_SECRET: $(openssl rand -hex 32)"
```

Save these values.

### 7.2 Generate JWT Keys

```bash
openssl genrsa -out private.pem 2048
openssl rsa -in private.pem -pubout -out public.pem
cat private.pem
cat public.pem
```

Save the key contents for config.toml.

### 7.3 Configure .env

```bash
vim .env
```

Fill in all values:

```
GITHUB_WEBHOOK_KEY=<from step 6.4>
DB_PASSWORD=<generated>
RFD_API=https://rfd-api.yourdomain.com
RFD_API_CLIENT_ID=<leave empty for now>
RFD_API_CLIENT_SECRET=<leave empty for now>
RFD_API_GITHUB_CALLBACK_URL=https://rfd.yourdomain.com/auth/github/callback
SESSION_SECRET=<generated>
```

### 7.4 Configure config.toml

```bash
cp config/config.toml.example config/config.toml
vim config/config.toml
```

Update these values:

```toml
database_url = "postgres://rfd:<DB_PASSWORD>@postgres:5432/rfd"
public_url = "https://rfd-api.yourdomain.com"

[jwt]
default_expiration = 3600

[[keys]]
kind = "local_signer"
kid = "rfd-key-1"
private = """
-----BEGIN PRIVATE KEY-----
<paste private.pem contents>
-----END PRIVATE KEY-----
"""

[[keys]]
kind = "local_verifier"
kid = "rfd-key-1"
public = """
-----BEGIN PUBLIC KEY-----
<paste public.pem contents>
-----END PUBLIC KEY-----
"""

[authn.oauth.github.web]
client_id = "<Web OAuth Client ID>"
client_secret = "<Web OAuth Client Secret>"
redirect_uri = "https://rfd-api.yourdomain.com/login/oauth/github/code/callback"

[authn.oauth.github.device]
client_id = "<CLI OAuth Client ID>"
client_secret = "<CLI OAuth Client Secret>"

[services.github]
owner = "<your-github-username>"
repo = "<your-rfd-repo-name>"
path = "rfd"
default_branch = "main"

[services.github.auth]
app_id = <GitHub App ID>
installation_id = <Installation ID>
private_key = """
-----BEGIN RSA PRIVATE KEY-----
<paste GitHub App private key contents>
-----END RSA PRIVATE KEY-----
"""

[search]
host = ""
key = ""
index = ""
```

### 7.5 Configure mappers.toml

```bash
cp config/mappers.toml.example config/mappers.toml
vim config/mappers.toml
```

Update the email address to your GitHub email:

```toml
[[mappers]]
name = "Initial admin"
rule = "email_address"
email = "your-github-email@example.com"
groups = ["admin"]
```

### 7.6 Configure processor-config.toml

```bash
cp config/processor-config.toml.example config/processor-config.toml
vim config/processor-config.toml
```

Update these values (same GitHub App credentials as config.toml):

```toml
database_url = "postgres://rfd:<DB_PASSWORD>@postgres:5432/rfd"

[auth.github]
app_id = <GitHub App ID>
installation_id = <Installation ID>
private_key = """
-----BEGIN RSA PRIVATE KEY-----
<paste GitHub App private key contents>
-----END RSA PRIVATE KEY-----
"""

[source]
owner = "<your-github-username>"
repo = "<your-rfd-repo-name>"
path = "rfd"
default_branch = "main"
```

### 7.7 Configure Caddyfile

```bash
vim caddy/Caddyfile
```

Replace with your actual domains:

```
rfd-api.yourdomain.com {
    reverse_proxy rfd-api:8080
}

rfd.yourdomain.com {
    reverse_proxy rfd-site:3000
}
```

## Part 8: Start Services

### 8.1 Start PostgreSQL

```bash
docker compose --profile db up -d postgres

# Wait for it to initialize
sleep 10
docker compose logs postgres
```

### 8.2 Run Database Migrations

```bash
docker compose run --rm rfd-api rfd-installer
```

This creates all necessary database tables.

### 8.3 Start rfd-api

```bash
docker compose up -d rfd-api
docker compose logs -f rfd-api
```

Wait until you see it running without errors. Press Ctrl+C to exit logs.

Verify it's working:

```bash
curl http://localhost:8080/.well-known/openid-configuration
```

### 8.4 Clear Mappers File

After the first successful start, clear mappers to prevent conflicts on restart:

```bash
cat > config/mappers.toml << 'EOF'
groups = []
mappers = []
EOF
```

### 8.5 Start rfd-processor

```bash
docker compose up -d rfd-processor
docker compose logs -f rfd-processor
```

Wait until it successfully scans your GitHub repo. Press Ctrl+C to exit logs.

### 8.6 Start Caddy

```bash
docker compose up -d caddy
docker compose logs -f caddy
```

Wait for SSL certificates to be issued. Press Ctrl+C to exit logs.

### 8.7 Test External Access

```bash
curl https://rfd-api.yourdomain.com/.well-known/openid-configuration
```

## Part 9: Configure rfd-site

### 9.1 Create OAuth Client for Site (from macOS)

```bash
# Configure CLI
rfd-cli config set host https://rfd-api.yourdomain.com

# Login via GitHub
rfd-cli auth login github

# Create OAuth client
CLIENT_ID=$(rfd-cli sys oauth create | jq -r '.id')
echo "Client ID: $CLIENT_ID"

# Add redirect URI
rfd-cli sys oauth redirect create \
  --client-id "$CLIENT_ID" \
  --redirect-uri "https://rfd.yourdomain.com/auth/github/callback"

# Create client secret
CLIENT_SECRET=$(rfd-cli sys oauth secret create --client-id "$CLIENT_ID" | jq -r '.key')
echo "Client Secret: $CLIENT_SECRET"
```

Save these values.

### 9.2 Update .env with OAuth Credentials (on server)

```bash
vim .env
```

Update:

```
RFD_API_CLIENT_ID=<Client ID from step 9.1>
RFD_API_CLIENT_SECRET=<Client Secret from step 9.1>
```

### 9.3 Start rfd-site

```bash
docker compose --profile frontend up -d rfd-site
docker compose logs -f rfd-site
```

You should see: `[react-router-serve] http://localhost:3000`

### 9.4 Test the Site

Open https://rfd.yourdomain.com in your browser. You should see your RFD site with your first RFD listed.

Try clicking "Sign in with GitHub" to test authentication.

## Part 10: Check All Services

```bash
docker compose ps
```

You should see all containers running:

```
NAME            IMAGE            STATUS
rfd-api         rfd-api:latest   Up
rfd-caddy       caddy:2-alpine   Up
rfd-postgres    postgres:14      Up
rfd-processor   rfd-api:latest   Up
rfd-site        rfd-site:latest  Up
```

## Updating/Redeploying

### Rebuild and Upload New Images (from macOS)

```bash
cd rfd-hosting/docker

# Rebuild
docker build --platform linux/amd64 -t rfd-api:latest -f Dockerfile .
docker build --platform linux/amd64 -t rfd-site:latest -f Dockerfile.site .

# Save
docker save rfd-api:latest | gzip > /tmp/rfd-api.tar.gz
docker save rfd-site:latest | gzip > /tmp/rfd-site.tar.gz

# Upload
scp /tmp/rfd-api.tar.gz root@rfd-api.yourdomain.com:/tmp/
scp /tmp/rfd-site.tar.gz root@rfd-api.yourdomain.com:/tmp/
```

### Load and Restart (on server)

```bash
cd /opt/rfd

# Stop services
docker compose down

# Remove old images
docker rmi rfd-api:latest rfd-site:latest

# Load new images
gunzip -c /tmp/rfd-api.tar.gz | docker load
gunzip -c /tmp/rfd-site.tar.gz | docker load

# Start services
docker compose --profile db --profile frontend up -d
```

## Troubleshooting

### Container Won't Start

Check logs:

```bash
docker compose logs <service-name>
```

### Config Changes Not Applied

Environment variable changes require recreating the container (not just restart):

```bash
docker compose rm -f <service-name>
docker compose up -d <service-name>
```

### "Missing configuration field" Errors

Check that all required fields are present in config files. Common missing fields:
- `config.toml`: `[jwt]` section, `[search]` section
- `processor-config.toml`: `processor_enabled`, `scanner_enabled`, `processor_batch_size`, `processor_capacity`

### Image Platform Mismatch

If you see "platform (linux/arm64) does not match", you built on Mac without the `--platform linux/amd64` flag. Rebuild with:

```bash
docker build --platform linux/amd64 -t rfd-api:latest -f Dockerfile .
```

### SSL Certificate Errors

Check Caddy logs:

```bash
docker compose logs caddy
```

Ensure DNS is properly configured and propagated:

```bash
dig rfd-api.yourdomain.com +short
dig rfd.yourdomain.com +short
```

### Processor Can't Find Branch

Ensure your RFD repository exists and has at least one commit on the main branch.

### Login Redirects to Wrong URL

Check that:
1. `.env` has `RFD_API=https://rfd-api.yourdomain.com` (public URL, not internal)
2. Container was recreated after changing `.env` (not just restarted)

### View All Logs

```bash
docker compose logs -f
```

### Check Database

```bash
docker compose exec postgres psql -U rfd -d rfd -c "\dt"
```

## Useful Commands

```bash
# View status
docker compose ps

# View logs for specific service
docker compose logs -f rfd-api

# Restart a service
docker compose restart rfd-api

# Stop everything
docker compose --profile db --profile frontend down

# Start everything
docker compose --profile db --profile frontend up -d

# Execute command in container
docker compose exec rfd-api rfd-installer

# Check environment variables in container
docker compose exec rfd-site env | grep RFD
```
