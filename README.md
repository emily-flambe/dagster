# Dagster PC Setup

Multi-project Dagster instance hosted on home PC (Windows + WSL2), accessible via Tailscale.

## Architecture

```
WSL2 Ubuntu
├── PostgreSQL (native)
├── dagster-webserver (systemd service)
├── dagster-daemon (systemd service)
└── Code locations mounted from Windows filesystem

Windows
├── Port proxy (forwards 0.0.0.0:3000 → WSL2:3000)
└── Firewall rule (allows port 3000)

Tailscale
└── Access UI at http://pceus:3000
```

## Directory Structure

```
Windows:
C:\Users\emily\Documents\GitHub\
├── dagster/                    # This repo - infrastructure docs
├── project-a/                  # Separate repo per project
│   └── dagster_definitions/
│       ├── __init__.py
│       └── definitions.py
└── project-b/
    └── dagster_definitions/

WSL2 (/opt/dagster/):
├── venv/                       # Python virtual environment
└── dagster_home/
    ├── dagster.yaml            # Instance config (postgres connection)
    └── workspace.yaml          # Code locations
```

## Setup

### 1. Configure WSL2

Create `C:\Users\emily\.wslconfig`:

```ini
[wsl2]
memory=8GB
swap=4GB
processors=4
vmIdleTimeout=-1

[experimental]
autoMemoryReclaim=disabled

[general]
instanceIdleTimeout=-1
```

**Important:** The timeout settings prevent WSL2 from auto-shutting down background services.

Then restart WSL: `wsl --shutdown`

### 2. Install PostgreSQL in WSL2

```bash
wsl -d Ubuntu
sudo apt-get update && sudo apt-get install -y postgresql postgresql-contrib
sudo service postgresql start
sudo -u postgres psql -c "CREATE USER dagster WITH PASSWORD 'dagster';"
sudo -u postgres psql -c "CREATE DATABASE dagster OWNER dagster;"
```

### 3. Install Dagster in WSL2

```bash
sudo mkdir -p /opt/dagster && sudo chown $(id -u):$(id -g) /opt/dagster
python3 -m venv /opt/dagster/venv
source /opt/dagster/venv/bin/activate
pip install dagster dagster-webserver dagster-postgres
mkdir -p /opt/dagster/dagster_home
```

### 4. Create Dagster Config

Create `/opt/dagster/dagster_home/dagster.yaml`:

```yaml
storage:
  postgres:
    postgres_db:
      username: dagster
      password: dagster
      hostname: localhost
      db_name: dagster
      port: 5432

run_launcher:
  module: dagster.core.launcher
  class: DefaultRunLauncher

run_coordinator:
  module: dagster.core.run_coordinator
  class: QueuedRunCoordinator
```

Create `/opt/dagster/dagster_home/workspace.yaml`:

```yaml
load_from:
  - python_file:
      relative_path: /mnt/c/Users/emily/Documents/GitHub/project-a/dagster_definitions/definitions.py
      location_name: project_a
```

### 5. Create systemd Services

Create `/etc/systemd/system/dagster-webserver.service`:

```ini
[Unit]
Description=Dagster Webserver
After=network.target postgresql.service

[Service]
Type=simple
User=root
Environment=DAGSTER_HOME=/opt/dagster/dagster_home
ExecStart=/opt/dagster/venv/bin/dagster-webserver -h 0.0.0.0 -p 3000 -w /opt/dagster/dagster_home/workspace.yaml
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Create `/etc/systemd/system/dagster-daemon.service`:

```ini
[Unit]
Description=Dagster Daemon
After=network.target postgresql.service

[Service]
Type=simple
User=root
Environment=DAGSTER_HOME=/opt/dagster/dagster_home
ExecStart=/opt/dagster/venv/bin/dagster-daemon run -w /opt/dagster/dagster_home/workspace.yaml
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable dagster-webserver dagster-daemon
sudo systemctl start dagster-webserver dagster-daemon
```

### 6. Configure Windows Network Access

WSL2 only forwards ports to Windows localhost by default. To access via Tailscale, run these in PowerShell (Admin):

```powershell
# Get WSL2 IP (changes on reboot)
wsl -d Ubuntu hostname -I

# Add port proxy (replace IP with actual WSL2 IP)
netsh interface portproxy add v4tov4 listenport=3000 listenaddress=0.0.0.0 connectport=3000 connectaddress=<WSL2_IP>

# Add firewall rule
netsh advfirewall firewall add rule name="Dagster WSL2" dir=in action=allow protocol=tcp localport=3000
```

**Note:** WSL2 IP changes on reboot. You may need to update the port proxy after restarting.

### 7. Access UI

From any device on Tailscale: `http://pceus:3000`

## Adding a New Project

1. **Create Dagster definitions in the project:**

   ```bash
   mkdir -p ~/Documents/GitHub/new-project/dagster_definitions
   touch ~/Documents/GitHub/new-project/dagster_definitions/__init__.py
   ```

   Create `definitions.py`:

   ```python
   from dagster import asset, Definitions

   @asset
   def my_asset():
       return "Hello!"

   defs = Definitions(assets=[my_asset])
   ```

2. **Add to workspace.yaml** (`/opt/dagster/dagster_home/workspace.yaml`):

   ```yaml
   - python_file:
       relative_path: /mnt/c/Users/emily/Documents/GitHub/new-project/dagster_definitions/definitions.py
       location_name: new_project
   ```

3. **Restart services:**

   ```bash
   wsl -d Ubuntu -e sudo systemctl restart dagster-webserver dagster-daemon
   ```

## Starting Services After Reboot

```bash
wsl -d Ubuntu -e bash -c "sudo service postgresql start && sudo systemctl start dagster-webserver dagster-daemon"
```

You may also need to update the port proxy if WSL2 IP changed:

```powershell
# Remove old proxy
netsh interface portproxy delete v4tov4 listenport=3000 listenaddress=0.0.0.0

# Get new WSL2 IP and add proxy
wsl -d Ubuntu hostname -I
netsh interface portproxy add v4tov4 listenport=3000 listenaddress=0.0.0.0 connectport=3000 connectaddress=<NEW_IP>
```

## Troubleshooting

```bash
# Check service status
wsl -d Ubuntu -e sudo systemctl status dagster-webserver dagster-daemon

# View logs
wsl -d Ubuntu -e sudo journalctl -u dagster-webserver -f
wsl -d Ubuntu -e sudo journalctl -u dagster-daemon -f

# Restart services
wsl -d Ubuntu -e sudo systemctl restart dagster-webserver dagster-daemon

# Check PostgreSQL
wsl -d Ubuntu -e sudo service postgresql status

# Check port proxy (Windows)
netsh interface portproxy show all

# Test connectivity
curl http://localhost:3000/
```
