# Panduan Implementasi Wazuh All-in-One di Windows (Docker Desktop + WSL2)

**Target PC:** Intel i3-10105F (4C/8T) · 16GB RAM (2x8GB 2666MHz) · RX 6700 XT 12GB
**OS:** Windows 10/11 (host langsung)
**Tipe deployment:** Single-node all-in-one (manager + indexer + dashboard dalam satu stack)
**Tanggal panduan dibuat:** 20 Juni 2026 — versi stabil terbaru saat ini **Wazuh 4.14.x**

---

## 1. Kenapa Docker Desktop + WSL2?

Komponen central Wazuh (server, indexer, dashboard) **hanya jalan di kernel Linux** — tidak ada build native untuk Windows. Karena kamu mau host langsung di Windows (bukan dual-boot/VM terpisah), tiga opsi yang ada:

| Opsi | Overhead RAM | Kemudahan maintenance | Direkomendasikan? |
|---|---|---|---|
| Docker Desktop + WSL2 backend | Sedang | Tinggi (docker compose, official support) | ✅ **Ya** |
| WSL2 + install paket Linux manual (apt) | Rendah | Rendah (upgrade/rollback manual) | Alternatif jika RAM sangat mepet |
| VM penuh (VirtualBox/Hyper-V) | Tinggi | Sedang | Tidak perlu, WSL2 lebih ringan |

Dengan 16GB RAM, Docker Desktop + WSL2 adalah titik tengah terbaik: didukung resmi oleh dokumentasi Wazuh, mudah di-upgrade, dan overhead-nya masih wajar untuk skala home-lab/single PC.

### Cek kesesuaian spek
Kebutuhan resmi minimum Wazuh all-in-one (untuk ±25 agent, retensi 90 hari): **4 CPU, 8GB RAM, 50GB disk**. PC kamu (4 core/8 thread, 16GB RAM) memenuhi ini dengan margin — cukup untuk monitoring PC ini sendiri plus beberapa agent tambahan di jaringan rumah. RX 6700 XT tidak dipakai sama sekali oleh Wazuh, jadi abaikan saja.

**Batasan realistis:** dengan 16GB RAM total (dipakai bersama Windows + Docker Desktop), jangan jalankan beban indexing berat (ratusan EPS atau ratusan agent) — itu butuh tier server terpisah. Untuk monitoring 1 PC + beberapa endpoint tambahan, ini lebih dari cukup.

---

## 2. Prasyarat

- Windows 10 versi 2004+ (build 19041+) atau Windows 11
- Virtualization (Intel VT-x) aktif di BIOS — i3-10105F mendukung ini, cek di Task Manager → tab Performance → CPU → "Virtualization: Enabled". Jika belum, masuk BIOS dan aktifkan `Intel Virtualization Technology`.
- Ruang disk kosong minimal 50GB (disarankan di drive SSD/NVMe, bukan HDD)
- Koneksi internet stabil (pull image Docker totalnya beberapa GB)
- Hak akses Administrator di Windows

---

## 3. Langkah 1 — Install & Konfigurasi WSL2

Buka **PowerShell sebagai Administrator**:

```powershell
wsl --install -d Ubuntu-22.04
```

Restart PC jika diminta. Setelah restart, Ubuntu akan minta setup username/password Linux — buat saja, ini terpisah dari akun Windows.

Pastikan WSL pakai versi 2 sebagai default:

```powershell
wsl --set-default-version 2
wsl --status
```

> **Catatan lokasi drive:** secara default, distro Ubuntu ini ter-install di drive **C:**. Ukurannya kecil (cuma OS dasar + nanti repo `wazuh-docker` + sertifikat, total beberapa GB) — data Wazuh yang besar (indexer, dll) sebenarnya tersimpan terpisah lewat *Docker Desktop disk image*, yang akan kita arahkan ke SSD 128GB (drive D:) di **Langkah 2** dan **Section 14**. Tapi kalau kamu mau distro Ubuntu-nya juga 100% pindah dari C: (bukan cuma data Docker), lebih efisien lakukan itu **sekarang, sebelum lanjut ke Langkah 3 (clone repo)** — ikuti perintah export/import di Section 14 langkah 3, baru lanjut ke langkah berikutnya di bawah ini.

### Batasi resource WSL2 agar tidak "memakan" semua RAM Windows

Buat/edit file `C:\Users\<USERNAME>\.wslconfig` (ganti `<USERNAME>`):

```ini
[wsl2]
memory=11GB
processors=6
swap=4GB
kernelCommandLine = "sysctl.vm.max_map_count=262144"
```

