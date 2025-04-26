# Locally Hosting n8n.io with Cloudflare Tunnel

This guide explains how to set up n8n locally using Docker and expose it securely to the internet via Cloudflare Tunnel.

## Installing and Running n8n Using Docker

### 1. Install Docker

Ensure Docker is installed on your system. If not, download and install it from [Docker's official website](https://www.docker.com/products/docker-desktop/).

### 2. Run n8n Docker Container

Use the following command to run n8n in a Docker container:

```bash
docker run --hostname=a219365dcb3b \
  --user=node \
  --env=PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin \
  --env=NODE_VERSION=20.19.0 \
  --env=YARN_VERSION=1.22.22 \
  --env=NODE_ICU_DATA=/usr/local/lib/node_modules/full-icu \
  --env=N8N_VERSION=1.86.1 \
  --env=NODE_ENV=production \
  --env=N8N_RELEASE_TYPE=stable \
  --env=SHELL=/bin/sh \
  --env=WEBHOOK_URL=https://subdomain.example.com \
  --network=bridge \
  --workdir=/home/node \
  -p 5678:5678 \
  --restart=no \
  --label='org.opencontainers.image.description=Workflow Automation Tool' \
  --label='org.opencontainers.image.source=https://github.com/n8n-io/n8n' \
  --label='org.opencontainers.image.title=n8n' \
  --label='org.opencontainers.image.url=https://n8n.io' \
  --label='org.opencontainers.image.version=1.86.1' \
  --runtime=runc \
  -d n8nio/n8n:latest
```

**Important**: Replace `https://subdomain.example.com` with the domain you've configured in your Cloudflare Tunnel. This setting is crucial for n8n to properly handle webhook callbacks and API connections.

The container exposes n8n on port 5678, which we'll route through the Cloudflare Tunnel.

For additional configuration options, refer to the [n8n Docker configuration documentation](https://docs.n8n.io/hosting/configuration/configuration-methods/#docker).

## Installing and Configuring Cloudflare Tunnel

### 1. Download and Install `cloudflared`

Download the appropriate `cloudflared` CLI tool for your operating system:

- [Windows](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/install-and-setup/installation/windows)
- [macOS](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/install-and-setup/installation/macos)
- [Linux](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/install-and-setup/installation/linux)

Follow the installation instructions specific to your OS.

### 2. Authenticate with Cloudflare

Open a terminal and run:

```bash
cloudflared tunnel login
```

This will open a browser window prompting you to log in to your Cloudflare account and authorize the tunnel connection.

### 3. Create a Named Tunnel

To create a persistent tunnel:

```bash
cloudflared tunnel create <TUNNEL_NAME>
```

Replace `<TUNNEL_NAME>` with a descriptive name for your tunnel.

### 4. Configure the Tunnel

Create a configuration file (e.g., `config.yml`) in your `.cloudflared` directory:

```yaml
tunnel: <TUNNEL_NAME>
credentials-file: <PATH_TO_CREDENTIALS_FILE>
ingress:
  - hostname: <YOUR_DOMAIN>
    service: http://localhost:5678
  - service: http_status:404
```

Notes:
- Replace `<TUNNEL_NAME>` with the name from step 3
- Replace `<PATH_TO_CREDENTIALS_FILE>` with the path to your credentials JSON file (typically in `~/.cloudflared/`)
- Replace `<YOUR_DOMAIN>` with your Cloudflare domain
- The default port for n8n is 5678

### 5. Start the Tunnel

Run the following command to start the tunnel:

```bash
cloudflared tunnel run <TUNNEL_NAME>
```

### Alternative: Using a Quick Tunnel

For temporary access (testing purposes):

1. Start your n8n Docker container
2. Run:
   ```bash
   cloudflared tunnel --url http://localhost:5678
   ```
3. Cloudflare will generate a temporary URL you can use to access your n8n instance

## Connecting Locally Installed Ollama with n8n Docker Container

### Understanding the Architecture

This setup consists of:
1. Ollama running natively on your host machine (not containerized)
2. n8n running inside a Docker container on the same host machine

This approach is particularly beneficial for Mac users, especially those with Apple Silicon, as running Ollama directly on macOS provides better performance and GPU utilization than running it inside Docker.

### Key Networking Concept: Connecting from Docker to Host

When a service inside a Docker container needs to connect to a service on the host machine, you cannot use `localhost` as it refers to the container itself. Instead, Docker provides a special DNS name:

```
host.docker.internal
```

This hostname resolves to the host machine's IP address from within Docker containers.

### Step-by-Step Configuration

1. **Ensure Ollama is running on your host machine**
   * Install Ollama from [the official website](https://ollama.ai/download)
   * Start the Ollama service
   * Verify it's accessible by running: `curl http://localhost:11434/models` in your terminal

2. **Configure Ollama connection in n8n**
   * Access your n8n instance through your Cloudflare tunnel domain
   * Navigate to Settings > Credentials
   * Create a new Ollama credential with:
     - Host: `host.docker.internal`
     - Port: `11434`
   * Save the credentials

3. **Test the connection**
   * Create a new workflow in n8n
   * Add an Ollama node
   * Configure it to use your newly created credentials
   * Run a simple test with a model that's installed on your Ollama instance

### Troubleshooting Connection Issues

If you encounter problems with the connection:

1. **Verify Ollama is running**:
   ```bash
   curl http://localhost:11434/models
   ```
   This should return a list of available models

2. **Check installed models**:
   ```bash
   ollama list
   ```
   Make sure you have models installed that you're trying to use in n8n

3. **Firewall settings**: Ensure your firewall isn't blocking connections to port 11434

4. **Alternative connection method**: If `host.docker.internal` doesn't work, try using your host's actual IP address (find it using `ifconfig` on macOS or `ipconfig` on Windows)
