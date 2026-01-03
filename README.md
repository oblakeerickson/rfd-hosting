# RFD Site Hosting Guide

This is a guide on how to deploy the rfd-api and rfd-site repos into production.

If you want to setup your own RFD website just like Oxide has for your own company this guide is for you.

# Accounts/Services you will need

- GitHub Account
- Digital Ocean Account
- Vercel Account

# Estimated Monthly Hosting Costs

```
$15.15/month Hosted Postgresql DB
$7/month Droplet
$0/month Vercel (Free Plan?)
===============================
$22.15/month
```

# Steps

1. Create a Digital Ocean Project: `rfd2.blake.app`
2. Create a Droplet: Ubuntu 24.04 (LTS) x64, Premium AMD NVMe SSD $7/mo
3. Create a Database: Postgres 14
4. SSH into the droplet

# Install Dependencies on the Droplet

```
# Update system
sudo apt update && sudo apt upgrade -y

# Install Rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source ~/.cargo/env

# Install Postgres client and Ruby
sudo apt install -y postgresql-client ruby ruby-dev build-essential

# Install Ruby gems
sudo gem install asciidoctor asciidoctor-pdf asciidoctor-mermaid rouge

# Install Node
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs

# Install Node packages
sudo npm install -g @mermaid-js/mermaid-cli
```

### Clone and Build rfd-api


```
sudo mkdir -p /opt/rfd-api
git clone https://github.com/oxidecomputer/rfd-api.git /opt/rfd-api
cd /opt/rfd-api
cargo build --release
```
