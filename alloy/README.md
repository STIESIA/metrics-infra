# Grafana Alloy – Log Shipper

Alloy dijalankan di tiap VM aplikasi. Tugasnya membaca file log
(`storage/logs/*.log`) lalu mengirimkannya ke Loki. Log dilihat lewat Grafana.

```
VM-1 (Alloy) ─┐
              ├─> Loki (VM devops) <─ Grafana
VM-2 (Alloy) ─┘
```

## Struktur

```
grafana-alloy/
├── .gitignore
├── config.example.alloy          # template config, di-commit
├── docker-compose.example.yaml   # template compose, di-commit
├── config.alloy                  # config asli per VM, TIDAK di-commit
└── docker-compose.yml            # compose asli per VM, TIDAK di-commit
```

`.gitignore`:

```
config.alloy
docker-compose.yml
```

## Prasyarat

- Docker dan Docker Compose
- Loki sudah jalan dan bisa dijangkau dari VM ini:
  `curl http://<LOKI_HOST>:3100/ready`
- App (Laravel/Go/Express/etc..) menulis log ke file (channel `daily`), dan folder log-nya sudah
  ter-mount ke host, contoh: `/var/docker-volume/app1/logs/app-2026-09-24.log`
- Contoh Format log : `[2026-09-24 10:00:00] production.ERROR: ...`

## Setup

**1. Salin template**

```bash
cp docker-compose.example.yaml docker-compose.yml
cp config.example.alloy config.alloy
```

**2. Edit `config.alloy`**

- Daftar app di `path_targets` (path log dan nama `app`)
- URL Loki di `loki.write`

**3. Edit `docker-compose.yml`**

- Tambah satu mount read-only untuk tiap folder log app
- **Path di container harus sama persis dengan path di `config.alloy`**

Contoh, kalau di `config.alloy` ada:

```alloy
{__path__ = "/var/docker-volume/app1/logs/*.log", app = "app1"},
```

maka di compose:

```yaml
- /var/docker-volume/app1/logs:/var/docker-volume/app1/logs:ro
```

**4. Buat folder state, lalu jalankan**

```bash
sudo mkdir -p /var/docker-volume/alloy_data
docker compose up -d
docker compose logs -f alloy
```

Di log tidak boleh ada `failed to evaluate config`.

## Cara kerja config

| Blok               | Fungsi                                                          |
| ------------------ | --------------------------------------------------------------- |
| `local.file_match` | Menentukan file log yang dibaca dan label `app` per file        |
| `loki.source.file` | Membaca file lalu meneruskannya ke pipeline                     |
| `stage.multiline`  | Menggabungkan stack trace multi-baris menjadi satu entry        |
| `stage.regex`      | Mengambil `ts` (waktu) dan `level` dari baris pertama           |
| `stage.timestamp`  | Memakai jam dari baris log, bukan waktu Alloy membaca (WIB)     |
| `stage.labels`     | Menjadikan `level` sebagai label (ERROR, WARNING, INFO, dst)    |
| `loki.write`       | Mengirim ke Loki                                                |

Label yang dihasilkan: `app`, `level`, `filename`.

Nama `stiesia_apps` hanya nama komponen (bebas), harus konsisten di semua blok
dan memakai underscore, bukan strip.

## Menambah app baru

1. Tambah baris di `path_targets` pada `config.alloy`:
   ```alloy
   {__path__ = "/var/docker-volume/app3/logs/*.log", app = "app3"},
   ```
2. Tambah mount-nya di `docker-compose.yml`:
   ```yaml
   - /var/docker-volume/app3/logs:/var/docker-volume/app3/logs:ro
   ```
3. Terapkan ulang:
   ```bash
   docker compose up -d --force-recreate
   ```

Nama `app` dipakai sebagai label di Loki dan filter di Grafana. Buat pendek,
huruf kecil, tanpa spasi.

## Verifikasi

Di Grafana Explore (datasource Loki):

```
{app="app1"}
{app="app1", level="ERROR"}
sum by (app) (count_over_time({app=~".+"}[1d]))
```

Memancing log error dari container app:

```bash
docker exec <container-app> php artisan tinker --execute="Log::error('test loki');"
```

## Catatan penting

- **Timestamp ikut jam di baris log.** Filter waktu di Grafana (misal
  *Yesterday*) mengikuti tanggal asli log, selama file-nya masih ada di disk.
- **Alloy membaca file dari awal** saat pertama menemukannya, lalu mengingat
  posisi di volume `alloy_data`. Jangan hapus volume itu sembarangan, kalau
  dihapus log akan terbaca ulang dan dobel.
- **Kalau label atau `stage.timestamp` diubah setelah data masuk**, log lama
  tetap memakai label dan timestamp lama. Untuk mengulang dari nol: hentikan
  Alloy dan Loki, kosongkan `alloy_data` dan data Loki, lalu jalankan lagi.
- **Loki menolak log yang lebih tua dari 7 hari** (default
  `reject_old_samples_max_age`). Naikkan limit itu di konfigurasi Loki kalau perlu.
- **Zona waktu** di `stage.timestamp` diset `Asia/Jakarta`.
- Loki tidak punya auth bawaan. Batasi akses port Loki lewat firewall atau
  reverse proxy dengan basic auth.

## Troubleshooting

| Gejala                                   | Penyebab umum                                                          |
| ---------------------------------------- | ---------------------------------------------------------------------- |
| `unrecognized block name "stage.regexp"` | Nama yang benar di Alloy adalah `stage.regex`                          |
| `no such file or directory`              | Mount di compose tidak cocok dengan path di `config.alloy`             |
| `permission denied`                      | Cek permission folder log di host                                      |
| `connection refused` ke Loki             | Loki belum jalan, IP/port salah, atau diblok firewall                  |
| `entry too far behind` / `too old`       | Baris log lebih tua dari batas Loki                                    |
| Log tidak muncul di Grafana              | Perbesar range waktu, cek `docker compose logs alloy`, cek label `app` |
| Filter *Yesterday* kosong                | Data lama masuk dengan timestamp lama, atau file kemarin sudah hilang  |

Cek cepat dari VM aplikasi:

```bash
docker compose logs alloy --tail 50
curl -s http://<LOKI_HOST>:3100/loki/api/v1/labels
```