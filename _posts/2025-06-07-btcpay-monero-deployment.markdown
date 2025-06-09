---
layout: post
title:  "BTCPay with Bitcoin and Monero on External Disk - Docker Deployment Guide"
date:   2025-06-07 08:45:02 +0200
categories: docker btcpay bitcoin monero deployment
---

This post guides you through deploying BTCPay Server with Bitcoin and Monero on an external disk using Docker. This configuration is ideal for those aiming to keep blockchain data separate from the main system drive, ensuring better performance and easier management.

## Prerequisites

Before you begin, ensure you have the following:

- A server or computer with Docker installed.
   - Recommended OS: Ubuntu 20.04 LTS or Debian 10/11.
   - CPU: 2 cores or more.
   - 4 GB RAM minimum (8 GB or more recommended).
   - At least 50 GB of free disk space for the operating system and Docker.
   - Attached external disk with at least 1 TB capacity for blockchain data.
- Basic knowledge of Docker and command-line operations.
- A domain name or local hostname (e.g., `btcpay.local`) for accessing BTCPay Server.
- SSH access to your server for remote management if needed.
- Ensure your server has a static IP address or a dynamic DNS service configured for consistent access.

## Important Notes
- This guide assumes you would like to run both Bitcoin and Monero full nodes alongside BTCPay Server. 

- If you would like to save space, bitcoind can be configured to run in pruned mode by setting the environment variable `opt-save-storage-xs` in step 7.

## Step 1: Prepare the External Disk

1. Connect your external disk to the server.
2. Format the disk to a suitable filesystem (e.g., ext4 for Linux):

   ```bash
   sudo mkfs.ext4 /dev/sdX1
   ```

   Replace `/dev/sdX1` with your disk's identifier.

3. Create a mount point and mount the disk:

   ```bash
   sudo mkdir -p /mnt/btcpay-data
   sudo mount /dev/sdX1 /mnt/btcpay-data
   ```

4. Ensure the directory has the correct permissions for Docker to access it:

   ```bash
   sudo chown -R 1000:1000 /mnt/btcpay-data
   sudo chmod -R 700 /mnt/btcpay-data
   ```

5. Verify that the directory is accessible:

   ```bash
   ls -l /mnt/btcpay-data
   ```

## Step 2: Install Required Packages

Install `fail2ban`, `git`, and `avahi-daemon`:

```bash
sudo apt update
sudo apt install -y fail2ban git avahi-daemon
```

## Step 3: Configure the Firewall

Set up UFW to allow necessary ports:

```bash
sudo apt install -y ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp   # SSH
sudo ufw allow 80/tcp   # HTTP
sudo ufw allow 443/tcp  # HTTPS
sudo ufw allow 8333/tcp # Bitcoin
sudo ufw allow 9735/tcp # Lightning Network
sudo ufw allow 18080/tcp # Monero
sudo ufw enable
sudo ufw status
```

## Step 4: Install Docker

Install Docker to manage containers:

```bash
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
apt-cache policy docker-ce
sudo apt install -y docker-ce
sudo systemctl enable docker
sudo systemctl start docker
sudo systemctl status docker
```

## Step 5: Configure Docker Volumes to Use External Disk

Redirect Docker's Volumes to the external disk:

```bash
fdisk /dev/sda
# type 'p' to list existing partitions
# type 'd' to delete currently selected partitions
# type 'n' to create a new partition
# type 'w' to write the new partition table and exit fdisk
mkfs.ext4 /dev/sda1
mkdir /mnt/btcpay-data
UUID="$(sudo blkid -s UUID -o value /dev/sda1)"
echo "UUID=$UUID /mnt/btcpay-data ext4 defaults,noatime,nofail 0 0" | sudo t7e -a /etc/fstab
mount -a
```
## Step 6: Create mount for Docker volumes
rm -rf /var/lib/docker
mkdir -p /var/lib/docker
mount --bind /mnt/btcpay-data/volumes /var/lib/docker/volumes
echo "/mnt/docker /var/lib/docker none bind,nobootwait 0 2" >> /etc/fstab
systemctl restart docker

## Step 6: Clone BTCPay Server Repository

Clone the BTCPay Server Docker repository:

```bash
cd ~
git clone https://github.com/btcpayserver/btcpayserver-docker
cd btcpayserver-docker
```

## Step 7: Configure BTCPay Server

Set environment variables to include both Bitcoin and Monero:

```bash
export BTCPAY_HOST="btcpay.local" # Replace with your domain or local hostname
export REVERSEPROXY_DEFAULT_HOST="$BTCPAY_HOST" 
export NBITCOIN_NETWORK="mainnet"
export BTCPAYGEN_CRYPTO1="btc" 
export BTCPAYGEN_LIGHTNING="clightning" # or "lnd" for LND
export BTCPAYGEN_REVERSEPROXY="nginx" # Use nginx for reverse proxy
export BTCPAY_ENABLE_SSH=true # Enable SSH for remote access
```

If you want to prune nodes to save space, add the following line:
```bash
export BTCPAYGEN_ADDITIONAL_FRAGMENTS="opt-save-storage-xs;"
```

If you have additional domains:

```bash
export BTCPAY_ADDITIONAL_HOSTS="btcpay.yourdomain.com"
```

Find all the available environment variables and their descriptions in the [BTCPay Server documentation](https://github.com/btcpayserver/btcpayserver-docker#environment-variables).

## Step 8: Run BTCPay Server Setup

Initiate the setup process:

```bash
. ./btcpay-setup.sh -i
```

This script installs the necessary Docker images and configures BTCPay Server with Bitcoin and Monero support.

## Step 9: Access BTCPay Server

Once setup is complete, access BTCPay Server via your browser:

```
http://btcpay.local
```

Ensure your DNS settings or `/etc/hosts` file are configured if the domain doesn't resolve automatically.

## Step 10: Install Monero Plugin
To enable Monero support, you need to install the Monero plugin via the Plugins tab in BTCPay Server:
1. Log in to your BTCPay Server.
2. Navigate to the "Manage Plugins" section.
3. Search for "Monero" and install the Monero plugin.
4. Restart the server to apply the plugin changes.

## Step 10: Setup View-Only Monero Wallet
To set up a view-only Monero wallet, you will need to create a view-only wallet from your existing Monero wallet. This can be done using the Monero CLI or GUI.You can also use the Monero GUI and Feather Wallet to create a view-only wallet by importing the view key.

## Step 10: Configure Monero Wallet in BTCPay Server
After installing the Monero plugin, configure your Monero wallet:
1. Go to the "Wallets" section in BTCPay Server.
2. Click on XMR Wallet and upload your wallet file, wallet.keys file and password.

## Step 11: Monitor and Maintain
After installation, monitor the logs and ensure everything is running smoothly:

```bash
docker logs  --follow generated_btcpayserver_1
docker logs  --follow btcpayserver_bitcoind
docker logs  --follow btcpayserver_monero_wallet
docker logs --follow btcpayserver_monerod
```

## Additional Notes

- Initial blockchain sync may take time depending on your hardware and internet speed.
- Regularly update BTCPay Server for security and features.
- Monitor your external disk for available space.
