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
