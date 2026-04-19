# IPAM Tool

A self-hosted IP Address Management (IPAM) tool built for homelab and small business networks. Integrates natively with **Pi-hole v6** for DNS record management and **Ubiquiti Dream Machine SE** for DHCP lease discovery. Deployed as a lightweight Ubuntu service behind Nginx.

---

## Features

| Feature | Details |
|---|---|
| **Subnet Management** | Add multiple CIDR blocks with VLAN, gateway, and DNS metadata |
| **IP Map** | Visual grid of every host address — click any IP for details or to reserve it |
| **Pi-hole v6 Sync** | Pulls custom DNS A records via Pi-hole REST API; can push new records back |
| **UniFi UDM-SE Sync** | Pulls DHCP leases and active clients from Dream Machine SE Integration API |
| **IP Reservation** | Manual reservations with hostname, MAC, and description; pushes DNS to Pi-hole |
| **Conflict Detection** | Flags IPs assigned more than once across sources |
| **Email Alerts** | SMTP alerting for IP conflicts and subnet utilisation thresholds |
| **REST API** | Full CRUD API for automation and integration with other tools |
| **Dark UI** | Industrial-style React frontend — no external cloud dependencies |

---

## Requirements

### Server (Ubuntu)
| Component | Minimum | Recommended |
|---|---|---|
| OS | Ubuntu 22.04 LTS | Ubuntu 24.04 LTS |
| CPU | 1 vCPU | 2 vCPU |
| RAM | 512 MB | 1–2 GB |
| Disk | 4 GB | 10 GB (SSD) |
| Network | Reachable from browser and from Pi-hole/UDM-SE | Static IP or DHCP reservation |

### Software (auto-installed by `install.sh`)
- Python 3.10+ with `pip` and `venv`
- Node.js 18+ and npm 8+
- Nginx
- SQLite 3

### Network Integrations

**Pi-hole v6.x**
- Pi-hole must be v6.0 or newer (v5.x uses a different API)
- The IPAM server must be able to reach Pi-hole on port 80 (HTTP) or 443 (HTTPS)
- You need a Pi-hole **App Password**: Admin UI → Settings → API → App Password

**Ubiquiti Dream Machine SE**
- UniFi Network Application **9.1.105 or newer** (required for the Integration API)
- IPAM server must reach the UDM-SE on port **443**
- API key from: UniFi Network → Settings → Control Plane → **Integrations** → Create API Key
  > ⚠️ **This is not the same as** UniFi OS → Settings → Control Plane → API Keys — that key is for a different API and will not work here.

