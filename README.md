# RFD Site Hosting Guide

![RFD Site Homepage](screenshots/rfd-site-homepage.png)

This is a guide on how to deploy the [rfd-api](https://github.com/oxidecomputer/rfd-api) and [rfd-site](https://github.com/oxidecomputer/rfd-site) repos into production.

If you want to setup your own RFD website just like Oxide has for your own company this guide is for you.

The current goal of this guide is to just learn how the rfd-api and rfd-site repos work and how to deploy them. This guide is not intended to supercede anything in rfd-api or rfd-site. As I document how things work I hope some useful changes will make it upstream to improve the original oxidecomputer repos:

- https://github.com/oxidecomputer/rfd-api
- https://github.com/oxidecomputer/rfd-site

# Docker Deployment

For a containerized setup, see the `docker/` directory. Quick start:

```bash
cd docker
cp .env.example .env
cp config/config.toml.example config/config.toml
cp config/mappers.toml.example config/mappers.toml
cp config/processor-config.toml.example config/processor-config.toml
# Edit configs with your values, then:
docker compose up -d
```

After first start, empty mappers to prevent conflicts: `echo -e "groups = []\nmappers = []" > config/mappers.toml`

The Docker setup runs rfd-api, rfd-processor, and Caddy. Database should be hosted externally.

---

# Manual Deployment

The following guide covers manual deployment on a VPS.

# Accounts/Services you will need

- GitHub Account
- Digital Ocean Account
- Vercel Account

# Estimated Monthly Hosting Costs

For a basic setup these are the current costs:

```
$15.15/month Hosted Postgresql DB
$7/month Droplet
$0/month Vercel (Free Plan?)
===============================
$22.15/month
```

By moving the DB to the droplet you could save on some costs.

# Steps

## Spin up a Droplet

1. Create a Digital Ocean Project: `RFD Site`
2. Create a Droplet: Ubuntu 24.04 (LTS) x64, Premium AMD NVMe SSD $7/mo
3. Point DNS to the droplet (use 'rfd-api.yourdomain.com' for the subdomain)
4. SSH into the droplet (`ssh root@rfd-api.yourdomain.com`)

### Add Swap

If you are building rfd-api from source you will need to add swap space to your droplet.

```
# Create 2GB swap file
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# Make it permanent
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

### Install Dependencies on the Droplet

Add the required repositories:
```
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
```

```
# Update system
sudo apt update && sudo DEBIAN_FRONTEND=noninteractive apt upgrade -y \
    -o Dpkg::Options::="--force-confdef" \
    -o Dpkg::Options::="--force-confold"


# Install Rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
source ~/.cargo/env

# Install Postgres client, Ruby, Node, Caddy, and other dependencies

sudo apt install -y \
  postgresql-client \
  ruby \
  ruby-dev \
  build-essential \
  libpq-dev \
  debian-keyring \
  debian-archive-keyring \
  apt-transport-https \
  curl \
  caddy \
  nodejs

# Install Ruby gems
sudo gem install asciidoctor asciidoctor-pdf asciidoctor-mermaid rouge

# Install Node packages
sudo npm install -g @mermaid-js/mermaid-cli
```

### Install Diesel CLI

```
cargo install diesel_cli --no-default-features --features postgres
```

### Clone and Build rfd-api

```
sudo mkdir -p /opt/rfd-api
sudo chown $USER:$USER /opt/rfd-api
git clone https://github.com/oxidecomputer/rfd-api.git /opt/rfd-api
cd /opt/rfd-api
cargo build --release
```

## Create the database and run migrations

Create a hosted postgres database at https://cloud.digitalocean.com/databases. Use PostgreSQL 14.

### Setup Initial Config

Before running migrations, create the config file with your database URL:

```bash
cd /opt/rfd-api/rfd-api
cp config.example.toml config.toml
```

Edit the config file and set the `database_url`:

```bash
vim config.toml
```

```toml
database_url = "postgres://user:pass@private-<do-db-server-name>.g.db.ondigitalocean.com:25060/rfd?sslmode=require"
```

You'll configure the remaining settings later. For now, only the `database_url` is needed.

### Create the db

```bash
psql "$(grep '^database_url' /opt/rfd-api/rfd-api/config.toml | cut -d'"' -f2 | sed 's|/rfd?|/defaultdb?|')" -c "CREATE DATABASE rfd;"
```

### Run migrations

Run the `rfd-installer` to run the v-api migrations:
```bash
cd /opt/rfd-api
V_ONLY=1 DATABASE_URL="$(grep '^database_url' /opt/rfd-api/rfd-api/config.toml | cut -d'"' -f2)" cargo run -p rfd-installer
```

Run the diesel migrations:
```bash
cd /opt/rfd-api/rfd-model
DATABASE_URL="$(grep '^database_url' /opt/rfd-api/rfd-api/config.toml | cut -d'"' -f2)" diesel migration run
```

### Setup Config Files

There are 3 config files we need to setup:

1. `rfd-api/config.toml` (already created with database_url)
2. `rfd-api/mappers.toml`
3. `rfd-processor/config.toml`

#### Generate RSA Key Pair for JWT Signing

On your local:
```
openssl genrsa -out private.pem 2048
openssl rsa -in private.pem -pubout -out public.pem
```

#### Edit config.toml

Continue editing the config file you created earlier:

```
cd /opt/rfd-api/rfd-api
vim config.toml
```

#### Basic settings

Add these settings (database_url is already set):

```toml
log_format = "pretty"
# log_directory = "/var/log/rfd-api"  # Comment out to use stdout
public_url = "https://rfd-api.yourdomain.com"
server_port = 8080
initial_mappers = "/opt/rfd-api/rfd-api/mappers.toml"
```

Comment out or remove the entire `[spec]` section:

```toml
# [spec]
# title = ""
# description = ""
# contact_url = ""
# contact_email = ""
# output_path = ""
```

#### Disable magic link authentication

Add this section to disable passwordless email login (not needed for GitHub OAuth):

```toml
[magic_link]
templates = []
```

#### Remove cloud key section

Find the `[[keys]]` section and remove the `# Cloud KMS - Signer` and `# Cloud KMS - Verifier` sections. For this guide we will be using a local key.

On your local:

```
cat private.pem | pbcopy
```

Add the private key to the config.toml file on the droplet:

```
[[keys]]
kind = "local_signer"
kid = "rfd-key-1"
private = """
-----BEGIN PRIVATE KEY-----
...
-----END PRIVATE KEY-----
"""
```

On your local:

```
cat private.pem | pbcopy
```

Add the public key to the config.toml file on the droplet:

```
[[keys]]
kind = "local_verifier"
kid = "rfd-key-1"
public = """
-----BEGIN PUBLIC KEY-----
...
-----END PUBLIC KEY-----
"""
```

### Setup GitHub OAuth App for Web

Visit https://github.com/settings/developers and create a new OAuth App.

- Application name: `RFD SITE`
- Homepage URL: `https://rfd.yourdomain.com`
- Authorization callback URL: `https://rfd-api.yourdomain.com/login/oauth/github/code/callback`
- Enable Device flow: `false`

Then add the client id and client secret to the config file:

```
[authn.oauth.github.web]
client_id = ""
client_secret = ""
redirect_uri = "https://<rfd-api-hostname>/login/oauth/github/code/callback"
```

### Setup GitHub OAuth App for CLI

Visit https://github.com/settings/developers and create a new OAuth App.

- Application name: `RFD CLI`
- Homepage URL: `https://rfd.yourdomain.com`
- Authorization callback URL: `https://rfd-api.yourdomain.com/login/oauth/github/code/callback` (not used, but required)
- Enable Device flow: `true`

Then add the client id and client secret to the config file:

```
[authn.oauth.github.device]
client_id = ""
client_secret = ""
```

### Setup a private GitHub repo for the RFDs

Create a private GitHub repo for the RFDs using the gh CLI:

```bash
cd ~/code
gh repo create rfd --private --clone
cd rfd
```

Or via GitHub web UI: go to https://github.com/new, create a new private repository named `rfd`, then clone it locally.

Then add it to the config file:

```
# The GitHub repository to use to write RFDs
[services.github]
# GitHub user or organization
owner = ""
# GitHub repository name
repo = "rfd"
# Path within the repository where RFDs are stored
path = "rfd"
# Branch to use as the default branch of the repository
default_branch = "main"
```

#### Setup a GitHub App

A GitHub App is required for the webhook integration. While you can use a Personal Access Token for basic functionality, webhooks will fail because the rfd-api expects the `installation` field in webhook payloads, which only GitHub Apps provide.

**Create the GitHub App:**

1. Go to https://github.com/settings/apps/new (or for an org: `https://github.com/organizations/YOUR_ORG/settings/apps/new`)

2. Fill in the basic information:
   - **GitHub App name:** `RFD API` (must be unique across GitHub)
   - **Homepage URL:** `https://rfd-api.yourdomain.com`

3. Disable the Webhook for now (we'll configure it separately):
   - **Active:** Unchecked (we'll enable after setup is complete)

4. Set the required permissions under **Repository permissions:**
   - **Contents:** Read and write
   - **Metadata:** Read-only (automatically selected)
   - **Pull requests:** Read and write

5. Under **Where can this GitHub App be installed?**
   - Select **Only on this account**

6. Click **Create GitHub App**

**Generate a private key:**

1. After creating the app, scroll down to **Private keys**
2. Click **Generate a private key**
3. A `.pem` file will be downloaded - keep this safe!

**Note the App ID:**

At the top of the app settings page, note the **App ID** (a number like `123456`).

**Install the App on your RFD repository:**

1. In the left sidebar, click **Install App**
2. Click **Install** next to your account
3. Select **Only select repositories** and choose your RFD repo
4. Click **Install**
5. After installation, note the **Installation ID** from the URL: `https://github.com/settings/installations/INSTALLATION_ID`

**Add to config.toml:**

Copy the contents of the downloaded `.pem` file and add to your config:

```toml
# GitHub App Installation
[services.github.auth]
app_id = 123456
installation_id = 789012
private_key = """
-----BEGIN RSA PRIVATE KEY-----
... your private key contents ...
-----END RSA PRIVATE KEY-----
"""
```

Delete the `token = ""` line if present - you can only have one authentication method.

### Mappers

The `initial_mappers` path was already set in Basic settings:

```
initial_mappers = "/opt/rfd-api/rfd-api/mappers.toml"
```

Now let's copy the example mappers file:

```
cp mappers.example.toml mappers.toml
```

#### Edit `mappers.toml`:

```
vim mappers.toml
```

**⚠️ IMPORTANT**: Permissions must use **variant names** (e.g., `"GetApiUserSelf"`), NOT scope strings (e.g., `"user:info:r"`). Using scope strings will cause 403 errors because they deserialize incorrectly.

```toml
[[groups]]
name = "admin"
permissions = [
  # User permissions
  "GetApiUserSelf",
  "GetApiUsersAssigned",
  "GetApiUsersAll",
  "CreateApiUser",
  "ManageApiUsersAssigned",
  "ManageApiUsersAll",
  "CreateUserApiProviderLinkToken",
  # API key permissions
  "GetApiKeysAssigned",
  "GetApiKeysAll",
  "CreateApiKeySelf",
  "CreateApiKeyAssigned",
  "CreateApiKeyAll",
  "ManageApiKeysAssigned",
  "ManageApiKeysAll",
  # Group permissions
  "GetGroupsJoined",
  "GetGroupsAll",
  "CreateGroup",
  "ManageGroupsAssigned",
  "ManageGroupsAll",
  "ManageGroupMembershipsAssigned",
  "ManageGroupMembershipsAll",
  # RFD permissions
  "GetRfdsAssigned",
  "GetRfdsAll",
  "CreateRfd",
  "UpdateRfdsAssigned",
  "UpdateRfdsAll",
  "ManageRfdsVisibilityAll",
  "GetDiscussionsAssigned",
  "GetDiscussionsAll",
  "SearchRfds",
  # OAuth client permissions
  "GetOAuthClientsAssigned",
  "GetOAuthClientsAll",
  "CreateOAuthClient",
  "ManageOAuthClientsAssigned",
  "ManageOAuthClientsAll"
]

[[mappers]]
name = "Initial admin"
rule = "email_address"
email = "your-github-email@example.com"
groups = [
  "admin"
]
```

### Setup rfd-processor config

```bash
cd /opt/rfd-api/rfd-processor
cp config.example.toml config.toml
vim config.toml
```

Update the following settings (use your RFD repo from the previous step):

```toml
log_format = "json"

# Set to "write" when ready to persist changes, use "read" for testing
processor_update_mode = "read"

# Database connection (copy from rfd-api/config.toml)
database_url = "<same as rfd-api/config.toml>"

# Enable GitHub-related actions (for basic setup without GCP/Google Drive/Meilisearch)
actions = [
  "CreatePullRequest",
  "UpdatePullRequest",
  "UpdateDiscussionUrl",
  "EnsureRfdWithPullRequestIsInValidState",
  "EnsureRfdOnDefaultIsInValidState",
]

# GitHub App authentication (same as rfd-api)
[auth.github]
app_id = 123456
installation_id = 789012
private_key = """
-----BEGIN RSA PRIVATE KEY-----
... your private key contents ...
-----END RSA PRIVATE KEY-----
"""

# GitHub repo settings - use YOUR repo from the previous step
[source]
owner = "<your-username-or-org>"
repo = "my-rfds"
path = "rfd"
default_branch = "main"
```

Delete any `token = ""` line if present - use the same GitHub App authentication as rfd-api.

Comment out or remove the `[[static_storage]]`, `[pdf_storage]`, and `[[search_storage]]` sections if you're not using those features.

### Setup GitHub Webhook (Requires GitHub App)

GitHub webhooks allow the rfd-api to immediately process RFD changes when you push to your repository, instead of waiting for the scanner's 15-minute interval.

**Important:** Webhooks require a GitHub App. The rfd-api webhook endpoint expects an `installation` field in the webhook payload, which is only included when webhooks are sent from a GitHub App installation. Regular repository webhooks (from Personal Access Tokens) will fail with a parsing error.

#### Generate a webhook secret

On your local machine, generate a random secret:

```bash
openssl rand -hex 32
```

Save this secret - you'll need it for both the GitHub App and the rfd-api systemd service.

#### Configure the webhook in your GitHub App

1. Go to https://github.com/settings/apps and click on your RFD API app
2. In the left sidebar, click **General**
3. Scroll down to the **Webhook** section
4. Configure the webhook:
   - **Active:** Checked
   - **Webhook URL:** `https://rfd-api.yourdomain.com/github`
   - **Webhook secret:** The secret you generated above
5. Click **Save changes**

6. Under **Subscribe to events**, ensure **Push** is checked (this should be automatic based on the Contents permission)

7. Click **Save changes**

#### Update the systemd service

Edit the rfd-api service file to include the webhook secret:

```bash
sudo vim /etc/systemd/system/rfd-api.service
```

Update the `Environment` line with your actual secret:

```ini
Environment="GITHUB_WEBHOOK_KEY=your-actual-secret-from-above"
```

Then reload and restart:

```bash
sudo systemctl daemon-reload
sudo systemctl restart rfd-api
```

#### Verify webhook delivery

After pushing a change to your RFD repo, check GitHub's webhook delivery history:

1. Go to **Settings** → **Webhooks** → click your webhook
2. Click **Recent Deliveries**
3. Successful deliveries show a green checkmark with status 202

You can also check the rfd-api logs:

```bash
sudo journalctl -u rfd-api -f
```

### Configure reverse proxy

Configure Caddy:

```
sudo vim /etc/caddy/Caddyfile
```

Replace contents with:

```
rfd-api.yourdomain.com {
    reverse_proxy localhost:8080
}
```

Verify the config and restart Caddy:

```bash
sudo caddy fmt --overwrite --config /etc/caddy/Caddyfile
sudo caddy validate --config /etc/caddy/Caddyfile
sudo systemctl restart caddy
```

### Start rfd-api

Start the server (it automatically looks for `rfd-api/config.toml`):

```bash
cd /opt/rfd-api
./target/release/rfd-api
```

Stop it with `Ctrl+C`:


The first run loads the mappers/groups into the database. Verify they were created:

```bash
psql "$(grep '^database_url' /opt/rfd-api/rfd-api/config.toml | cut -d'"' -f2)" \
  -c "SELECT name, permissions FROM access_groups;"
```

You should see your `admin` group with the permissions you defined.

Now save the original mappers file and replace it with an empty one:

```bash
mv rfd-api/mappers.toml rfd-api/mappers.toml.loaded
cat > rfd-api/mappers.toml << 'EOF'
groups = []
mappers = []
EOF
```

This is to prevent conflicts on restart:

```bash
root@rfd4-api:/opt/rfd-api# ./target/release/rfd-api
  2026-01-06T14:02:23.971291Z ERROR v_model::storage::postgres: error: Conflict
    at /root/.cargo/git/checkouts/v-api-d40051c45e59353f/f69a1b8/v-model/src/storage/postgres.rs:1453

```

Restart rfd-api to verify it starts without errors:

```bash
./target/release/rfd-api
```

Verify it is working (run on the droplet in another ssh session):

```bash
curl http://localhost:8080/.well-known/openid-configuration
```

You should see JSON output. Once that works, verify HTTPS is working:

```bash
curl https://rfd-api.yourdomain.com/.well-known/openid-configuration
```

### Run rfd-api as a systemd service

Create a systemd service file:

```bash
sudo vim /etc/systemd/system/rfd-api.service
```

Add the following:

```ini
[Unit]
Description=RFD API Server
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/opt/rfd-api
ExecStart=/opt/rfd-api/target/release/rfd-api
Restart=on-failure
RestartSec=5
# Required for GitHub webhook signature verification (see "Setup GitHub Webhook" section)
Environment="GITHUB_WEBHOOK_KEY=your-webhook-secret-here"

[Install]
WantedBy=multi-user.target
```

**Note:** Replace `your-webhook-secret-here` with the actual webhook secret you'll create in the "Setup GitHub Webhook" section below.

Enable and start the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable rfd-api
sudo systemctl start rfd-api
```

Check the status:

```bash
sudo systemctl status rfd-api
```

View logs:

```bash
sudo journalctl -u rfd-api -f
```

# Deploy rfd-site to Vercel

## Create OAuth client

First we need to create an OAuth client in the rfd-api so the site can authenticate users.

From your local machine, clone the rfd-api repo and install the CLI:

```bash
cd ~/code
git clone https://github.com/oxidecomputer/rfd-api.git
cd rfd-api
cargo install --path rfd-cli
```

This installs `rfd-cli` to `~/.cargo/bin` which should already be in your PATH.

Configure the CLI to point to your rfd-api server:

```bash
rfd-cli config set host https://rfd-api.yourdomain.com
```
You should see: `Configuration updated`

Login via GitHub:

```bash
rfd-cli auth login github
```

And you should see:

```
To complete login visit: https://github.com/login/device and enter ASDF-1234
Configuration updated

```

Now create the OAuth client:

```bash
# 1. Create the OAuth client and capture the client ID
CLIENT_ID=$(rfd-cli sys oauth create | jq -r '.id')
echo "Client ID: $CLIENT_ID"

# 2. Add redirect URI
rfd-cli sys oauth redirect create --client-id "$CLIENT_ID" --redirect-uri "https://rfd.yourdomain.com/auth/github/callback"

# 3. Create client secret and capture it
CLIENT_SECRET=$(rfd-cli sys oauth secret create --client-id "$CLIENT_ID" | jq -r '.key')
echo "Client Secret: $CLIENT_SECRET"
```

Save the client ID and secret - you'll need them for Vercel.

## Fork and deploy rfd-site

Fork the `oxidecomputer/rfd-site` repo to your GitHub account:

```bash
cd ~/code
gh repo fork oxidecomputer/rfd-site --clone --fork-name rfd4-site
cd my-rfd-site
```

Or via GitHub web UI: visit https://github.com/oxidecomputer/rfd-site and click "Fork", then customize the repository name.

Install the Vercel CLI and deploy:

```bash
npm i -g vercel
vercel
```

When prompted:
- **Set up and deploy?** → Yes
- **Which scope?** → Select your account/team
- **Link to existing project?** → No (for new project)
- **Project name?** → Enter your project name
- **Detected a repository. Connect it?** → **No** (avoids git author issues)
- **Modify settings?** → No

## Configure Environment Variables

In the Vercel dashboard (or via CLI), add these environment variables to the production environment:

| Variable | Value |
|----------|-------|
| `RFD_API` | `https://rfd-api.yourdomain.com` |
| `RFD_API_CLIENT_ID` | `<client id from step 1>` |
| `RFD_API_CLIENT_SECRET` | `<secret key from step 3>` |
| `RFD_API_GITHUB_CALLBACK_URL` | `https://rfd.yourdomain.com/auth/github/callback` |
| `SESSION_SECRET` | `<random string, e.g. openssl rand -hex 32>` |

Or via CLI:

```bash
# Generate a session secret first
SESSION_SECRET=$(openssl rand -hex 32)
echo "SESSION_SECRET: $SESSION_SECRET"

# Add each environment variable (you'll be prompted for values)
vercel env add SESSION_SECRET           # paste the generated secret above
vercel env add RFD_API                  # https://rfd-api.yourdomain.com
vercel env add RFD_API_CLIENT_ID        # client id from OAuth client creation
vercel env add RFD_API_CLIENT_SECRET    # secret from OAuth client creation
vercel env add RFD_API_GITHUB_CALLBACK_URL  # https://rfd.yourdomain.com/auth/github/callback
```

## Set Custom Domain

Add a custom domain via CLI:

```bash
vercel domains add rfd.yourdomain.com
```

Then add an A record in your DNS provider (e.g., DigitalOcean):

| Type | Hostname | Value |
|------|----------|-------|
| A | rfd | 76.76.21.21 |

Note: `76.76.21.21` is Vercel's standard IP for all custom domains.

Vercel will verify the domain and send you an email when it's ready.

## Redeploy

After adding environment variables, trigger a redeploy:

```bash
vercel --prod
```

Your RFD site should now be live at `https://rfd.yourdomain.com`!

# Create Your First RFD

Now that everything is deployed, add your first RFD to the repository you created earlier.

## Add an RFD to Your Repository

From your local machine, navigate to the RFD repo you created earlier and add your first RFD:

```bash
cd ~/code/rfd
mkdir -p rfd/0001
cat > rfd/0001/README.adoc << 'EOF'
= RFD 1 My First RFD
Your Name <you@example.com>
:state: published

== Introduction

This is my first RFD.

== Background

Add background information here.

== Proposal

Describe your proposal here.
EOF
```

**AsciiDoc format notes:**
- Title line: `= RFD 1 Title Here`
- Author line immediately after title (not as `:authors:` attribute)
- `:state:` attribute after author line
- Blank line before content sections

Push to GitHub:

```bash
git add .
git commit -m "Add first RFD"
git push
```

## Run rfd-processor to Sync

On the droplet, run the processor to sync RFDs from GitHub to the database:

```bash
ssh root@rfd-api.yourdomain.com
cd /opt/rfd-api
./target/release/rfd-processor
```

The processor will scan your GitHub repo and import the RFDs. Once complete, refresh your RFD site to see your first RFD!

![RFD Site Homepage](screenshots/rfd-site-homepage.png)

**Note:** By default, new RFDs are only visible to admins. To make an RFD public, use the CLI:

```bash
rfd-cli edit visibility --number 1 --visibility public
```

## Run rfd-processor as a systemd service

The `rfd-processor` is designed to run as a long-running service with built-in intervals:

- `scanner_interval` - how often to scan GitHub for RFD changes (default: 900 seconds / 15 minutes)
- `processor_interval` - how often to process job batches (default: 30 seconds)

### Create the service file

```bash
sudo vim /etc/systemd/system/rfd-processor.service
```

Add the following:

```ini
[Unit]
Description=RFD Processor
After=network.target rfd-api.service

[Service]
Type=simple
User=root
WorkingDirectory=/opt/rfd-api
ExecStart=/opt/rfd-api/target/release/rfd-processor
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### Enable and start the service

```bash
sudo systemctl daemon-reload
sudo systemctl enable rfd-processor
sudo systemctl start rfd-processor
```

### Check status

```bash
# View service status
sudo systemctl status rfd-processor

# View processor logs
sudo journalctl -u rfd-processor -f
```

### Configure scan interval

To change how often the processor scans GitHub, edit the processor config:

```bash
vim /opt/rfd-api/rfd-processor/config.toml
```

```toml
# How often the processor scanner should check the remote GitHub repo for RFDs (in seconds)
# Default: 900 (15 minutes). Adjust based on your repo size and GitHub rate limits.
scanner_interval = 900

# How often to select a batch of jobs to process (in seconds)
processor_interval = 30
```

Then restart the service:

```bash
sudo systemctl restart rfd-processor
```
