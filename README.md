# metrics-infra

Stack monitoring berbasis Prometheus untuk server metrics. Terdiri dari **Prometheus**, **Node Exporter**, dan **cAdvisor**.

## Stack

| Service | Fungsi | Default Port |
|---|---|---|
| Prometheus | Time-series database & scrape engine | `9090` |
| Node Exporter | Metrics OS & hardware (CPU, RAM, disk, network) | `9100` |
| cAdvisor | Metrics container Docker | `8080` |

---

## Prerequisites

- Docker & Docker Compose v2 sudah terinstall
- Akses `sudo` di server target
- Server sudah tergabung dalam network eksternal berikut (jika dipakai):
  - `pgsql_database_network`
  - `mysql_database_network`
  - `mariadb_database_network`

---

## Setup

### 1. Clone repo

```bash
git clone <repo-url> metrics-infra
cd metrics-infra
```

---

### 2. Buat direktori volume Prometheus

Prometheus menyimpan data time-series ke volume ini. Buat dulu sebelum menjalankan container, supaya Docker tidak auto-create dengan permission `root`.

```bash
sudo mkdir -p /var/docker-volume/monitoring/prometheus
sudo chown -R 65534:65534 /var/docker-volume/monitoring/prometheus
```

> `65534` adalah UID `nobody` — user yang dipakai image `prom/prometheus` secara default.

---

### 3. Setting `.env`

Copy dari template lalu sesuaikan:

```bash
cp .env.example .env
```

Edit `.env`:

```env
# IP address server ini — digunakan untuk binding port ke interface spesifik
IP_HOST=192.168.0.193

# Port Prometheus
PROMETHEUS_PORT=9090

# Port Node Exporter
NODE_EXPORTER_PORT=9100

# Port cAdvisor
CADVISOR_PORT=8080
```

> **Penting:** `IP_HOST` harus IP interface yang aktif di server. Cek dengan `ip a`.

---

### 4. Setting `prometheus.yml`

File ini **tidak** support interpolasi dari `.env`, jadi IP dan port perlu di-hardcode secara manual.

Sesuaikan bagian `scrape_configs` dengan nilai yang sama seperti di `.env`:

```yaml
scrape_configs:

  - job_name: "node_exporter"
    static_configs:
      - targets: ["192.168.0.193:9100"]   # IP_HOST:NODE_EXPORTER_PORT
        labels:
          server: "nama-server-kamu"

  - job_name: "cadvisor"
    static_configs:
      - targets: ["192.168.0.193:8080"]   # IP_HOST:CADVISOR_PORT
        labels:
          server: "nama-server-kamu"
```

> Kalau IP atau port di `.env` diubah, jangan lupa update `prometheus.yml` juga secara manual.

---

### 5. Jalankan stack

```bash
docker compose up -d
```

Cek semua container berjalan:

```bash
docker compose ps
```

Verifikasi Prometheus bisa scrape semua target — buka di browser:

```
http://<IP_HOST>:<PROMETHEUS_PORT>/targets
```

Semua target harus berstatus **UP**.

---

## Struktur File

```
metrics-infra/
├── docker-compose.yml     # Definisi service
├── .env                   # Konfigurasi port & IP (tidak di-commit)
├── .env.example           # Template .env
├── prometheus.yml         # Konfigurasi scrape Prometheus
├── web.yml                # Konfigurasi web server Prometheus (basic auth, TLS)
└── README.md
```

---

## Operasional

### Restart stack

```bash
docker compose restart
```

### Reload konfigurasi Prometheus (tanpa restart)

```bash
curl -X POST http://<IP_HOST>:<PROMETHEUS_PORT>/-/reload
```

> Pastikan flag `--web.enable-lifecycle` aktif di `command` Prometheus jika ingin pakai ini.

### Lihat log

```bash
# Semua service
docker compose logs -f

# Per service
docker compose logs -f prometheus
docker compose logs -f node-exporter
docker compose logs -f cadvisor
```

### Stop stack

```bash
docker compose down
```

---

## Troubleshooting

**Target di Prometheus berstatus DOWN**
- Pastikan `IP_HOST` dan port di `prometheus.yml` sesuai dengan yang berjalan
- Cek apakah container node-exporter/cadvisor sudah running: `docker compose ps`
- Cek firewall: `sudo ufw status` — port harus allow dari IP Prometheus

**Container gagal start**
- Cek permission volume: `ls -la /var/docker-volume/monitoring/prometheus`
- Cek port tidak bentrok: `ss -tlnp | grep <PORT>`

**cAdvisor tidak jalan / permission error**
- Pastikan `privileged: true` ada di `docker-compose.yml`
- Device `/dev/kmsg` harus accessible di host

---

## Catatan

- Data Prometheus disimpan selama **3 hari** (`--storage.tsdb.retention.time=3d`). Sesuaikan di `docker-compose.yml` jika perlu lebih lama.
- File `.env` **jangan di-commit** ke Git. Pastikan sudah ada di `.gitignore`.
- `web.yml` digunakan untuk konfigurasi basic auth / TLS Prometheus — jaga kerahasiaannya.