**SMTP (optional)**
- Any SMTP server: Gmail, Office 365, self-hosted Postfix, etc.
- Supports STARTTLS (port 587) and SSL/TLS (port 465)
- For Gmail, use an [App Password](https://support.google.com/accounts/answer/185833) with 2FA enabled

---

## Installation

### Option A — Quick Install (recommended)

```bash
# 1. Clone the repository
git clone https://github.com/your-username/ipam-tool.git
cd ipam-tool

# 2. Run the installer as root
sudo bash deploy/install.sh
```

The installer will:
- Install all system dependencies
- Create a dedicated `ipam` system user
- Set up a Python virtualenv and install Python packages
- Build the React frontend
- Create and initialise the SQLite database
- Configure a systemd service (`ipam.service`)
- Configure Nginx as a reverse proxy on port 80
- Apply UFW firewall rules (SSH + HTTP)

Installation takes approximately **3–5 minutes** on a fresh Ubuntu server.

### Option B — Custom Install Paths

You can override defaults with environment variables:

```bash
IPAM_INSTALL_DIR=/srv/ipam \
IPAM_DATA_DIR=/var/lib/ipam \
IPAM_PORT=5000 \
sudo -E bash deploy/install.sh
```

### Option C — VMware vSphere OVA

Build an importable OVA for vSphere using the provided build script:

```bash
# Requires: qemu-utils, cloud-image-utils, VMware ovftool
sudo apt-get install -y qemu-utils cloud-image-utils
chmod +x deploy/build-ova.sh
./deploy/build-ova.sh
# Produces: deploy/ipam-tool.ova
```

Import into vSphere: **Hosts & Clusters → right-click → Deploy OVF Template → select ipam-tool.ova**

---

## First-Time Setup

1. Open `http://<server-ip>/` in your browser
2. **Settings → Pi-hole**: Enter your Pi-hole URL and App Password → Test → Save
3. **Settings → Ubiquiti Dream Machine SE**: Enter UDM-SE URL and Integration API key → Test → Save
4. **Subnets**: Add your network ranges (e.g. `192.168.1.0/24`)
5. **Settings → Sync All**: Import existing DNS records and DHCP leases
6. **Settings → Email Alerts** *(optional)*: Configure SMTP for conflict and utilisation alerts

---

## Updating

```bash
# Pull latest code
cd /path/to/ipam-tool
git pull

# Update backend
sudo cp backend/app.py /opt/ipam/app/app.py
sudo /opt/ipam/venv/bin/pip install -q -r backend/requirements.txt
sudo systemctl restart ipam

# Rebuild frontend
cd frontend && npm install && npm run build
sudo rm -rf /opt/ipam/app/frontend/dist
sudo cp -r dist /opt/ipam/app/frontend/dist

# Migrate database (adds any new tables/columns without data loss)
sudo -u ipam /opt/ipam/venv/bin/python3 -c "
import sys; sys.path.insert(0, '/opt/ipam/app')
from app import app, db
with app.app_context(): db.create_all(); print('DB OK')
"
sudo systemctl restart ipam
```

---

## REST API

Base URL: `http://<server-ip>/api`

All requests and responses use JSON. No authentication is required by default (deploy behind a VPN or add auth middleware for production).

### Subnets
| Method | Endpoint | Description |
|---|---|---|
| GET | `/api/subnets` | List all subnets with utilisation stats |
| POST | `/api/subnets` | Create subnet `{name, cidr, gateway?, vlan_id?, description?}` |
| PUT | `/api/subnets/:id` | Update subnet metadata |
| DELETE | `/api/subnets/:id` | Delete subnet and all its reservations |
| GET | `/api/subnets/:id/ips` | All IPs in subnet (allocated + free) |
| GET | `/api/subnets/:id/next-free` | Next available IP address |

### Reservations
| Method | Endpoint | Description |
|---|---|---|
| GET | `/api/reservations` | List reservations (`?subnet_id=&source=&status=`) |
| POST | `/api/reservations` | Reserve IP `{subnet_id, ip_address, hostname?, mac_address?, push_to_pihole?}` |
| PUT | `/api/reservations/:id` | Update reservation |
| DELETE | `/api/reservations/:id` | Release IP (`?remove_from_pihole=true`) |

### Sync
| Method | Endpoint | Description |
|---|---|---|
| POST | `/api/pihole/sync` | Trigger Pi-hole sync |
| POST | `/api/unifi/sync` | Trigger UniFi sync |
| POST | `/api/sync/all` | Trigger both |
| POST | `/api/pihole/test` | Test Pi-hole connection `{base_url, app_password}` |
| POST | `/api/unifi/test` | Test UniFi connection `{base_url, api_key, site}` |

### Alerts
| Method | Endpoint | Description |
|---|---|---|
| GET | `/api/alerts/config` | Get alert configuration |
| POST | `/api/alerts/config` | Save alert configuration |
| POST | `/api/alerts/test` | Send a test email |
| POST | `/api/alerts/trigger` | Run alert checks immediately |

### Other
| Method | Endpoint | Description |
|---|---|---|
| GET | `/api/stats` | Dashboard statistics |
| GET | `/api/conflicts` | Unresolved IP conflicts |
| POST | `/api/conflicts/:id/resolve` | Mark conflict resolved |
| GET | `/api/health` | Service health check |

### Example: Reserve an IP via API
```bash
curl -X POST http://192.168.1.50/api/reservations \
  -H "Content-Type: application/json" \
  -d '{
    "subnet_id": 1,
    "ip_address": "192.168.1.100",
    "hostname": "myserver.local",
    "mac_address": "aa:bb:cc:dd:ee:ff",
    "push_to_pihole": true
  }'
```

---

## Service Management

```bash
# Status
sudo systemctl status ipam

# Restart
sudo systemctl restart ipam

# Live logs
sudo journalctl -u ipam -f

# Access log
sudo tail -f /var/log/ipam-access.log

# Error log
sudo tail -f /var/log/ipam-error.log

# Database location
ls -lh /opt/ipam/data/ipam.db

# Inspect database tables
sqlite3 /opt/ipam/data/ipam.db ".tables"
sqlite3 /opt/ipam/data/ipam.db "SELECT COUNT(*) FROM ip_reservations;"
```

---

## File Structure

```
ipam-tool/
├── README.md
├── backend/
│   ├── app.py               # Flask application (API + Pi-hole + UniFi + SMTP)
│   └── requirements.txt     # Python dependencies
├── frontend/
│   ├── src/
│   │   ├── App.jsx           # Root component + navigation
│   │   ├── App.css           # Global styles (dark industrial theme)
│   │   ├── api.js            # Shared fetch wrapper with error handling
│   │   └── components/
│   │       ├── Dashboard.jsx  # Overview stats + quick actions
│   │       ├── Subnets.jsx    # Subnet list + IP Map overlay
│   │       ├── Reservations.jsx
│   │       ├── Conflicts.jsx
│   │       └── Settings.jsx   # Pi-hole, UniFi, SMTP alert config
│   ├── index.html
│   ├── package.json
│   └── vite.config.js
└── deploy/
    ├── install.sh            # Universal Ubuntu installer
    ├── build-ova.sh          # VMware OVA builder
    └── ipam-tool.ovf         # OVF descriptor (2 vCPU, 2 GB RAM, 20 GB disk)
```

---

## Architecture

```
Browser
  │  HTTP :80
  ▼
Nginx ──────────── /api/* ──────► Gunicorn :5000 ──► Flask app
  │                                                      │
  └── /* ──► /opt/ipam/app/frontend/dist/ (React SPA)   │
                                                         ├──► SQLite DB
                                                         │    /opt/ipam/data/ipam.db
                                                         │
                                              ┌──────────┴──────────┐
                                              │   Background sync   │
                                              │   (configurable     │
                                              │    interval)        │
                                              └──────┬──────────────┘
                                                     │
                                         ┌───────────┴───────────┐
                                         │                       │
                                  Pi-hole v6                UDM-SE
                                  /api/customdns    /proxy/network/
                                  /api/auth         integration/v1/
```

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| "Network error" on any form | Run `curl http://localhost:5000/api/health` — if it fails, check `journalctl -u ipam -n 30` |
| Pi-hole test fails | Verify URL includes `http://` and the VM can ping the Pi-hole IP |
| UniFi "Invalid API key" | Ensure you're using the key from UniFi **Network** → Settings → Control Plane → **Integrations** (not UniFi OS) |
| UniFi sync shows 0 IPs | Check `journalctl -u ipam -f` during a sync — verify your subnet CIDRs match the IP range from the UDM |
| Emails not sending | Check the SMTP credentials, try port 587 with STARTTLS first; Gmail requires an App Password |
| 502 Bad Gateway | Flask isn't running — `sudo systemctl restart ipam` |
| Frontend shows "not built" | Run: `cd /opt/ipam/frontend && npm run build && sudo cp -r dist /opt/ipam/app/frontend/dist` |

---

## Security Recommendations

- **Network access**: Place the IPAM server on a management VLAN or behind a VPN — the API has no authentication by default
- **HTTPS**: Add a TLS certificate via [Certbot](https://certbot.eff.org/) or a self-signed cert in Nginx for encrypted access
- **API key storage**: Pi-hole and UniFi credentials are stored in SQLite — use an encrypted datastore or full-disk encryption for production
- **Backups**: Back up `/opt/ipam/data/ipam.db` regularly — this is the only persistent state
- **Updates**: Keep Ubuntu, Nginx, and Python packages updated; run `sudo apt upgrade` regularly

```bash
# Quick backup
sudo cp /opt/ipam/data/ipam.db /opt/ipam/data/ipam.db.bak.$(date +%Y%m%d)

# Restore
sudo systemctl stop ipam
sudo cp /opt/ipam/data/ipam.db.bak.YYYYMMDD /opt/ipam/data/ipam.db
sudo systemctl start ipam
```

---

## Contributing

Pull requests are welcome. For major changes, please open an issue first.

```bash
# Local development
# Backend (requires Python 3.10+)
cd backend
python3 -m venv venv && source venv/bin/activate
pip install -r requirements.txt
DATABASE_URL=sqlite:///./dev.db python3 app.py

# Frontend (in a separate terminal)
cd frontend
npm install
npm run dev   # Vite dev server with API proxy to localhost:5000
```

---

## License

MIT — see [LICENSE](LICENSE) for details.