**Kenapa setting ini:**
- `memory=11GB` — menyisakan ±5GB untuk Windows + Docker Desktop UI + aplikasi lain. Dengan 16GB total, ini batas aman supaya Windows tidak ikut nge-lag.
- `processors=6` — sisakan 2 thread untuk Windows, dari total 8 thread i3-10105F.
- `kernelCommandLine` — ini **wajib**, karena Wazuh indexer (berbasis OpenSearch) butuh `vm.max_map_count ≥ 262144` untuk membuat banyak memory-mapped area. Tanpa ini, container indexer akan gagal start dengan error `max virtual memory areas vm.max_map_count too low`.

Restart WSL agar setting berlaku:

```powershell
wsl --shutdown
```

---

## 4. Langkah 2 — Install Docker Desktop

1. Download dari https://www.docker.com/products/docker-desktop/
2. Saat instalasi, **centang "Use WSL 2 instead of Hyper-V"**
3. Setelah install, buka Docker Desktop → **Settings → Resources → WSL Integration** → aktifkan toggle untuk distro `Ubuntu-22.04`
4. Verifikasi dari dalam WSL (buka terminal Ubuntu):

```bash
docker --version
docker compose version
```

Keduanya harus menampilkan versi tanpa error.

---

## 5. Langkah 3 — Clone Repo & Generate Sertifikat

Semua langkah berikut dijalankan **di dalam terminal WSL (Ubuntu)**, bukan PowerShell — supaya performa I/O Docker lebih optimal (Docker Desktop WSL2 backend jauh lebih cepat mengakses filesystem Linux native).

```bash
cd ~
git clone https://github.com/wazuh/wazuh-docker.git -b v4.14.5
cd wazuh-docker/single-node
```

> Sebelum clone, cek dulu apakah ada versi 4.14.x yang lebih baru di https://github.com/wazuh/wazuh-docker/releases — Wazuh 5.0 kemungkinan rilis sekitar akhir Juni/Juli 2026, tapi untuk stabilitas di tahap awal implementasi, **tetap pakai jalur 4.14.x** dulu sampai 5.0 matang.

Generate sertifikat untuk komunikasi aman antar-komponen:

```bash
docker compose -f generate-indexer-certs.yml run --rm generator
```

Tunggu sampai selesai (akan tersimpan otomatis di `config/wazuh_indexer_ssl_certs/`).

---

## 6. Langkah 4 — Sesuaikan Resource Indexer (penting untuk RAM 16GB)

Buka `single-node/docker-compose.yml` dan cari service `wazuh.indexer`. Default biasanya pakai heap kecil, tapi pastikan environment variable JVM heap-nya wajar untuk RAM yang kamu alokasikan ke WSL2 (11GB):

```bash
nano single-node/docker-compose.yml
```

Cari bagian `wazuh.indexer` → `environment`, pastikan ada baris seperti ini (sesuaikan jika belum ada):

```yaml
environment:
  - "OPENSEARCH_JAVA_OPTS=-Xms2g -Xmx2g"
```

**Aturan:** heap JVM idealnya setengah dari RAM yang dialokasikan ke WSL2, tapi jangan lebih dari ~31GB (limit JVM compressed pointers — tidak relevan untuk kasusmu). Dengan WSL2 dibatasi 11GB, alokasi 2GB untuk indexer heap sudah cukup untuk skala home-lab dan menyisakan ruang untuk manager + dashboard + OS overhead container.

Simpan file (`Ctrl+O`, `Enter`, `Ctrl+X` di nano).

---

## 7. Langkah 5 — Jalankan Stack

```bash
docker compose up -d
```

Proses awal butuh 1–3 menit. Cek status:

```bash
docker compose ps
```

Kamu akan lihat log seperti `Failed to connect to Wazuh indexer port 9200` atau `Wazuh dashboard server is not ready yet` saat startup — **ini normal**, tunggu sampai semua service status `running`.

---

## 8. Langkah 6 — Akses Dashboard & Ganti Password Default

Buka browser di Windows (host), akses:

```
https://localhost
```

(Docker Desktop otomatis forward port container ke `localhost` Windows, jadi tidak perlu konfigurasi tambahan.)

Login default:
- **Username:** `admin`
- **Password:** `SecretPassword`

**Segera ganti password ini** — dokumentasi resmi Wazuh secara eksplisit merekomendasikan mengganti credential default setelah instalasi pertama, lewat menu Security di dashboard atau API.

---

## 9. Langkah 7 — Buka Port di Firewall Windows (jika mau monitor PC/device lain)

