# Dagster PC Setup

Multi-project Dagster instance hosted on home PC, accessible via Tailscale.

## Architecture

```
One Dagster instance → Multiple code locations (one per project)
                    → Shared postgres, daemon, UI
                    → Each project repo contains its own Dagster definitions
```

## Directory structure

```
~/
├── dagster/                    # This repo - Dagster infrastructure
│   ├── docker-compose.yml
│   ├── config/
│   │   ├── dagster.yaml
│   │   └── workspace.yaml
│   └── README.md
│
├── project-a/                  # Separate repo per project
│   ├── dagster_definitions/
│   │   ├── __init__.py
│   │   └── definitions.py
│   └── ...
│
└── project-b/
    ├── dagster_definitions/
    │   ├── __init__.py
    │   └── definitions.py
    └── ...
```

## Setup

### 1. Install Docker
```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
# Log out and back in
```

### 2. Start services
```bash
cd ~/dagster
docker compose up -d
```

### 3. Access UI
From MacBook: `http://<pc-tailscale-hostname>:3000`

## Adding a new project

1. **Create Dagster definitions in the project:**
   ```bash
   mkdir -p ~/new-project/dagster_definitions
   cat > ~/new-project/dagster_definitions/__init__.py << 'EOF'
   EOF
   cat > ~/new-project/dagster_definitions/definitions.py << 'EOF'
   from dagster import asset, Definitions

   @asset
   def my_asset():
       return "Hello!"

   defs = Definitions(assets=[my_asset])
   EOF
   ```

2. **Add mount to `docker-compose.yml`:**
   ```yaml
   volumes:
     # ... existing mounts ...
     - ~/new-project:/projects/new-project:ro
   ```
   (Add to both `dagster-webserver` and `dagster-daemon`)

3. **Add code location to `workspace.yaml`:**
   ```yaml
   - python_file:
       relative_path: /projects/new-project/dagster_definitions/definitions.py
       location_name: new_project
   ```

4. **Restart Dagster:**
   ```bash
   cd ~/dagster
   docker compose restart dagster-webserver dagster-daemon
   ```

## Dependency isolation (optional)

If projects need different Python packages, run each as a gRPC server.

**In project repo, add `Dockerfile`:**
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install dagster dagster-postgres grpcio -r requirements.txt
COPY dagster_definitions/ ./dagster_definitions/
CMD ["dagster", "api", "grpc", "-h", "0.0.0.0", "-p", "4000", "-f", "dagster_definitions/definitions.py"]
```

**Add to `docker-compose.yml`:**
```yaml
dagster-code-project-a:
  build: ~/project-a
  restart: unless-stopped
```

**Update `workspace.yaml`:**
```yaml
- grpc_server:
    host: dagster-code-project-a
    port: 4000
    location_name: project_a
```

Only do this if you hit actual dependency conflicts.

## Verification

- [ ] `docker compose ps` shows 3 services healthy
- [ ] UI accessible at `http://<tailscale-hostname>:3000`
- [ ] Each project appears as separate code location in UI
- [ ] Can materialize assets from each project

## Troubleshooting

```bash
# View logs
docker compose logs -f dagster-webserver
docker compose logs -f dagster-daemon

# Restart after config changes
docker compose restart dagster-webserver dagster-daemon

# Full rebuild
docker compose down && docker compose up -d

# Check workspace is valid
docker compose exec dagster-webserver dagster debug workspace
```
