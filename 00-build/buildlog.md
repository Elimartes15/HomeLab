# Build Log
 
Chronological record of the lab build: what was done, why, and what broke.
 
---
 
## 2026-10-17 · Planning
 
- Designed the lab architecture: single Proxmox hypervisor (Dell G5), OPNsense as the only router between six lab VLANs, Wazuh and Kali on dedicated mini PCs, and the old Cisco 2950 as the core switch.
- Sized the hypervisor against its 32 GB RAM limit. Moving Wazuh and Kali off the G5 frees about 12 GB, so every lab VM can run at once (16 GB of VMs plus host overhead).
- Chose OS versions and sources.
---
 
## 2026-10-19 · Wazuh on mini PC 1
 
> **Superseded on 2026-10-20:** Wazuh was replaced by Elastic Security. Kept here as a record of the evaluation. See [why Elastic over Wazuh](../docs/2026-10-20-why-elastic-over-wazuh.md).
 
**OS install**
 
- Ubuntu Server LTS, **standard install** (not minimized). Minimized strips editors and troubleshooting tools I'd reinstall while debugging, and saves only about 100 MB of RAM.
- OpenSSH server enabled during install, so the box is managed over SSH.
- Network: DHCP on the home LAN, planned to move to `10.10.60.10` on VLAN 60.
- Disk: `<disk size>`, LVM, root logical volume `<expanded to full disk: yes/no>`.
- Fully updated after install:
```bash
sudo apt update && sudo apt full-upgrade -y
sudo reboot
```
 
**Wazuh install**
 
- Wazuh `4.14`, all-in-one (manager, indexer, dashboard) using the official quickstart installation assistant.
- Admin password saved to my password manager.
- No agents were enrolled.
**Indexer JVM heap: 2 GB**
 
The default guidance is half of RAM, but that assumes the indexer has the machine to itself. On an 8 GB all-in-one, the RAM is shared:
 
| Component | RAM |
| --- | --- |
| Ubuntu | ~0.5 GB |
| Wazuh manager | ~0.5–1 GB |
| Dashboard | ~0.5–1 GB |
| Indexer heap | 2 GB |
| OS file cache (used by the indexer for search) | ~3–4 GB |
 
1 GB risks circuit-breaker errors once all agents are reporting; 4 GB would starve the manager and dashboard. Set in `/etc/wazuh-indexer/jvm.options`:
 
```
-Xms2g
-Xmx2g
```
 
**Proxmox installer USB**
 
- Reused the Ubuntu USB. Rufus overwrites the whole drive, so no manual wipe was needed.
- Proxmox VE `9.2.2` ISO from proxmox.com. SHA256 `4e88fe416df9b527624a175f24c9aa07c714d3332afb1ee3dbf3879573ef2c6c` matched the published value.
- Rufus settings: GPT, UEFI, **DD Image mode**. The Proxmox installer doesn't boot reliably from Rufus's default ISO mode.
- Dismissed Windows' "You need to format the disk" prompt after writing.
---
 
## 2026-10-19 · Proxmox VE on the Dell G5
 
**BIOS (F2)**
 
- Storage mode set to **AHCI**. Dell ships in RAID/Intel RST mode, which hides the SSDs from the Linux installer.
- **Intel Virtualization Technology (VT-x)** and **VT for Direct I/O (VT-d)** enabled.
- Booted the USB from the F12 one-time boot menu.
**Install**
 
- Proxmox VE `<version>`, graphical installer.
- Target: disk 1, **ext4**. Disk 2 is left for a separate LVM-thin pool for lab VMs.
- Hostname `pve01.lab.local`.
- Management IP: static `10.0.0.90/24`, set manually in place of the DHCP address the installer pre-filled.
- Installer USB removed after the install and kept as a rescue disk.
**First-boot issue**
 
- The console showed repeated `EXT4-fs error ... Detected aborted journal`, and the console login then hung.
- Meaning: the kernel hit write errors, set the root filesystem read-only, then lost access to the disk.
- A hard reboot came back clean.
- Suspected cause: NVMe power-state (APST) dropout, a known Linux issue on some SSDs. A failing or loosely seated drive hasn't been ruled out yet.
Follow-up checks:
 
```bash
mount | grep " / "                                          # root must be rw
dmesg | grep -iE "ext4|nvme|ata|i/o error|reset|timeout"    # no errors this boot
lsblk -o NAME,SIZE,MODEL,TRAN,MOUNTPOINT                    # confirm NVMe vs SATA
smartctl -a /dev/nvme0                                      # Critical Warning 0x00, Media Errors 0
```
 
If NVMe, disable deep power states in `/etc/default/grub`, then run `update-grub` and reboot:
 
```
GRUB_CMDLINE_LINUX_DEFAULT="quiet nvme_core.default_ps_max_latency_us=0 pcie_aspm=off"
```
 
Result: everything showed normal after reboot.
 
---
 
## 2026-10-20 · SIEM decision: Elastic Security replaces Wazuh
 
**What changed**
 
| Area | Before | Now |
| --- | --- | --- |
| SIEM | Wazuh all-in-one on mini PC 1 | Elasticsearch + Kibana + Fleet Server, VM on the G5, VLAN 60, 10 GB RAM |
| Endpoint agent | Wazuh agent | Elastic Agent (Windows, Sysmon, PowerShell integrations; Elastic Defend optional) |
| Network logs | Syslog to Wazuh | Elastic Agent OPNsense and Cisco integrations |
| Sigma | Hand-translated to Wazuh XML | `sigma convert` with the Elasticsearch backend → Kibana detection rules via CI |
| YARA | Wazuh FIM + active response | Rules CI-tested in the repo; scheduled scan writes matches to the event log, Elastic Agent ships them |
| Mini PC 1 | Wazuh host | Freed: second DMZ Linux target or spare |
 
**Why:** there is no official Sigma backend for Wazuh's rule format, so every detection would have been translated by hand. Elastic has a maintained pySigma backend, a free Basic tier with a full detection engine, and a free EDR. Full reasoning in [why Elastic over Wazuh](../docs/2026-10-20-why-elastic-over-wazuh.md).
 
**Alternatives evaluated:** OpenSearch Security Analytics (runner-up), Security Onion, Splunk Free, Graylog Open, LimaCharlie, Grafana Loki.
 
**G5 RAM budget after the change**
 
| VM | RAM |
| --- | --- |
| OPNsense | 2 GB |
| DC01 | 4 GB |
| WS01 | 4 GB |
| WS02 | 4 GB |
| Ubuntu DMZ web | 2 GB |
| Elastic SIEM | 10 GB |
| **Total** | **26 GB** |
| Headroom for Proxmox host (of 32 GB) | 6 GB |
 
**Next**
 
- No-subscription repo and updates, wipe disk 2 and create the `vm-lab` LVM-thin pool, ISO downloads (add Ubuntu Server for the Elastic VM).
- switch config and VLAN-aware bridge, moving management to VLAN 100 in staged steps.
- Mini PC 1: reimage once its new role is decided.
---
 
## Issues and fixes
 
| Date | Issue | Cause | Fix |
| --- | --- | --- | --- |
| 2026-10-19 | `EXT4-fs error: Detected aborted journal` on first boot; root went read-only and console login hung | Suspected NVMe APST power-state dropout | Hard reboot |
 
