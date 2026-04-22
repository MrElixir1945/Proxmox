# Proxmox VE Post-Installation Setup Guide

This documentation provides the essential steps required to configure Proxmox VE immediately after installation. The primary objective is to transition from the default Enterprise repository to the community-supported No-Subscription repository to enable system updates.

## 1. Environment Information
- **Operating System:** Proxmox VE 9.x
- **Debian Base:** Trixie (Testing/Unstable)
- **Target:** Homelab / Development Environment

## 2. Repository Configuration

By default, Proxmox VE is configured with Enterprise repositories which require a valid subscription. To enable updates without a subscription, the repository sources must be modified.

### 2.1. Update Main Sources List
Edit the file `/etc/apt/sources.list` and ensure it contains the following repositories:

```bash
deb [http://deb.debian.org/debian](http://deb.debian.org/debian) trixie main contrib non-free-firmware
deb [http://deb.debian.org/debian](http://deb.debian.org/debian) trixie-updates main contrib non-free-firmware
deb [http://security.debian.org/debian-security](http://security.debian.org/debian-security) trixie-security main contrib non-free-firmware

# Proxmox VE No-Subscription Repository
deb [http://download.proxmox.com/debian/pve](http://download.proxmox.com/debian/pve) trixie pve-no-subscription

# Ceph No-Subscription Repository
deb [http://download.proxmox.com/debian/ceph-squid](http://download.proxmox.com/debian/ceph-squid) trixie no-subscription
2.2. Remove Misconfigured Enterprise Sources
To prevent 401 Unauthorized errors during the update process, remove the default enterprise source files:

Bash
rm /etc/apt/sources.list.d/pve-enterprise.list
rm /etc/apt/sources.list.d/pve-enterprise.sources
rm /etc/apt/sources.list.d/ceph.sources
rm /etc/apt/sources.list.d/debian.sources
3. System Update and Upgrade
After configuring the repositories, execute the following commands to refresh the package database and upgrade the system components:

Bash
apt update
apt dist-upgrade -y
Note: dist-upgrade is preferred over upgrade to ensure all dependencies and kernel updates are handled correctly.
