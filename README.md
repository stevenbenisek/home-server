# Home Server

A home server setup providing media automation, system monitoring, network management, and home automation services.

**Architecture:** All services run in Docker containers except home automation, which uses Home Assistant OS in a virtual machine for full add-on and Supervisor support.

## Quick Start

1. [Set up Home Assistant VM](#1-set-up-home-assistant-vm)
2. [Initialize Docker networks](#2-initialize-docker-networks)
3. [Configure Docker services](#3-configure-docker-services)
4. [Start Docker services](#4-start-docker-services)
5. [Complete post-installation setup](#5-post-installation)

## Prerequisites

- Docker and Docker Compose installed
- NordVPN account (for media automation with VPN)
- Basic understanding of Docker networking
- UTM or similar virtualization software (for Home Assistant VM on macOS)

## Installation

### 1. Set up Home Assistant VM

Home automation runs separately in a virtual machine to provide full Home Assistant OS functionality. Follow the [official Home Assistant macOS installation guide](https://www.home-assistant.io/installation/macos/) to:
   - Download the Home Assistant OS QEMU image
   - Create and configure the VM in UTM
   - Start the VM and complete initial setup

**Benefits of running Home Assistant OS in a VM:**
- Full add-on support
- Supervisor for managing system services
- Native hardware integration support
- Automatic updates and backups
- Optimized performance for home automation

### 2. Initialize Docker Networks

All Docker services require pre-configured Docker networks. Create them before starting any services:

```bash
# Internal networks (isolated from external traffic)
docker network create --internal media-automation
docker network create --internal media-playback
docker network create --internal system-monitoring

# External networks (handle external traffic routing)
docker network create media-automation-vpn
docker network create network-gateway
```

**Verify network creation:**

```bash
docker network ls
```

You should see all five networks listed.

### 3. Configure Docker Services

Each Docker service requires environment variables configured in a `.env` file.

#### Create Environment Files

Copy the example files to create your environment configurations:

```bash
# Network Gateway
cp services/network-gateway/.env.example services/network-gateway/.env

# Media Automation
cp services/media-automation/.env.example services/media-automation/.env

# Media Playback
cp services/media-playback/.env.example services/media-playback/.env

# System Monitoring
cp services/system-monitoring/.env.example services/system-monitoring/.env
```

#### Configure Common Variables

Edit each `.env` file with your specific values. Common variables include:

- `PUID` - User ID for file permissions (find with `id -u`)
- `PGID` - Group ID for file permissions (find with `id -g`)
- `TZ` - Timezone (e.g., `Europe/London`, `America/New_York`)

Additional service-specific variables are documented in each `.env.example` file.

#### Obtain WireGuard Private Key (Media Automation)

The media automation service requires a NordVPN WireGuard private key:

1. Navigate to the [NordVPN dashboard](https://my.nordaccount.com/dashboard/nordvpn/) and generate an access token
2. Retrieve your credentials using the token:
   ```bash
   curl -s -u token:<ACCESS_TOKEN> https://api.nordvpn.com/v1/users/services/credentials
   ```
   Replace `<ACCESS_TOKEN>` with your generated token.
3. Copy the `nordlynx_private_key` from the response
4. Set the `WIREGUARD_PRIVATE_KEY` variable in `services/media-automation/.env`

### 4. Start Docker Services

Start Docker services from the repository root using Docker Compose.

#### Start Individual Services

```bash
docker compose -f services/network-gateway/docker-compose.yml up -d
docker compose -f services/media-automation/docker-compose.yml up -d
docker compose -f services/media-playback/docker-compose.yml up -d
docker compose -f services/system-monitoring/docker-compose.yml up -d
```

#### Start All Services

```bash
docker compose \
  -f services/network-gateway/docker-compose.yml \
  -f services/media-automation/docker-compose.yml \
  -f services/media-playback/docker-compose.yml \
  -f services/system-monitoring/docker-compose.yml \
  up -d
```

### 5. Post-Installation

#### Configure Home Assistant

After the first startup, Home Assistant requires additional configuration:

1. Complete the initial setup wizard
2. Configure trusted proxies for reverse proxy support
3. Follow the [Home Assistant reverse proxy documentation](https://www.home-assistant.io/integrations/http/#reverse-proxies)

## Platform-Specific Configuration

### Docker Compose Profiles

Services can be enabled or disabled using Docker Compose profiles based on your platform and hardware:

- `gpu-passthrough` - Enables hardware transcoding in Jellyfin (Linux only)

### Linux

Linux supports GPU passthrough for hardware-accelerated transcoding. Enable the profile when starting services:

```bash
docker compose --profile gpu-passthrough -f services/media-playback/docker-compose.yml up -d
```

### macOS

Docker on macOS does not support GPU passthrough. Start services without profile flags:

```bash
docker compose -f services/media-playback/docker-compose.yml up -d
```

**Note:** For hardware-accelerated transcoding on macOS, run Jellyfin natively instead of through Docker.

## Managing Services

### View Service Logs

```bash
docker compose -f services/[service]/docker-compose.yml logs -f
```

### Stop Services

```bash
# Stop individual service
docker compose -f services/[service]/docker-compose.yml down

# Stop all services
docker compose \
  -f services/network-gateway/docker-compose.yml \
  -f services/media-automation/docker-compose.yml \
  -f services/media-playback/docker-compose.yml \
  -f services/system-monitoring/docker-compose.yml \
  down
```

### Update Services

```bash
# Pull latest images
docker compose -f services/[service]/docker-compose.yml pull

# Restart with new images
docker compose -f services/[service]/docker-compose.yml up -d
```
