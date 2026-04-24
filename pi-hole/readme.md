# Setup Pi-hole dan Jaringan Lab

## Konsep Jaringan

Trafik DNS dari perangkat client diarahkan ke Pi-hole untuk penyaringan iklan. Jika domain bukan iklan, Pi-hole meneruskan query ke Router untuk dienkripsi via **DNS over HTTPS (DoH)** sebelum keluar ke internet melalui ISP.

```
Client → Pi-hole (filter iklan) → Router/MikroTik (DoH) → ISP → Internet
```

---

## Spesifikasi Container (LXC)

| Komponen | Spesifikasi         |
|----------|---------------------|
| OS       | Ubuntu 24.04 LTS    |
| CPU      | 1 Core              |
| RAM      | 512 MB              |
| Storage  | 8 GB                |
| Network  | Static IP           |

> Pastikan IP container di-set statis agar IP Pi-hole tidak berubah saat reboot.

---

## Instalasi Pi-hole

```bash
# 1. Update sistem
apt update && apt upgrade -y

# 2. Install curl
apt install curl -y

# 3. Jalankan installer
curl -sSL https://install.pi-hole.net | bash
```

### Konfigurasi Wizard

| Opsi            | Nilai                        |
|-----------------|------------------------------|
| Interface       | `eth0`                       |
| Upstream DNS    | Custom → IP Router/MikroTik  |
| Web Interface   | Enabled                      |
| Logging         | Enabled                      |
| Privacy Mode    | `0` (Show everything)        |

---

## Konfigurasi DNS Lanjutan

### 1. Pi-hole Dashboard

Masuk ke **Settings > DNS**, lalu:

- Nonaktifkan semua upstream DNS provider publik (Google, Cloudflare, dll.)
- Isi **Custom 1 (IPv4)** dengan IP Router/MikroTik

Dengan ini, semua query yang lolos filter Pi-hole akan diteruskan ke MikroTik untuk diproses via DoH.

### 2. Distribusi DNS via DHCP

Agar semua perangkat di jaringan otomatis menggunakan Pi-hole sebagai DNS:

- Buka pengaturan **DHCP Server** di Router/MikroTik
- Set **DNS Server** ke IP Pi-hole saja
- Hapus DNS publik (8.8.8.8, 1.1.1.1, dll.) dari konfigurasi DHCP

> Jika DNS publik dibiarkan di DHCP, perangkat client bisa bypass Pi-hole dan langsung query ke sana — iklan tidak tersaring.

---

## Catatan Setup Pribadi: MikroTik

Pada setup ini, MikroTik berperan sebagai **DHCP Server** sekaligus **DoH resolver**. Konfigurasi DNS diarahkan ke Pi-hole melalui pengaturan berikut:

### IP > DHCP Client

Pada interface yang terhubung ke ISP (biasanya `ether1`), buka:

**IP > DHCP Client > [pilih interface] > tab DNS**

- Nonaktifkan opsi **"Use Peer DNS"** agar MikroTik tidak mengambil DNS otomatis dari ISP
- Atau biarkan enabled, tapi pastikan DNS yang didistribusikan ke client tetap mengarah ke Pi-hole (diatur di DHCP Server, bukan DHCP Client)

### IP > DHCP Server > Networks

Buka **IP > DHCP Server > Networks > [pilih network 10.10.x.x]**:

- Set kolom **DNS Servers** ke IP Pi-hole (contoh: `10.10.x.x`)
- Kosongkan atau hapus DNS publik dari kolom ini

Dengan konfigurasi ini, setiap perangkat yang mendapat IP dari MikroTik via DHCP akan otomatis menggunakan Pi-hole sebagai DNS pertama.

### Alur DNS pada Setup Ini

```
Client (dapat IP dari MikroTik DHCP)
  → DNS query ke Pi-hole
    → Jika iklan: diblokir
    → Jika bukan iklan: diteruskan ke MikroTik
      → MikroTik enkripsi via DoH
        → Keluar ke Internet
```

---

## Verifikasi Setup

**Cek DNS aktif di client (Linux):**
```bash
resolvectl status
# atau
nslookup google.com
```
Pastikan server yang muncul adalah IP Pi-hole, bukan IP lain.

**Flush DNS cache jika perubahan belum terasa:**

```bash
# Linux
sudo resolvectl flush-caches

# Windows
ipconfig /flushdns
```

**Browser:** Matikan fitur **Secure DNS / DNS-over-HTTPS** bawaan browser (Chrome, Firefox, dll.) agar browser tidak bypass Pi-hole dan menggunakan DNS sistem.

> Di Chrome: Settings > Privacy and Security > Security > Use secure DNS → **Off**
> Di Firefox: Settings > Network Settings > Enable DNS over HTTPS → **Off**
