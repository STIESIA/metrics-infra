# metrics-infra

Stack monitoring berbasis Prometheus untuk server metrics STIESIA. Terdiri dari **Prometheus**, **Node Exporter**, dan **cAdvisor** — dikemas via Docker Compose untuk provisioning cepat di server manapun.

## Daftar Isi

- [Stack](#stack)
- [Arsitektur](#arsitektur)
- [Prerequisites](#prerequisites)
- [Setup](#setup)
- [Firewall (UFW)](#firewall-ufw)
- [Generate Password `web.yml`](#generate-password-webyml)
- [Struktur File](#struktur-file)
- [Operasional](#operasional)
- [Troubleshooting](#troubleshooting)
- [Catatan](#catatan)

---

## Stack

| Service | Fungsi | Default Port |
|---|---|---|
| Prometheus | Time-series database & scrape engine | `9090` |
| Node Exporter | Metrics OS & hardware (CPU, RAM, disk, network) | `9100` |
| cAdvisor | Metrics container Docker | `8080` |

## Arsitektur

```
                 ┌─────────────┐
   scrape (9100) │Node Exporter│  (host metrics)
        ┌────────┤             │
        │        └─────────────┘
        │
┌───────▼───────┐    ┌─────────────┐
│   Prometheus   │◄───┤   cAdvisor  │  (container metrics)
│    (9090)      │scrape (8080)
└────────────────┘
```

Prometheus scrape kedua exporter secara periodik dan menyimpan hasilnya sebagai time-series data. Semua service berjalan di satu host lewat Docker Compose — cocok untuk monitoring per-VM (bukan multi-cluster).

---

## Prerequisites

- Docker & Docker Compose sudah terinstall
- Akses `sudo` di server target
- Port `9090`, `9100`, `8080` tidak dipakai service lain (cek dengan `ss -tlnp`)

---

## Setup

### 1. Clone repo

```bash
git clone <repo-url> metrics-infra
cd metrics-infra
```

### 2. Buat direktori volume Prometheus

Prometheus menyimpan data time-series ke volume ini. Buat dulu sebelum menjalankan container, supaya Docker tidak auto-create dengan permission `root`.

```bash
sudo mkdir -p /var/docker-volume/metrics/prometheus
sudo chown -R 65534:65534 /var/docker-volume/metrics/prometheus
```

> `65534` adalah UID `nobody` — user yang dipakai image `prom/prometheus` secara default.

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

### 4. Setting `prometheus.yml`

File ini **tidak** support interpolasi dari `.env`, jadi IP dan port perlu di-hardcode secara manual. Sesuaikan bagian `scrape_configs` dengan nilai yang sama seperti di `.env`:

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

## Firewall (UFW)

Izinkan akses ke port monitoring dari subnet internal.

```bash
sudo ufw allow from 192.168.0.0/24 to any port 9090
sudo ufw allow from 192.168.0.0/24 to any port 9100
sudo ufw allow from 192.168.0.0/24 to any port 8080
```

Reload dan cek status:

```bash
sudo ufw reload
sudo ufw status numbered
```

> Ganti `192.168.0.0/24` dengan subnet atau IP spesifik yang sesuai environment kamu. Jangan expose port ini ke internet publik tanpa basic auth/TLS (lihat `web.yml`).

---

## Generate Password `web.yml`

`web.yml` dipakai Prometheus untuk basic auth (`--web.config.file`). Password-nya harus dalam bentuk **bcrypt hash**, bukan plaintext.

```bash
docker run --rm httpd:2.4-alpine htpasswd -nbBC 10 admin passwordkamu
```

Output-nya berupa `admin:$2y$10$xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx` — ambil bagian hash setelah tanda `:`, lalu masukkan ke `web.yml`:

```yaml
basic_auth_users:
  admin: $2y$10$xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

> Setelah edit `web.yml`, restart/reload Prometheus (lihat [Operasional](#operasional)) supaya perubahan kepake. Jangan commit `web.yml` yang sudah berisi hash asli — pastikan sudah masuk `.gitignore`.

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

### Update image

```bash
docker compose pull
docker compose up -d
```

---

## Troubleshooting

**Target di Prometheus berstatus DOWN**
- Pastikan `IP_HOST` dan port di `prometheus.yml` sesuai dengan yang berjalan
- Cek apakah container node-exporter/cadvisor sudah running: `docker compose ps`
- Cek firewall: `sudo ufw status` — port harus allow dari IP Prometheus

**Container gagal start**
- Cek permission volume: `ls -la /var/docker-volume/metrics/prometheus`
- Cek port tidak bentrok: `ss -tlnp | grep <PORT>`

**cAdvisor tidak jalan / permission error**
- Pastikan `privileged: true` ada di `docker-compose.yml`
- Device `/dev/kmsg` harus accessible di host

---

## Catatan

- Data Prometheus disimpan selama **3 hari** (`--storage.tsdb.retention.time=3d`). Sesuaikan di `docker-compose.yml` jika perlu lebih lama.
- File `.env` **jangan di-commit** ke Git. Pastikan sudah ada di `.gitignore`.
- `web.yml` digunakan untuk konfigurasi basic auth / TLS Prometheus — jaga kerahasiaannya.
- Stack ini belum termasuk visualisasi (Grafana) — Prometheus expose data lewat `/graph` dan `/targets` bawaan, tapi untuk dashboard yang lebih enak dilihat pertimbangkan tambah Grafana sebagai service terpisah ke depannya.