Jika nanti kamu mau pasang Wazuh agent di laptop/PC lain di jaringan rumah, buka port berikut di **Windows Defender Firewall → Inbound Rules**:

| Port | Fungsi |
|---|---|
| 443/tcp | Akses dashboard via browser |
| 1514/tcp | Komunikasi data agent → manager |
| 1515/tcp | Enrollment/registrasi agent baru |
| 55000/tcp | Wazuh REST API (opsional) |

Port 9200 (indexer) **sebaiknya tidak** dibuka ke jaringan luar — cukup diakses internal oleh manager dan dashboard saja.

---

## 10. Langkah 8 — Monitor PC Windows Ini Sendiri (Opsional)

Karena PC ini hosting manager-nya di WSL2/Docker, kamu juga bisa pasang Wazuh **agent native Windows** di PC yang sama untuk memonitor dirinya sendiri (file integrity, log event Windows, dll):

1. Download installer `.msi` dari https://documentation.wazuh.com (cari "Wazuh agent Windows package" untuk versi 4.14.x)
2. Install via PowerShell (Administrator):

```powershell
msiexec.exe /i wazuh-agent-4.14.5-1.msi /q WAZUH_MANAGER='127.0.0.1' WAZUH_AGENT_NAME='PC-Windows-Utama'
```

3. Start service:

```powershell
NET START WazuhSvc
```

4. Cek di dashboard → menu **Agents**, agent baru akan muncul setelah enrollment berhasil (biasanya <1 menit).

---

## 11. Maintenance & Update

**Stop stack** (tanpa hapus data):
```bash
docker compose down
```

**Start lagi:**
```bash
docker compose up -d
```

**Update ke versi minor baru** (misal 4.14.5 → 4.14.x berikutnya): ikuti panduan upgrade resmi di https://documentation.wazuh.com/current/deployment-options/docker/upgrading-wazuh-docker.html — intinya ganti tag image di `docker-compose.yml`, regenerate sertifikat jika diminta, lalu `docker compose up -d` ulang.

**Backup data:** volume Docker (`single-node_wazuh-indexer-data`, dll) menyimpan semua data — backup dengan `docker run --rm -v <volume_name>:/data -v $(pwd):/backup busybox tar czf /backup/backup.tar.gz /data`.

---

## 12. Troubleshooting Cepat

| Gejala | Penyebab umum | Solusi |
|---|---|---|
| Container indexer restart terus / error `max virtual memory areas` | `vm.max_map_count` belum ke-set | Cek `.wslconfig`, jalankan `wsl --shutdown` lalu buka ulang WSL, verifikasi dengan `sysctl vm.max_map_count` di dalam WSL (harus ≥262144) |
| Dashboard tidak bisa diakses di `https://localhost` | Stack belum fully up, atau port bentrok | `docker compose ps` cek status, pastikan tidak ada aplikasi lain pakai port 443 |
| Docker Desktop terasa berat / Windows lag | Alokasi memory WSL2 kebesaran | Turunkan `memory=` di `.wslconfig`, jangan lebih dari ~70% RAM fisik |
| Agent tidak muncul di dashboard setelah install | Firewall blokir port 1514/1515, atau salah `WAZUH_MANAGER` IP | Cek firewall rule, pastikan IP/hostname manager benar dan reachable |

---

## 13. Langkah Lanjutan (Opsional)

- Tambah agent di PC/laptop lain di jaringan rumah (proses sama seperti langkah 10, tapi `WAZUH_MANAGER` diisi IP PC ini, bukan `127.0.0.1`)
- Integrasi notifikasi (Slack/Telegram/email) untuk alert kritis
- Aktifkan modul **File Integrity Monitoring (FIM)** dan **Vulnerability Detection** di `wazuh_manager.conf`
- Kalau nanti mau eksperimen Wazuh 5.0, lakukan di environment terpisah dulu sebelum migrasi stack utama

---

## 14. Drive Mana yang Sebaiknya Dipakai?

Setup ini mengasumsikan 4 drive internal yang terpasang di PC:

| Drive | Tipe | Cocok untuk Wazuh? |
|---|---|---|
| 256GB SSD M.2 SATA (OS) | SSD | Bisa, tapi sebaiknya jangan — OS makin padat dan ikut bottleneck saat indexer aktif |
| 1TB HDD 7200 2.5" (data) | HDD | ❌ Tidak disarankan — indexer butuh random I/O cepat, HDD jadi bottleneck berat saat startup/indexing |
| 3TB HDD 7200 3.5" (data) | HDD | ❌ Tidak disarankan — alasan sama, HDD |
| 128GB SSD M.2 SATA (kosong) | SSD | ✅ **Pilihan terbaik** — kosong, SSD, lebih dari cukup untuk kebutuhan minimum |

