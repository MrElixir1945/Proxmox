# Deploy Web Statis via Cloudflare Tunnel

Panduan singkat untuk mempublikasikan web statis dari server lokal ke internet menggunakan Cloudflare Tunnel, tanpa perlu membuka port di router.

---

## Prasyarat

- Akun Cloudflare dengan domain yang sudah terdaftar
- Akses ke [Cloudflare Zero Trust Dashboard](https://one.dash.cloudflare.com)
- Server Linux dengan akses root/sudo

---

## Spesifikasi Environment (Setup Pribadi)

Setup ini berjalan di atas **Proxmox LXC Container** dengan spesifikasi berikut:

| Komponen       | Spesifikasi              |
|----------------|--------------------------|
| CT Template    | Ubuntu 24.04 LTS         |
| CPU            | 2 Core                   |
| RAM            | 1 GB                     |
| Swap           | 0 (disabled)             |
| Disk           | 20 GB                    |
| Type           | LXC / CT (Unprivileged)  |

### Catatan Spesifikasi

**RAM 1 GB tanpa swap** — Cukup untuk Python `http.server` + `cloudflared`. Hindari menjalankan proses berat lain secara bersamaan. Pantau penggunaan memori dengan:

```bash
free -h
```

Jika ingin menambah sedikit buffer saat memory pressure tinggi, kamu bisa aktifkan swap kecil (opsional):

```bash
# Buat swap file 512MB (opsional, tidak wajib)
fallocate -l 512M /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab
```

**Disk 20 GB** — Lebih dari cukup untuk web statis. Pantau penggunaan disk dengan:

```bash
df -h
```

**CPU 2 Core** — Ideal untuk beban ringan seperti static file serving. `cloudflared` dan Python `http.server` sangat ringan di CPU.

---

## 1. Jalankan Web Server Lokal

Buat systemd service agar web server otomatis berjalan saat server hidup. Jangan gunakan `nohup` karena tidak persistent setelah reboot.

Buat file service:

```bash
nano /etc/systemd/system/web-server.service
```

Isi dengan konfigurasi berikut (sesuaikan `WorkingDirectory` dengan path project kamu):

```ini
[Unit]
Description=Python Web Server
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/var/www/my-project
ExecStart=/usr/bin/python3 -m http.server 8080
Restart=always

[Install]
WantedBy=multi-user.target
```

Simpan (`Ctrl+O`, `Enter`, `Ctrl+X`), lalu aktifkan:

```bash
systemctl daemon-reload
systemctl enable web-server.service
systemctl start web-server.service
```

Server berjalan di `localhost:8080` dan akan otomatis start ulang saat reboot.

---

## 2. Install Cloudflare Tunnel (cloudflared)

1. Buka **Cloudflare Zero Trust Dashboard** > Networks > Tunnels
2. Buat tunnel baru dan salin token yang diberikan
3. Jalankan perintah berikut di server:

```bash
cloudflared service install <YOUR_TUNNEL_TOKEN>
```

4. Pastikan status tunnel menunjukkan **Healthy** di dashboard.

`cloudflared service install` secara otomatis mendaftarkan diri ke systemd. Jika perlu mengaktifkan manual:

```bash
systemctl enable cloudflared
systemctl start cloudflared
```

---

## 3. Konfigurasi Public Hostname

Di dashboard tunnel, buka tab **Public Hostname** dan tambahkan:

| Field     | Value          |
|-----------|----------------|
| Hostname  | your.domain.com |
| Type      | HTTP           |
| URL       | localhost:8080 |

---

## 4. Konfigurasi DNS

Buka **DNS Records** di Cloudflare dan tambahkan record berikut:

| Field         | Value                        |
|---------------|------------------------------|
| Type          | CNAME                        |
| Name          | your.domain.com atau @       |
| Target        | `<tunnel-id>.cfargotunnel.com` |
| Proxy Status  | Proxied (ikon oranye)        |

---

## 5. Verifikasi

Cek status tunnel:
```bash
systemctl status cloudflared
```

Cek apakah port lokal aktif:
```bash
netstat -tulnp | grep 8080
```

---

## Troubleshooting

Jika domain belum bisa diakses setelah setup, coba flush DNS di perangkat klien.

**Linux:**
```bash
sudo resolvectl flush-caches
```

**Windows:**
```bash
ipconfig /flushdns
```

---

## Auto Start: Proxmox LXC Container

Jika server berjalan di dalam LXC container Proxmox, pastikan container ikut menyala saat Proxmox di-reboot.

1. Buka Proxmox Web UI
2. Pilih container yang digunakan
3. Masuk ke **Options**
4. Klik **Start at boot** > Edit > centang **Yes** > OK

Dengan ini, saat Proxmox hidup, container otomatis start, dan karena web server serta cloudflared sudah terdaftar di systemd, keduanya ikut berjalan otomatis.

---

## Catatan Penggunaan Pribadi: CasaOS File Browser

Setup ini menggunakan **CasaOS** sebagai file manager untuk mengelola source code frontend langsung dari browser, tanpa perlu SSH setiap kali ingin update file.

### Install CasaOS

Kompatibel dengan Ubuntu, Debian, Raspberry Pi OS, dan CentOS. Jalankan salah satu perintah berikut:

```bash
curl -fsSL https://get.casaos.io | sudo bash
```

atau

```bash
wget -qO- https://get.casaos.io | sudo bash
```

Setelah selesai, akses CasaOS di browser via `http://<ip-server>`.

Untuk update:

```bash
curl -fsSL https://get.casaos.io/update | sudo bash
```

Untuk uninstall:

```bash
casaos-uninstall
```

### Alur Kerja

1. Upload atau edit file frontend via **CasaOS File Browser**
2. Pastikan direktori yang digunakan CasaOS sama dengan direktori yang dilayani web server

### Sinkronisasi Direktori

Arahkan `WorkingDirectory` di file service systemd ke direktori yang diakses CasaOS. Secara default CasaOS menyimpan file di `/DATA/`:

```ini
WorkingDirectory=/DATA/my-project
```

Dengan begitu, setiap perubahan file yang dilakukan lewat CasaOS File Browser langsung tercermin di website tanpa perlu restart server.
