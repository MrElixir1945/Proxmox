# Deploy Web Statis via Cloudflare Tunnel

Panduan singkat untuk mempublikasikan web statis dari server lokal ke internet menggunakan Cloudflare Tunnel, tanpa perlu membuka port di router.

---

## Prasyarat

- Akun Cloudflare dengan domain yang sudah terdaftar
- Akses ke [Cloudflare Zero Trust Dashboard](https://one.dash.cloudflare.com)
- Server Linux dengan akses root/sudo

---

## 1. Jalankan Web Server Lokal

Pastikan file website berada di satu direktori, lalu jalankan server lokal.

```bash
cd /var/www/my-project
nohup python3 -m http.server 8080 > /dev/null 2>&1 &
```

Server berjalan di `localhost:8080`.

---

## 2. Install Cloudflare Tunnel (cloudflared)

1. Buka **Cloudflare Zero Trust Dashboard** > Networks > Tunnels
2. Buat tunnel baru dan salin token yang diberikan
3. Jalankan perintah berikut di server:

```bash
cloudflared service install <YOUR_TUNNEL_TOKEN>
```

4. Pastikan status tunnel menunjukkan **Healthy** di dashboard.

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

Saat menjalankan web server, arahkan ke direktori yang dapat diakses CasaOS. Secara default CasaOS menyimpan file di `/DATA/`, sesuaikan path-nya:

```bash
cd /DATA/my-project
nohup python3 -m http.server 8080 > /dev/null 2>&1 &
```

Dengan begitu, setiap perubahan file yang dilakukan lewat CasaOS File Browser langsung tercermin di website tanpa perlu restart server.