**Rekomendasi: pakai drive #4 (128GB SSD M.2 SATA yang kosong).** Karena ini drive internal tetap (bukan removable via USB), kekhawatiran "jangan dicabut saat jalan" tidak relevan di sini — drive ini selalu terpasang seperti drive OS.

### Langkah setup

**1. Pastikan drive punya drive letter & sudah diformat NTFS**

- Buka `Disk Management` (klik kanan Start → *Disk Management*, atau jalankan `diskmgmt.msc`)
- Cari drive 128GB — kalau belum pernah dipakai biasanya tampil sebagai *"Unallocated"*
- Klik kanan → **New Simple Volume** → format **NTFS** → assign drive letter (misal `D:`)

**2. Arahkan Docker Desktop disk image ke drive ini**

- Quit Docker Desktop sepenuhnya (klik kanan ikon tray → **Quit Docker Desktop**)
- Settings → **Resources → Advanced** → kolom **"Disk image location"** → Browse → pilih `D:\DockerData`
- **Apply & restart** — Docker Desktop otomatis memindahkan VHDX ke lokasi baru

Setelah ini, semua volume Docker (termasuk data indexer Wazuh) tersimpan di SSD 128GB, terpisah dari drive OS.

**3. (Opsional) Pindahkan juga distro WSL Ubuntu ke drive ini**, supaya repo `wazuh-docker` dan sertifikat juga ikut di sana:

```powershell
wsl --shutdown
wsl --export Ubuntu-22.04 D:\WSL-Backup\ubuntu-export.tar
wsl --unregister Ubuntu-22.04
wsl --import Ubuntu-22.04 D:\WSL\Ubuntu-22.04 D:\WSL-Backup\ubuntu-export.tar --version 2
wsl -d Ubuntu-22.04 -u root sh -c "echo -e '[user]\ndefault=<username>' >> /etc/wsl.conf"
wsl --shutdown
```

### Soal kapasitas 128GB

Kebutuhan minimum resmi Wazuh untuk skala home-lab (±25 agent, retensi 90 hari) adalah 50GB — 128GB memberi ruang cukup lega untuk mulai. Tetap pantau pemakaian disk dari waktu ke waktu (`docker system df` di WSL); kalau nanti agent bertambah banyak atau volume alert membesar, pertimbangkan turunkan retensi index atau pindahkan index lama ke HDD sebagai cold storage (fitur index lifecycle management — di luar scope dasar panduan ini).

### Soal dua HDD (1TB & 3TB)

Tetap berguna untuk:
- **Backup berkala** data Wazuh (lihat perintah `docker run ... tar czf` di bagian Maintenance) — backup file besar yang jarang diakses cocok di HDD
- **Cold storage** index lama kalau nanti retensi makin besar

Tapi **jangan jadikan lokasi utama** indexer aktif di kedua HDD ini — performa write/random-read OpenSearch akan terasa jauh lebih lambat dibanding di SSD 128GB.

---

## 15. Hardening Keamanan Dasar — Ganti Password Default

Ada **dua password terpisah** yang perlu diganti — sering tertukar karena namanya mirip:

| Password | Dipakai untuk | Default |
|---|---|---|
| `admin` (indexer + dashboard) | Login ke dashboard, akses penuh ke data indexer | `admin` / `SecretPassword` |
| API user `wazuh-wui` | Komunikasi internal dashboard ↔ Wazuh manager REST API | `wazuh-wui` / `MyS3cr37P450r.*-` |

Jalankan semua perintah berikut dari dalam WSL, di folder `~/wazuh-docker/single-node`.

### Bagian A — Ganti password `admin` (paling kritis)

**1. Generate hash untuk password baru:**

```bash
docker run --rm -ti wazuh/wazuh-indexer:4.14.5 bash /usr/share/wazuh-indexer/plugins/opensearch-security/tools/hash.sh -p 'PASSWORD_BARU_KAMU'
```

Pakai password kuat: minimal 8 karakter, kombinasi huruf besar/kecil, angka, dan simbol — kalau tidak memenuhi syarat, OpenSearch akan reject saat apply nanti. Output-nya berupa hash bcrypt panjang (`$2y$12$...`) — copy seluruhnya.

**2. Edit file `internal_users.yml`:**

```bash
nano config/wazuh_indexer/internal_users.yml
```

Cari section `admin:`, ganti nilai `hash:` dengan hash baru dari langkah 1 (timpa hash lama). Simpan (`Ctrl+O`, `Enter`, `Ctrl+X`).

