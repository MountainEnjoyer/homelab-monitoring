# Monitoring stack

Metrics, logs and dashboards for the homelab, running in a 1 GB LXC container on Proxmox.

| Component | Role | Port |
|---|---|---|
| Prometheus | Collects metrics every 30 s, keeps 15 days | 9090 |
| Loki | Stores logs, keeps 7 days | 3100 |
| Grafana | Dashboards and alerts | 3000 |
| prometheus-pve-exporter | Reads VM/LXC metrics from the Proxmox API | internal |
| node_exporter *(on each machine)* | CPU, RAM, disk, network | 9100 |
| Grafana Alloy *(on each VM)* | Ships logs to Loki | — |

```
 Proxmox host ──node_exporter──┐
      │ API ── pve-exporter ───┤
 Web server ──node_exporter────┼──► Prometheus ──┐
 Coolify    ──node_exporter────┘                 ├──► Grafana
 Web server ──Alloy (logs)─────┐                 │
 Coolify    ──Alloy (logs)─────┴──► Loki ────────┘
```

---

## 1. Create the LXC container (Proxmox web UI)

**Create CT** with:
- **Template:** Debian 12 or 13 (download it under *local → CT Templates → Templates* first)
- **Unprivileged:** yes
- **Disk:** 16 GB · **CPU:** 2 cores · **Memory:** 1024 MiB · **Swap:** 512 MiB
- **Network:** DHCP (then add a DHCP reservation for it in your router)

Before starting it: **Options → Features → tick `nesting` and `keyctl`** (needed for Docker inside LXC).

## 2. Install Docker in the container

From the container's console:

```bash
apt update && apt install -y curl git
curl -fsSL https://get.docker.com | sh
docker run --rm hello-world
```

## 3. Install node_exporter on the Proxmox host and both VMs

On **each** of the three machines:

```bash
sudo apt install -y prometheus-node-exporter
curl -s localhost:9100/metrics | head -n 3     # should print metrics
```

## 4. Create a read-only Proxmox API token

In the **Proxmox host's** shell:

```bash
pveum user add prometheus@pve
pveum acl modify / --users prometheus@pve --roles PVEAuditor
pveum user token add prometheus@pve monitoring --privsep 0
```

Copy the `value` it prints — it's only shown once.

## 5. Configure and start the stack

In the container:

```bash
# copy this folder into the container (git clone your repo, or scp it)
cd homelab/monitoring

cp .env.example .env                              # then set a real Grafana password
cp pve-exporter/pve.yml.example pve-exporter/pve.yml   # then paste the token value

# replace the placeholder IPs with yours
sed -i 's/PROXMOX_IP/192.168.1.10/g; s/WEBSERVER_IP/192.168.1.20/g; s/COOLIFY_IP/192.168.1.30/g' prometheus/prometheus.yml

docker compose up -d
docker compose ps
```

Check that every target is **UP** at `http://<container-ip>:9090/targets`.

## 6. Dashboards

Open `http://<container-ip>:3000` and log in. Prometheus and Loki are already connected.

**Dashboards → New → Import**, then import by ID:
- **1860** — Node Exporter Full (one machine in detail)
- **10347** — Proxmox via Prometheus (all VMs/LXCs)

## 7. Logs: install Grafana Alloy on each VM

On the **web server** and **Coolify** VMs:

```bash
sudo mkdir -p /etc/apt/keyrings
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee /etc/apt/sources.list.d/grafana.list
sudo apt update && sudo apt install -y alloy

# let Alloy read the journal and /var/log
sudo usermod -aG systemd-journal,adm alloy
```

Copy `agents/config.alloy` to `/etc/alloy/config.alloy`, replace `MONITORING_IP` and `HOSTNAME`, then:

```bash
sudo systemctl enable --now alloy
sudo systemctl restart alloy
```

In Grafana: **Explore → Loki →** `{host="webserver"}`.

## 8. First alerts (Grafana → Alerting)

Start with three:
- **Target down:** `up == 0` for 2 minutes
- **Low memory:** `node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes < 0.10` for 5 minutes
- **Disk almost full:** `node_filesystem_avail_bytes{fstype!~"tmpfs|overlay"} / node_filesystem_size_bytes < 0.15`

Add a **contact point** (Telegram, Discord or email) so alerts reach your phone.

---

## Security notes

- Ports 9100 (node_exporter) and 3100 (Loki) should only be reachable from the monitoring container. Restrict them with the Proxmox firewall or `ufw` on each VM.
- `.env` and `pve-exporter/pve.yml` contain secrets and are git-ignored.
- The Proxmox token only has the read-only `PVEAuditor` role.

## Next steps

- Pin image versions in `docker-compose.yml`
- Put Grafana behind a reverse proxy with HTTPS
- Deploy all of this with Ansible instead of by hand
