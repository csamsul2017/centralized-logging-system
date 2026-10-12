# Centralized Observability (Logging & Metrics)

Stack observabilitas terpusat menggunakan **Loki**, **Prometheus**, dan **Grafana** berbasis Docker Compose.

Metrik sistem (CPU, RAM, Disk, dll) dan log dikirim melalui **Grafana Alloy** langsung ke stack ini.

---

## 🏗️ Komponen Stack

| Service | Port | Deskripsi |
| :--- | :--- | :--- |
| **Grafana** | `3000` | Dashboard visualisasi untuk log (Loki) dan metrik (Prometheus) |
| **Prometheus** | `9090` | Time-series database (dengan Remote Write Receiver aktif) |
| **Loki** | `3100` | Log aggregation system |

---

## 🚀 Cara Menjalankan

### 1. Siapkan Environment Variable
Salin contoh file `.env.example` ke `.env`:

```bash
cp .env.example .env
```

Isi kredensial admin Grafana pada file `.env`:
```env
GF_SECURITY_ADMIN_USER=admin
GF_SECURITY_ADMIN_PASSWORD=admin
```

### 2. Jalankan Stack
Jalankan container menggunakan Docker Compose:

```bash
docker compose up -d
```

Cek status container:
```bash
docker compose ps
```

---

## ⚡ Integrasi Grafana Alloy (Metrics & Logs)

Prometheus telah dikonfigurasi dengan flag `--web.enable-remote-write-receiver`, sehingga **Grafana Alloy** dapat langsung mengirim metrik CPU, RAM, OS via Remote Write endpoint:
- **Prometheus Remote Write endpoint**: `http://<HOST_IP>:9090/api/v1/write`
- **Loki Push endpoint**: `http://<HOST_IP>:3100/loki/api/v1/push`

### Contoh Konfigurasi Grafana Alloy (`config.alloy`)

```alloy
// 1. Export metrik sistem (CPU, RAM, Disk, Network)
// Gunakan prometheus.exporter.unix untuk Linux atau prometheus.exporter.windows untuk Windows
prometheus.exporter.unix "node" { }

prometheus.scrape "node_metrics" {
  targets    = prometheus.exporter.unix.node.targets
  forward_to = [prometheus.remote_write.local_prom.receiver]
}

// 2. Kirim metrik ke Prometheus
prometheus.remote_write "local_prom" {
  endpoint {
    url = "http://localhost:9090/api/v1/write"
  }
}

// 3. Kirim log ke Loki
loki.write "local_loki" {
  endpoint {
    url = "http://localhost:3100/loki/api/v1/push"
  }
}
```

---

## 📊 Akses & Konfigurasi Grafana

1. Buka browser dan akses **`http://localhost:3000`**.
2. Login menggunakan username dan password yang telah diatur di `.env`.
3. Datasource **Loki** dan **Prometheus** sudah terhubung otomatis (*auto-provisioned*).
   - Verifikasi melalui menu **Connections** > **Data sources**.

---

## 📈 Query PromQL Umum untuk CPU & RAM

Di Grafana menu **Explore** (pilih datasource **Prometheus**):

- **CPU Usage (%)**:
  ```promql
  100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[1m])) * 100)
  ```

- **RAM Terpakai (%)**:
  ```promql
  (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100
  ```

- **RAM Terpakai (GB)**:
  ```promql
  (node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / 1024 / 1024 / 1024
  ```

- **Disk Usage (%)**:
  ```promql
  100 - ((node_filesystem_avail_bytes{mountpoint="/"} * 100) / node_filesystem_size_bytes{mountpoint="/"})
  ```

---

## 📋 Rekomendasi Dashboard Grafana

Untuk mendapatkan dashboard visual lengkap dengan grafik CPU, Memory, Disk, dan Network:

1. Di Grafana, klik **Dashboards** > **New** > **Import**.
2. Masukkan Dashboard ID: **`1860`** (*Node Exporter Full* - kompatibel penuh dengan metrik dari Grafana Alloy `prometheus.exporter.unix`).
3. Klik **Load**.
4. Pilih datasource **Prometheus**, lalu klik **Import**.
