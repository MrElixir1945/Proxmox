# Proxmox VE Post-Installation Setup Guide

Dokumentasi ini menjelaskan langkah-langkah esensial konfigurasi Proxmox VE setelah instalasi. Tujuan utamanya adalah beralih dari repository Enterprise default ke repository No-Subscription yang didukung komunitas, guna mengaktifkan pembaruan sistem.

---

## 1. Informasi Lingkungan

| Komponen | Detail |
|---|---|
| **Operating System** | Proxmox VE 9.x |
| **Debian Base** | Trixie (Testing/Unstable) |
| **Target** | Homelab / Development Environment |

---

## 2. Konfigurasi Repository

Secara default, Proxmox VE dikonfigurasi dengan repository Enterprise yang memerlukan lisensi berbayar. Untuk mengaktifkan pembaruan tanpa berlangganan, sumber repository harus dimodifikasi.

### 2.1. Perbarui Main Sources List

Edit file `/etc/apt/sources.list` dan pastikan isinya adalah sebagai berikut:

```
deb http://deb.debian.org/debian trixie main contrib non-free-firmware
deb http://deb.debian.org/debian trixie-updates main contrib non-free-firmware
deb http://security.debian.org/debian-security trixie-security main contrib non-free-firmware

# Proxmox VE No-Subscription Repository
deb http://download.proxmox.com/debian/pve trixie pve-no-subscription

# Ceph No-Subscription Repository
deb http://download.proxmox.com/debian/ceph-squid trixie no-subscription
```

### 2.2. Hapus Enterprise Sources yang Tidak Diperlukan

Untuk mencegah error `401 Unauthorized` saat proses pembaruan, hapus file sumber enterprise default yang sering menyebabkan konflik:

```bash
rm /etc/apt/sources.list.d/pve-enterprise.list
rm /etc/apt/sources.list.d/pve-enterprise.sources
rm /etc/apt/sources.list.d/ceph.sources
rm /etc/apt/sources.list.d/debian.sources
```

---

## 3. Update dan Upgrade Sistem

Setelah repository dikonfigurasi, jalankan perintah berikut untuk menyegarkan database paket dan memperbarui komponen sistem:

```bash
apt update && apt dist-upgrade -y
```

> **Catatan:** `dist-upgrade` lebih disarankan dibanding `upgrade` karena menangani semua dependensi dan pembaruan kernel dengan lebih baik.