**3. Cek nama container indexer yang aktif:**

```bash
docker compose ps
```

Catat nama persisnya (biasanya `single-node-wazuh.indexer-1`).

**4. Apply perubahan ke indexer yang sedang jalan:**

```bash
docker exec -it single-node-wazuh.indexer-1 bash /usr/share/wazuh-indexer/plugins/opensearch-security/tools/securityadmin.sh \
  -cd /usr/share/wazuh-indexer/opensearch-security/ \
  -nhnv \
  -cacert /usr/share/wazuh-indexer/certs/root-ca.pem \
  -cert /usr/share/wazuh-indexer/certs/admin.pem \
  -key /usr/share/wazuh-indexer/certs/admin-key.pem \
  -p 9200 -icl
```

Harus berakhir dengan pesan sukses (`Done with success`).

**5. Update referensi password di `docker-compose.yml`:**

```bash
nano docker-compose.yml
```

Cari semua baris `INDEXER_PASSWORD=SecretPassword` (muncul di service `wazuh.manager` dan `wazuh.dashboard`), ganti `SecretPassword` jadi password baru kamu di **kedua tempat**.

**6. Restart container yang terdampak:**

```bash
docker compose restart wazuh.manager wazuh.dashboard
```

**7. Login ke `https://localhost`** pakai `admin` + password baru — password lama otomatis tidak berlaku lagi.

### Bagian B — Ganti password API `wazuh-wui`

Password ini dipakai dashboard untuk autentikasi ke REST API manager (port 55000) — beda sistem dari indexer, jadi prosesnya beda juga.

**1. Dapatkan auth token pakai credential lama:**

```bash
TOKEN=$(curl -s -u wazuh-wui:MyS3cr37P450r.*- -k -X POST "https://localhost:55000/security/user/authenticate?raw=true")
```

**2. Cari ID user `wazuh-wui`:**

```bash
curl -s -k -X GET "https://localhost:55000/security/users?username=wazuh-wui" -H "Authorization: Bearer $TOKEN" | python3 -m json.tool
```

Catat nilai `id` dari hasilnya (biasanya `2`).

**3. Set password baru** (ganti `<id>` sesuai hasil langkah 2):

```bash
curl -s -k -X PUT "https://localhost:55000/security/users/<id>" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"password":"PASSWORD_API_BARU"}'
```

**4. Update `docker-compose.yml`** — cari baris `API_PASSWORD=MyS3cr37P450r.*-` di service `wazuh.dashboard`, ganti dengan password baru dari langkah 3.

**5. Restart dashboard:**

```bash
docker compose restart wazuh.dashboard
```

### Catatan tambahan

- Simpan kedua password baru ini di password manager — kalau hilang, recovery butuh ulangi proses generate hash dari awal.
- Container `wazuh.indexer` dan `wazuh.manager` punya beberapa user internal lain (`kibanaserver`, `kibanaro`, dll di `internal_users.yml`) yang masih default — untuk home-lab skala kecil biasanya tidak kritis selama dashboard tidak diekspos ke internet publik, tapi prosesnya sama persis seperti Bagian A kalau suatu saat mau diganti semua.
- Jangan expose port 443/9200/55000 ke internet (port forwarding router) tanpa VPN/reverse-proxy tambahan — setup ini didesain untuk jaringan rumah (LAN), bukan akses publik.

## 16. Roadmap Lanjutan

Status: hardening dasar (Bagian A admin + Bagian B API `wazuh-wui`) **selesai**. Urutan berikutnya:

| # | Topik | Status |
|---|---|---|
| 1 | Tambah agent di device lain | ⬜ Belum |
| 2 | Ganti password user internal lain (`kibanaserver`, dll) | ⬜ Belum |
| 3 | Belajar dashboard — FIM & vulnerability detection | ⬜ Belum |
| 4 | Rules & tuning rules | ⬜ Belum |
| 5 | Alerting eksternal (Slack/Telegram/email) | ⬜ Belum |

Urutan ini sengaja: agent lain dulu (cepat, langsung nambah value), baru cleanup security, baru eksplorasi dashboard, baru tuning rules (butuh data dari beberapa agent supaya ada bahan), terakhir alerting (supaya notifikasi yang dikirim sudah relevan, bukan alert mentah yang noisy).

---

**Catatan akhir:** RX 6700 XT di spek kamu tidak punya peran apa pun dalam stack ini — Wazuh murni CPU + RAM + disk I/O. Jangan khawatir soal driver GPU atau passthrough.
