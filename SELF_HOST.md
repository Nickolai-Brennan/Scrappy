# Self-hosting Firecrawl

#### Contributor?

Welcome to [Firecrawl](https://firecrawl.dev) 🔥! Here are some instructions on how to get the project locally so you can run it on your own and contribute.

If you're contributing, note that the process is similar to other open-source repos, i.e., fork Firecrawl, make changes, run tests, PR.

If you have any questions or would like help getting on board, join our Discord community [here](https://discord.gg/gSmWdAkdwd) for more information or submit an issue on Github [here](https://github.com/firecrawl/firecrawl/issues/new/choose)!

## Why?

Self-hosting Firecrawl is particularly beneficial for organizations with stringent security policies that require data to remain within controlled environments. Here are some key reasons to consider self-hosting:

- **Enhanced Security and Compliance:** By self-hosting, you ensure that all data handling and processing complies with internal and external regulations, keeping sensitive information within your secure infrastructure. Note that Firecrawl is a Mendable product and relies on SOC2 Type2 certification, which means that the platform adheres to high industry standards for managing data security.
- **Customizable Services:** Self-hosting allows you to tailor the services, such as the Playwright service, to meet specific needs or handle particular use cases that may not be supported by the standard cloud offering.
- **Learning and Community Contribution:** By setting up and maintaining your own instance, you gain a deeper understanding of how Firecrawl works, which can also lead to more meaningful contributions to the project.

### Considerations

However, there are some limitations and additional responsibilities to be aware of:

1. **Limited Access to Fire-engine:** Currently, self-hosted instances of Firecrawl do not have access to Fire-engine, which includes advanced features for handling IP blocks, robot detection mechanisms, and more. This means that while you can manage basic scraping tasks, more complex scenarios might require additional configuration or might not be supported.
2. **Manual Configuration Required:** If you need to use scraping methods beyond the basic fetch and Playwright options, you will need to manually configure these in the `.env` file. This requires a deeper understanding of the technologies and might involve more setup time.

Self-hosting Firecrawl is ideal for those who need full control over their scraping and data processing environments but comes with the trade-off of additional maintenance and configuration efforts.

## Deploying on an Ubuntu VPS

The steps below walk you through a production-ready deployment on a fresh Ubuntu 22.04 / 24.04 server.

### 1. Install Docker and Docker Compose

```bash
# Update package index and install prerequisites
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg lsb-release

# Add Docker's official GPG key and repository
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Allow your user to run Docker without sudo (log out and back in to apply)
sudo usermod -aG docker $USER
```

Verify the installation:

```bash
docker --version
docker compose version
```

### 2. Clone the repository

```bash
git clone https://github.com/Nickolai-Brennan/Scrappy.git
cd Scrappy
```

### 3. Configure environment variables

Copy the provided template and edit the values:

```bash
cp .env.example .env
nano .env   # or your preferred editor
```

**Minimum required changes:**

| Variable | What to set |
|----------|-------------|
| `BULL_AUTH_KEY` | A strong random secret (e.g. `openssl rand -hex 32`) |
| `POSTGRES_PASSWORD` | A strong password for the database |
| `OPENAI_API_KEY` | Your key — only needed for AI/extract features |

Everything else has sensible defaults for a self-hosted deployment.

> **Do not commit `.env` to version control.** The `.gitignore` already excludes it.

### 4. Build and start the stack

```bash
docker compose build
docker compose up -d
```

The API will be available on `http://YOUR_SERVER_IP:3002` once all services are healthy (allow ~60 seconds on first boot).

Check health:

```bash
curl http://localhost:3002/e2e-test   # should return HTTP 200 with body "OK"
```

You can also view the Bull Queue Manager UI at:
`http://localhost:3002/admin/<BULL_AUTH_KEY>/queues`

### 5. *(Optional)* Test the API

```bash
curl -X POST http://localhost:3002/v1/crawl \
    -H 'Content-Type: application/json' \
    -d '{"url": "https://firecrawl.dev"}'
```

---

## Operational commands

| Task | Command |
|------|---------|
| Start all services | `docker compose up -d` |
| Stop all services | `docker compose down` |
| View logs (all) | `docker compose logs -f` |
| View API logs only | `docker compose logs -f api` |
| Restart API service | `docker compose restart api` |
| Pull latest images & rebuild | `git pull && docker compose build && docker compose up -d` |
| Check service health | `docker compose ps` |
| Open a DB shell | `docker compose exec nuq-postgres psql -U postgres` |

---

## Security considerations

- **Use strong PostgreSQL credentials.** The defaults in the `.env` template are for local development only. When deploying to a server, set `POSTGRES_USER`, `POSTGRES_PASSWORD`, and `POSTGRES_DB` to secure values and ensure they match the database service configuration.
- **Keep the database port internal.** The provided `docker-compose.yaml` does not expose PostgreSQL to the host or the internet. Avoid adding a `ports` mapping for `nuq-postgres` unless you are restricting access with a firewall. To access the database for maintenance, prefer using `docker compose exec nuq-postgres psql` or a temporary, firewalled tunnel.
- **Protect the admin UI.** Set `BULL_AUTH_KEY` to a strong secret, especially on any deployment reachable from untrusted networks.
- **Restrict port 3002 with a firewall** if you are fronting the app with Nginx/Caddy. Example (UFW):
  ```bash
  sudo ufw allow OpenSSH
  sudo ufw allow 'Nginx Full'   # ports 80 and 443
  sudo ufw enable
  # Do NOT open port 3002 publicly when using a reverse proxy
  ```

---

## Reverse proxy with Nginx and HTTPS (optional)

A sample Nginx configuration is provided in [`deploy/nginx.conf`](deploy/nginx.conf).

### Install Nginx and Certbot

```bash
sudo apt-get install -y nginx certbot python3-certbot-nginx
```

### Configure Nginx

```bash
# Copy and edit the sample config
sudo cp deploy/nginx.conf /etc/nginx/sites-available/scrappy
sudo nano /etc/nginx/sites-available/scrappy
# Replace "your-domain.example.com" with your actual domain

sudo ln -s /etc/nginx/sites-available/scrappy /etc/nginx/sites-enabled/
sudo nginx -t          # verify syntax
sudo systemctl reload nginx
```

### Obtain a free TLS certificate (Let's Encrypt)

```bash
sudo certbot --nginx -d your-domain.example.com
```

Certbot will automatically update your Nginx config to handle HTTPS and renewal. After this step, the API is accessible at `https://your-domain.example.com`.

---

## Environment variables

The full list of supported variables with descriptions is in [`apps/api/.env.example`](apps/api/.env.example). The root [`.env.example`](.env.example) contains the minimal subset needed for a typical self-hosted deployment.

Minimal `.env` for a self-hosted deployment:

```
# ===== Required ENVS ======
PORT=3002
HOST=0.0.0.0
USE_DB_AUTHENTICATION=false
BULL_AUTH_KEY=CHANGEME

# ===== Optional ENVS ======
# OPENAI_API_KEY=
# POSTGRES_USER=firecrawl
# POSTGRES_PASSWORD=change_me_in_production
# POSTGRES_DB=firecrawl
```


## Troubleshooting

This section provides solutions to common issues you might encounter while setting up or running your self-hosted instance of Firecrawl.

### API Keys for SDK Usage

**Note:** When using Firecrawl SDKs with a self-hosted instance, API keys are optional. API keys are only required when connecting to the cloud service (api.firecrawl.dev).

### Supabase client is not configured

**Symptom:**
```bash
[YYYY-MM-DDTHH:MM:SS.SSSz]ERROR - Attempted to access Supabase client when it's not configured.
[YYYY-MM-DDTHH:MM:SS.SSSz]ERROR - Error inserting scrape event: Error: Supabase client is not configured.
```

**Explanation:**
This error occurs because the Supabase client setup is not completed. You should be able to scrape and crawl with no problems. Right now it's not possible to configure Supabase in self-hosted instances.

### You're bypassing authentication

**Symptom:**
```bash
[YYYY-MM-DDTHH:MM:SS.SSSz]WARN - You're bypassing authentication
```

**Explanation:**
This error occurs because the Supabase client setup is not completed. You should be able to scrape and crawl with no problems. Right now it's not possible to configure Supabase in self-hosted instances.

### Docker containers fail to start

**Symptom:**
Docker containers exit unexpectedly or fail to start.

**Solution:**
Check the Docker logs for any error messages using the command:
```bash
docker logs [container_name]
```

- Ensure all required environment variables are set correctly in the .env file.
- Verify that all Docker services defined in docker-compose.yml are correctly configured and the necessary images are available.

### Connection issues with Redis

**Symptom:**
Errors related to connecting to Redis, such as timeouts or "Connection refused".

**Solution:**
- Ensure that the Redis service is up and running in your Docker environment.
- Verify that the REDIS_URL and REDIS_RATE_LIMIT_URL in your .env file point to the correct Redis instance, ensure that it points to the same URL in the `docker-compose.yaml` file (`redis://redis:6379`)
- Check network settings and firewall rules that may block the connection to the Redis port.

### API endpoint does not respond

**Symptom:**
API requests to the Firecrawl instance timeout or return no response.

**Solution:**
- Ensure that the Firecrawl service is running by checking the Docker container status.
- Verify that the PORT and HOST settings in your .env file are correct and that no other service is using the same port.
- Check the network configuration to ensure that the host is accessible from the client making the API request.

By addressing these common issues, you can ensure a smoother setup and operation of your self-hosted Firecrawl instance.

## Install Firecrawl on a Kubernetes Cluster (Simple Version)

Read the [examples/kubernetes/cluster-install/README.md](https://github.com/firecrawl/firecrawl/blob/main/examples/kubernetes/cluster-install/README.md) for instructions on how to install Firecrawl on a Kubernetes Cluster.

## Install Firecrawl on a Kubernetes Cluster with Helm

Read the [examples/kubernetes/firecrawl-helm/README.md](https://github.com/firecrawl/firecrawl/blob/main/examples/kubernetes/firecrawl-helm/README.md) for instructions on how to install Firecrawl on a Kubernetes Cluster with Helm.
