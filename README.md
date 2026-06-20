# 🛡️ Wazuh SIEM Home Lab

> Implementasi Wazuh SIEM single-node pribadi — dijalankan di Windows lewat Docker Desktop + WSL2, untuk monitoring keamanan PC sendiri dan eksperimen blue-team/SOC skala home-lab.

![Wazuh](https://img.shields.io/badge/Wazuh-4.14.x-1A93D9?logo=wazuh&logoColor=white)
![Platform](https://img.shields.io/badge/Platform-Windows%20%2B%20WSL2-0078D6?logo=windows)
![Status](https://img.shields.io/badge/Status-In%20Progress-yellow)
![License](https://img.shields.io/badge/License-MIT-green)

---

## 📋 Tentang Project

Repo ini mendokumentasikan implementasi **Wazuh** (open-source SIEM/XDR) sebagai home-lab pribadi. Tujuannya dua arah:

1. **Praktis** — punya visibility keamanan nyata terhadap PC pribadi (FIM, log Windows Event, deteksi anomali).
2. **Pembelajaran** — latihan hands-on operasional SIEM di luar konteks kerja sehari-hari sebagai SOC Analyst, termasuk hardening, tuning rules, dan integrasi alerting.

> ⚠️ **Catatan:** Ini setup home-lab, **bukan** untuk production atau diekspos ke internet publik. Semua kredensial default sudah diganti dan stack hanya diakses dari jaringan lokal (LAN).

---

## 🏗️ Arsitektur

Deployment tipe **single-node all-in-one** — manager, indexer, dan dashboard berjalan dalam satu stack Docker, di-host di WSL2 di atas Windows.

```
┌─────────────────────────────────────────────────────┐
│                  Windows 10/11 (Host)                │
│                                                        │
│  ┌──────────────────────────────────────────────┐    │
│  │         WSL2 + Docker Desktop                  │    │
│  │  ┌────────────┐ ┌──────────┐ ┌────────────┐  │    │
│  │  │   Wazuh    │ │  Wazuh   │ │   Wazuh    │  │    │
│  │  │  Manager   │ │ Indexer  │ │ Dashboard  │  │    │
│  │  └────────────┘ └──────────┘ └────────────┘  │    │
│  └──────────────────────────────────────────────┘    │
│                        ▲                              │
│                        │ 127.0.0.1:1514/1515          │
│                        │                              │
│  ┌──────────────────────────────────────────────┐    │
│  │     Wazuh Agent (native Windows)               │    │
│  │     → memonitor host ini sendiri               │    │
│  └──────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────┘
```

**Kenapa Docker Desktop + WSL2?** Komponen central Wazuh (manager, indexer, dashboard) hanya punya build untuk kernel Linux. WSL2 + Docker Desktop dipilih dibanding VM penuh karena overhead RAM lebih ringan dan official-supported oleh dokumentasi Wazuh, sambil tetap host langsung di Windows (tanpa dual-boot).

---

## 🖥️ Spesifikasi Home-Lab

| Komponen | Spesifikasi |
|---|---|
| CPU | Intel i3-10105F (4C/8T) |
| RAM | 16GB (2x8GB 2666MHz) |
| GPU | RX 6700 XT 12GB *(tidak dipakai oleh Wazuh — pure CPU/RAM/disk I/O)* |
| OS | Windows 10/11 |
| Storage stack Wazuh | SSD M.2 SATA terpisah dari drive OS |

Kebutuhan resmi minimum Wazuh all-in-one (±25 agent, retensi 90 hari): **4 CPU, 8GB RAM, 50GB disk**. Spek di atas memenuhi dengan margin untuk skala 1 PC + beberapa endpoint tambahan.

---

## 🧱 Stack

| Layer | Teknologi |
|---|---|
| SIEM Core | Wazuh 4.14.x (manager + indexer berbasis OpenSearch + dashboard) |
| Container Runtime | Docker Desktop (WSL2 backend) |
| Linux Layer | WSL2 — Ubuntu 22.04 |
| Endpoint Agent | Wazuh Agent native (Windows) |

---

## ✅ Status Implementasi

| # | Tahapan | Status |
|---|---|---|
| 1 | Setup WSL2 + Docker Desktop, deploy stack single-node | ✅ Selesai |
| 2 | Hardening — ganti password default `admin` (indexer/dashboard) | ✅ Selesai |
| 3 | Hardening — ganti password default API `wazuh-wui` | ✅ Selesai |
| 4 | Pasang Wazuh Agent di PC sendiri (monitoring host pertama) | 🔄 Sedang berjalan |
| 5 | Ganti password user internal lain (`kibanaserver`, dll) | ⬜ Belum |
| 6 | Aktivasi modul **File Integrity Monitoring (FIM)** custom path | ⬜ Belum |
| 7 | Aktivasi modul **Vulnerability Detection** | ⬜ Belum |
| 8 | Tambah agent di device lain di jaringan rumah | ⬜ Belum |
| 9 | Rules & tuning custom | ⬜ Belum |
| 10 | Alerting eksternal (Slack/Telegram/email) | ⬜ Belum |

---

## 🔐 Hardening yang Sudah Diterapkan

- [x] Password default `admin` (indexer + dashboard) diganti
- [x] Password default API user `wazuh-wui` diganti
- [x] Stack **tidak** diekspos ke internet — akses LAN-only, tidak ada port forwarding router
- [x] Port `9200` (indexer) tidak dibuka ke jaringan luar, hanya internal antar-container
- [ ] User internal lain (`kibanaserver`, `kibanaro`, dll) masih default — dijadwalkan
- [ ] TLS/reverse-proxy tambahan jika nanti ada kebutuhan akses remote

---

## 🚀 Cara Replikasi (Ringkas)

> Detail lengkap step-by-step (termasuk konfigurasi `.wslconfig`, JVM heap indexer, dan prosedur ganti password) ada di [`docs/panduan-implementasi-wazuh.md`](./docs/panduan-implementasi-wazuh.md).

1. Install WSL2 + Ubuntu 22.04, set resource limit via `.wslconfig` (termasuk `vm.max_map_count=262144` — wajib untuk indexer).
2. Install Docker Desktop dengan WSL2 backend, aktifkan WSL integration.
3. Clone `wazuh/wazuh-docker` versi 4.14.x, generate sertifikat indexer.
4. Sesuaikan JVM heap indexer sesuai RAM yang dialokasikan ke WSL2.
5. `docker compose up -d`, tunggu semua service `running`.
6. Akses dashboard di `https://localhost`, **segera ganti kredensial default**.
7. Pasang Wazuh Agent native di endpoint yang ingin dimonitor.

---

## 🗺️ Roadmap Berikutnya

- [ ] Custom FIM path untuk folder kerja & project security tools pribadi
- [ ] Enable Vulnerability Detection module di sisi manager
- [ ] Tambah agent untuk device lain di jaringan rumah
- [ ] Custom detection rules berbasis use case pribadi
- [ ] Integrasi alerting ke Slack/Telegram untuk alert kritis
- [ ] Evaluasi migrasi ke Wazuh 5.0 setelah versi tersebut matang (di environment terpisah dulu)

---

## ⚠️ Disclaimer

Repo ini murni untuk dokumentasi home-lab pribadi dan tujuan edukasi. Tidak ada data klien, kredensial asli, atau informasi sensitif apa pun yang disertakan — semua contoh password/IP di dokumentasi pendukung adalah placeholder atau nilai default resmi dari dokumentasi Wazuh.

---

## 📝 Lisensi

[MIT](./LICENSE) — silakan dimodifikasi/dipakai ulang untuk keperluan home-lab sendiri.

---

<p align="center">
  <sub>Built & maintained by <a href="https://github.com/p3nr0s3">@p3nr0s3</a></sub>
</p>
