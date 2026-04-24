# Setup Remote Access: NPM & Cloudflare Tunnel

## Konsep Jaringan

Cloudflare Tunnel menghubungkan jaringan lokal ke internet **tanpa perlu membuka port** di MikroTik. Nginx Proxy Manager (NPM) bertugas sebagai reverse proxy yang meneruskan domain ke container yang tepat.

```
Internet → Cloudflare Edge → Cloudflare Tunnel (LXC 114) → NPM (LXC 114) → Target Service (LXC lain)
```

> NPM dan Cloudflared berjalan di container yang sama (LXC 114) menggunakan Docker Compose.

---

## Spesifikasi Container (LXC 114)

| Komponen | Spesifikasi    | Fungsi                              |
|----------|----------------|-------------------------------------|
| OS       | Ubuntu 24.04   | Base sistem                         |
| CPU      | 1 Core         | Menangani trafik proxy & tunnel     |
| RAM      | 1 GB           | Alokasi Docker engine               |
| Storage  | 10 GB          | Database NPM & log trafik           |
| Network  | Static IP      | Gateway internal untuk tunnel       |

---

## Instalasi Docker & NPM

### 1. Install Docker

```bash
apt update && apt upgrade -y
curl -fsSL https://get.docker.com | sh
```

### 2. Buat File Docker Compose

```bash
mkdir ~/npm && cd ~/npm
nano docker-compose.yml
```

Isi dengan konfigurasi berikut:

```yaml
services:
  app:
    image: 'jc21/nginx-proxy-manager:latest'
    restart: unless-stopped
    ports:
      - '80:80'    # HTTP
      - '81:81'    # Dashboard NPM
      - '443:443'  # HTTPS
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt

  tunnel:
    image: cloudflare/cloudflared:latest
    restart: unless-stopped
    command: tunnel --no-autoupdate run --token <TOKEN_TUNNEL_ANDA>
```

> Ganti `<TOKEN_TUNNEL_ANDA>` dengan token dari **Cloudflare Zero Trust Dashboard > Networks > Tunnels**.

### 3. Jalankan Service

```bash
docker compose up -d
```

Akses dashboard NPM di `http://[IP_LXC_114]:81`

| Kredensial | Default             |
|------------|---------------------|
| Email      | `admin@example.com` |
| Password   | `changeme`          |

> **Segera ganti email dan password setelah login pertama.**

---

## Konfigurasi Cloudflare Tunnel

Buka **Cloudflare Zero Trust Dashboard > Networks > Tunnels > [nama tunnel] > Public Hostnames**, lalu tambahkan entry baru:

| Field        | Nilai                     | Keterangan                        |
|--------------|---------------------------|-----------------------------------|
| Subdomain    | `immich` / `nextcloud`    | Nama subdomain yang diinginkan    |
| Domain       | `domainanda.com`          | Domain yang terdaftar di CF       |
| Service Type | `HTTP`                    | Protokol ke NPM                   |
| URL          | `10.10.x.x:80`            | IP container NPM                  |
| Path         | *(kosongkan)*             | Wajib kosong, jika diisi bisa menyebabkan error 1033 |

---

## Konfigurasi Proxy Host (NPM)

Tambahkan Proxy Host baru di NPM untuk meneruskan domain ke aplikasi target.

### Tab Details

| Field                  | Nilai                        |
|------------------------|------------------------------|
| Domain Names           | `subdomain.domainanda.com`   |
| Scheme                 | `http`                       |
| Forward Hostname / IP  | IP container target          |
| Forward Port           | Port aplikasi (contoh: `2283` untuk Immich) |
| Websockets Support     | **ON**                       |

> Websockets wajib aktif untuk aplikasi yang menggunakan koneksi real-time seperti Immich.

### Tab SSL

| Field           | Nilai   | Keterangan                                    |
|-----------------|---------|-----------------------------------------------|
| SSL Certificate | `None`  | Enkripsi sudah ditangani di sisi Cloudflare Edge |
| Force SSL       | **ON**  | Redirect HTTP → HTTPS otomatis                |

---

## Optimasi Jaringan Lokal (Split DNS)

Tanpa konfigurasi ini, akses dari dalam jaringan rumah ke `subdomain.domainanda.com` akan keluar ke internet dahulu lalu balik lagi — tidak efisien.

Dengan menambahkan **Local DNS Record di Pi-hole**, akses dari lokal langsung diarahkan ke IP NPM tanpa keluar ke internet.

**Pi-hole Dashboard > Local DNS > DNS Records:**

| Field      | Nilai                      |
|------------|----------------------------|
| Domain     | `subdomain.domainanda.com` |
| IP Address | IP container NPM           |

---

## Troubleshooting

| Gejala                                  | Solusi                                                                 |
|-----------------------------------------|------------------------------------------------------------------------|
| Status tunnel bukan HEALTHY di CF       | Cek log: `docker compose logs tunnel`                                  |
| Muncul halaman default Nginx di browser | Domain Name di Proxy Host NPM tidak cocok dengan yang diakses         |
| Aplikasi mobile gagal login / upload    | Aktifkan **Websockets Support** di Proxy Host NPM                     |
| Error 1033 dari Cloudflare              | Kosongkan kolom **Path** di konfigurasi Public Hostname Cloudflare    |
| Service tidak jalan setelah reboot      | Pastikan `restart: unless-stopped` ada di semua service di Compose file |

---

## Infrastruktur Keseluruhan

Setup ini berjalan sepenuhnya di atas **Proxmox** menggunakan LXC Container, dengan setiap service diisolasi di container masing-masing untuk efisiensi resource dan kemudahan manajemen.

```
Proxmox Host
├── LXC (DNS Filter)      → Pi-hole, menyaring iklan & query DNS
├── LXC (Reverse Proxy)   → NPM + Cloudflare Tunnel, menangani akses dari luar
└── LXC (Self-hosted App) → Aplikasi utama (foto, cloud, dsb.)
```

Jaringan dikelola oleh **Router** yang bertugas sebagai:
- **DHCP Server** → mendistribusikan DNS ke Pi-hole untuk semua perangkat
- **DoH (DNS over HTTPS)** → mengenkripsi trafik DNS sebelum keluar ke internet
- **Network Isolation** → jaringan lab dipisah dari jaringan utama agar tidak saling mengganggu

Dengan arsitektur ini, semua layanan self-hosted dapat diakses dari internet maupun lokal secara aman, tanpa perlu membuka satu pun port di router.
