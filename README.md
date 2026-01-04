# RFD Site Hosting Guide

This is a guide on how to deploy the [rfd-api](https://github.com/oxidecomputer/rfd-api) and [rfd-site](https://github.com/oxidecomputer/rfd-site) repos into production.

If you want to setup your own RFD website just like Oxide has for your own company this guide is for you.

The current goal of this guide is to just learn how the rfd-api and rfd-site repos work and how to deploy them. This guide is not intended to supercede anything in rfd-api or rfd-site. As I document how things work I hope some useful changes will make it upstream to improve the original oxidecomputer repos:

- https://github.com/oxidecomputer/rfd-api
- https://github.com/oxidecomputer/rfd-site

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
sudo apt update && sudo apt upgrade -y

# Install Rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
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

### Create the db

```bash
psql "postgres://user:pass@private-<do-db-server-name>.g.db.ondigitalocean.com:25060/defaultdb?sslmode=require" -c "CREATE DATABASE rfd;"
```

### Run migrations

rfd-installer runs v-api migrations
```
cd rfd-model
V_ONLY=1 \
DATABASE_URL="postgres://user:pass@private-<do-db-server-name>.g.db.ondigitalocean.com:25060/rfd?sslmode=require" \
cargo run -p rfd-installer

DATABASE_URL="postgres://user:pass@private-<do-db-server-name>.g.db.ondigitalocean.com:25060/rfd?sslmode=require" \
diesel migration run
```

### Setup Config Files

There are 3 config files we need to setup:

1. `rfd-api/config.toml`
2. `rfd-api/mappers.toml`
3. `rfd-processor/config.toml`

#### Generate RSA Key Pair for JWT Signing

On your local:
```
openssl genrsa -out private.pem 2048
openssl rsa -in private.pem -pubout -out public.pem
```

#### Copy example

```
cd /opt/rfd-api/rfd-api
cp config.example.toml config.toml
```

#### Edit config.toml

```
vim config.toml
```

#### Basic settings

```toml
log_format = "pretty"
# log_directory = "/var/log/rfd-api"  # Comment out to use stdout
public_url = "https://rfd-api.yourdomain.com"
server_port = 8080
database_url = "postgres://user:pass@private-<do-db-server-name>.g.db.ondigitalocean.com:25060/rfd?sslmode=require"
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

Go to GitHub.com and create a new private GitHub repo for the RFDs.

```
git@github.com:<your-username-or-org>/rfd.git
```

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

#### Setup up Fine-grained personal access tokens

Go to https://github.com/settings/personal-access-tokens and create a new Fine-grained personal access token for the private RFD repo.

**Required Permissions:**
- **Contents**: Read and write (to read/write RFD files)
- **Metadata**: Read-only (required for all tokens)
- **Pull requests**: Read and write (to create/update PRs for RFD changes)

```
# Access Token
[services.github.auth]
token = "<your-access-token>"
```

Delete the other `[services.github.auth]` section (the App Installation one) so that there is only one in the config file.

### Mappers

#### Edit `config.toml` to point to the mappers file:

```
initial_mappers = "/opt/rfd-api/rfd-api/mappers.toml"
```

That should be the last thing in `config.toml`. Now let's copy the example mappers file:

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

### Create Your RFD Repository

Before configuring rfd-processor, create a GitHub repository to store your RFDs. From your local machine:

```bash
cd ~/code
mkdir my-rfds
cd my-rfds
git init
mkdir -p rfd/0001
echo "# RFDs" > README.md
git add .
git commit -m "Initial commit"
gh repo create my-rfds --private --source=. --push
```

Note the owner and repo name (e.g., `oblakeerickson/my-rfds`) - you'll need these for the config.

### Setup rfd-processor config

```bash
cd /opt/rfd-api/rfd-processor
cp config.example.toml config.toml
vim config.toml
```

Update the following settings (use your RFD repo from the previous step):

```toml
log_format = "pretty"

# Set to "write" when ready to persist changes, use "read" for testing
processor_update_mode = "read"

# Database connection (same as rfd-api)
database_url = "postgres://user:pass@private-<do-db-server-name>.g.db.ondigitalocean.com:25060/rfd?sslmode=require"

# Enable GitHub-related actions (for basic setup without GCP/Google Drive/Meilisearch)
actions = [
  "CreatePullRequest",
  "UpdatePullRequest",
  "UpdateDiscussionUrl",
  "EnsureRfdWithPullRequestIsInValidState",
  "EnsureRfdOnDefaultIsInValidState",
]

# GitHub access token (same token as rfd-api)
[auth.github]
token = "<your-access-token>"

# GitHub repo settings - use YOUR repo from the previous step
[source]
owner = "<your-username-or-org>"
repo = "my-rfds"
path = "rfd"
default_branch = "main"
```

**Actions reference:**

| Action | Requires | Description |
|--------|----------|-------------|
| `CopyImagesToStorage` | `[[static_storage]]` (GCP bucket) | Copies images from RFDs to cloud storage |
| `UpdateSearch` | `[[search_storage]]` (Meilisearch) | Indexes RFD content for search |
| `UpdatePdfs` | `[pdf_storage]` (Google Drive) | Generates PDFs of RFDs |
| `CreatePullRequest` | GitHub token | Creates PRs for new RFDs |
| `UpdatePullRequest` | GitHub token | Updates existing RFD PRs |
| `UpdateDiscussionUrl` | GitHub token | Updates discussion links in RFDs |
| `EnsureRfdWithPullRequestIsInValidState` | GitHub token | Validates RFD state on PRs |
| `EnsureRfdOnDefaultIsInValidState` | GitHub token | Validates RFD state on main branch |

Delete the App Installation `[auth.github]` section (keep only the token-based one).

Comment out or remove the `[[static_storage]]`, `[pdf_storage]`, and `[[search_storage]]` sections if you're not using those features.

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
sudo caddy validate --config /etc/caddy/Caddyfile
sudo systemctl restart caddy
```

### Start rfd-api

Start the server (it automatically looks for `rfd-api/config.toml`):

```bash
cd /opt/rfd-api
./target/release/rfd-api
```

The first run loads the mappers/groups into the database. Verify they were created:

```bash
psql "postgres://user:pass@private-<do-db-server-name>.g.db.ondigitalocean.com:25060/rfd?sslmode=require" \
  -c "SELECT name, permissions FROM access_groups;"
```

You should see your `admin` group with the permissions you defined.

Now save the original mappers file and replace it with an empty one to prevent conflicts on restart:

```bash
mv rfd-api/mappers.toml rfd-api/mappers.toml.loaded
cat > rfd-api/mappers.toml << 'EOF'
groups = []
mappers = []
EOF
```

Restart rfd-api to verify it starts without errors:

```bash
./target/release/rfd-api
```

Verify it is working (run locally on the droplet):

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

[Install]
WantedBy=multi-user.target
```

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
# 1. Create the OAuth client
rfd-cli sys oauth create
# Returns: {"id":"<client-id>", ...}

# 2. Add redirect URI
rfd-cli sys oauth redirect create --client-id <client-id> --redirect-uri "https://rfd.yourdomain.com/auth/github/callback"

# 3. Create client secret
rfd-cli sys oauth secret create --client-id <client-id>
# Returns the secret to use in Vercel
```

Save the client ID and secret - you'll need them for Vercel.

## Fork and deploy rfd-site

Fork the `oxidecomputer/rfd-site` repo to your GitHub account:

```bash
cd ~/code
gh repo fork oxidecomputer/rfd-site --clone --fork-name my-rfd-site
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

In the Vercel dashboard (or via CLI), add these environment variables:

| Variable | Value |
|----------|-------|
| `RFD_API` | `https://rfd-api.yourdomain.com` |
| `RFD_API_CLIENT_ID` | `<client id from step 1>` |
| `RFD_API_CLIENT_SECRET` | `<secret key from step 3>` |
| `RFD_API_GITHUB_CALLBACK_URL` | `https://rfd.yourdomain.com/auth/github/callback` |
| `SESSION_SECRET` | `<random string, e.g. openssl rand -hex 32>` |

Or via CLI:

```bash
# Generate a session secret
openssl rand -hex 32

vercel env add RFD_API
vercel env add RFD_API_CLIENT_ID
vercel env add RFD_API_CLIENT_SECRET
vercel env add RFD_API_GITHUB_CALLBACK_URL
vercel env add SESSION_SECRET
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

From your local machine, navigate to your RFD repo and create an AsciiDoc file for RFD 1:

```bash
cd ~/code/my-rfds
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
gh repo create my-rfds --private --source=. --push
```

## Run rfd-processor to Sync

On the droplet, run the processor to sync RFDs from GitHub to the database:

```bash
ssh root@rfd-api.yourdomain.com
cd /opt/rfd-api
./target/release/rfd-processor
```

The processor will scan your GitHub repo and import the RFDs. Once complete, refresh your RFD site to see your first RFD!

**Note:** By default, new RFDs are only visible to admins. To make an RFD public, use the CLI:

```bash
rfd-cli edit visibility --number 1 --visibility public
```
