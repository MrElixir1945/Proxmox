# Instalasi Immich di Proxmox LXC

Dokumentasi instalasi Immich menggunakan LXC Container dengan pemisahan disk sistem dan disk data untuk memastikan persistensi data media.

---

## Arsitektur Setup

```
Proxmox Host
└── LXC Container (Ubuntu 24.04)
    ├── Disk OS  (rootfs)     → sistem operasi + Docker + config Immich
    └── Disk Data (mp0)       → /mnt/immich-data → foto & video
```

> Memisahkan disk data dari disk OS memastikan file media tetap aman meskipun container rusak atau perlu di-reinstall.

---

## 1. Persiapan Resource LXC

Buat container baru di Proxmox dengan spesifikasi berikut:

| Komponen         | Keterangan                                      |
|------------------|-------------------------------------------------|
| OS Template      | Ubuntu 24.04 LTS                                |
| Disk OS (rootfs) | Sesuaikan kebutuhan (minimal 10–20 GB)          |
| Disk Data (mp0)  | Minimal 100 GB, di-mount ke `/mnt/immich-data`  |
| Network          | IP Statis sesuai segmen jaringan lokal          |
| Mode Container   | **Unprivileged**                                |

### Cara Tambah Disk Data (mp0) di Proxmox

1. Pilih container di Proxmox Web UI
2. Buka **Resources** > klik **Add** > pilih **Mount Point**
3. Isi path: `/mnt/immich-data`, tentukan ukuran disk
4. Klik **Add**

---

## 2. Konfigurasi Fitur LXC (di Host Proxmox)

Docker membutuhkan fitur kernel khusus agar bisa berjalan di dalam LXC Unprivileged. Konfigurasi ini dilakukan di **shell Host Proxmox**, bukan di dalam container.

```bash
nano /etc/pve/lxc/[ID_CONTAINER].conf
```

Tambahkan baris berikut di bagian bawah file:

```
features: nesting=1,keyctl=1
```

Simpan (`Ctrl+O`, `Enter`, `Ctrl+X`), lalu start container.

> **Kenapa perlu ini?**
> - `nesting=1` → mengizinkan Docker berjalan di dalam LXC
> - `keyctl=1` → diperlukan untuk manajemen kernel keyring oleh Docker

---

## 3. Setup Docker (di dalam Console LXC)

Masuk ke console container, lalu jalankan perintah berikut.

### Install Docker

```bash
apt update && apt upgrade -y
apt install -y curl wget nano
curl -fsSL https://get.docker.com | sh
```

### Buat Direktori yang Diperlukan

```bash
mkdir -p /opt/immich
mkdir -p /mnt/immich-data
chown -R root:root /mnt/immich-data
chmod -R 755 /mnt/immich-data
```

> `/opt/immich` → tempat menyimpan konfigurasi Docker Compose
> `/mnt/immich-data` → tempat menyimpan foto dan video (disk data terpisah)

---

## 4. Konfigurasi Immich

### Download File Konfigurasi

```bash
cd /opt/immich
wget -O docker-compose.yml https://github.com/immich-app/immich/releases/latest/download/docker-compose.yml
wget -O .env https://github.com/immich-app/immich/releases/latest/download/example.env
```

### Edit File `.env`

```bash
nano .env
```

Sesuaikan dua baris berikut:

```env
UPLOAD_LOCATION=/mnt/immich-data
TZ=Asia/Makassar
```

> Ganti `TZ` dengan timezone lokal kamu. Untuk Indonesia:
> - WIB → `Asia/Jakarta`
> - WITA → `Asia/Makassar`
> - WIT → `Asia/Jayapura`

---

## 5. Jalankan Immich

```bash
docker compose up -d
```

Cek status semua service:

```bash
docker compose ps
```

Semua service berikut harus berstatus **running**:

| Service            | Keterangan                        |
|--------------------|-----------------------------------|
| `immich_server`    | Backend utama Immich              |
| `immich_machine_learning` | Fitur pengenalan wajah/objek |
| `database`         | PostgreSQL                        |
| `redis`            | Cache                             |

---

## 6. Akses Immich

Buka browser dan akses:

```
http://[IP_LXC]:2283
```

Buat akun admin pada saat pertama kali membuka halaman tersebut.

---

## Troubleshooting

**Docker gagal start di LXC:**
Pastikan baris `features: nesting=1,keyctl=1` sudah ada di file `.conf` container dan container sudah di-restart setelah perubahan.

**Disk data tidak terbaca:**
Cek apakah mount point sudah terpasang dengan benar:
```bash
df -h | grep immich
```
Jika kosong, cek konfigurasi mp0 di Proxmox Web UI.

**Service tidak running setelah reboot:**
Docker Compose tidak otomatis start ulang kecuali policy restart sudah diset. Pastikan semua service di `docker-compose.yml` memiliki:
```yaml
restart: unless-stopped
```
Ini biasanya sudah ada di file resmi Immich, tapi pastikan tetap ada setelah update.
