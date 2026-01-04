# RFD Site Hosting Guide

This is a guide on how to deploy the rfd-api and rfd-site repos into production.

If you want to setup your own RFD website just like Oxide has for your own company this guide is for you.

The current goal of this guide is to just learn how the rfd-api and rfd-site repos work and how to deploy them. This guide is not intended to supercede anything in rfd-api or rfd-site. As I document how things work I hope some useful changes will make it upstream to improve the original oxidecomputer repos.

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
V_ONLY=1 DATABASE_URL="postgres://user:pass@private-<do-db-server-name>.g.db.ondigitalocean.com:25060/rfd?sslmode=require" cargo run -p rfd-installer

DATABASE_URL="postgres://user:pass@private-<do-db-server-name>.g.db.ondigitalocean.com:25060/rfd?sslmode=require" diesel migration run
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
cd rfd-api
cp config.example.toml config.toml
```

#### Edit config.toml

```
vim config.toml
```

#### Remove cloud key section

Find the `[[keys]]` section and remove the `# CLOUD KMS - Signer` section. For this guide we will be using a local key.

```
cat private.pem | pbcopy
cat public.pem | pbcopy
```

```
[[keys]]
kind = "local_signer"
kid = "rfd-key-1"
private = """
-----BEGIN PRIVATE KEY-----
...
-----END PRIVATE KEY-----
"""

[[keys]]
kind = "local_verifier"
kid = "rfd-key-1"
public = """
-----BEGIN PUBLIC KEY-----
...
-----END PUBLIC KEY-----
"""
```

### Setup GitHub OAuth App

```
[authn.oauth.github.web]
client_id = ""
client_secret = ""
redirect_uri = "https://<rfd-api-hostname>/login/oauth/github/code/callback"
```

### Setup a private GitHub repo for the RFDs

```
git@github.com:oblakeerickson/rfd.git
```

Then add it to the config file:

```
# The GitHub repository to use to write RFDs
[services.github]
# GitHub user or organization
owner = ""
# GitHub repository name
repo = ""
# Path within the repository where RFDs are stored
path = ""
# Branch to use as the default branch of the repository
default_branch = ""
```

Setup up Fine-grained personal access tokens

contents, metadata, Pull requests (write access?)

delete the other `[services.github.auth]` section so that there is only one in the config file.

### Mappers

```
cp mappers.example.toml mappers.toml
```

#### Edit `config.toml` to point to the mappers file:

```
initial_mappers = "/opt/rfd-api/rfd-api/mappers.toml"
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

### Configure path for openapi spec file

Comment out the `output_path` line in `config.toml`:

```
# output_path = "/opt/rfd-api/spec/openapi.json"
```

### Configure reverse proxy


Install caddy:

```
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install caddy
```

Configure Caddy:

```
sudo vim /etc/caddy/Caddyfile
```

Replace contents with:

```
rfd2-api.blake.app {
    reverse_proxy localhost:8080
}
```

Verify it is working by visiting:

```
https://rfd2-api.blake.app/.well-known/openid-configuration
```

and you should see json output.

# Deploy rfd-site to Vercel

First we need to create an OAuth client in the rfd-api so the site can authenticate users.

```
rfd-cli config set host https://rfd2-api.blake.app
```
You should see: `Configuration updated`

```
rfd-cli auth login github
```

And you should see:

```
To complete login visit: https://github.com/login/device and enter ASDF-1234
Configuration updated

```

Now create the OAuth client:

```
# 1. Create the OAuth client
rfd-cli sys oauth create
# Returns: {"id":"<client-id>", ...}

# 2. Add redirect URI
rfd-cli sys oauth redirect create --client-id <client-id> --redirect-uri "https://rfd2.blake.app/auth/github/callback"

# 3. Create client secret
rfd-cli sys oauth secret create --client-id <client-id>
# Returns the secret to use in Vercel
```
