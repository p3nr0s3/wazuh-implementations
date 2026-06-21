# Panduan Implementasi Wazuh All-in-One di Windows (Docker Desktop + WSL2)

**Skala target:** Home-lab / single-host — 1-10 agent, retensi 30-90 hari
**Contoh implementasi:** Intel i3-10105F (4C/8T) · 16GB RAM (2x8GB 2666MHz) · RX 6700 XT 12GB *(GPU tidak relevan — Wazuh murni CPU/RAM/disk I/O)*
**OS:** Windows 10/11 (host langsung)
**Tipe deployment:** Single-node all-in-one (manager + indexer + dashboard dalam satu stack)
**Versi Wazuh:** 4.14.5
**Update terakhir:** 21 Juni 2026

---

## Cakupan Panduan Ini

Setup dasar (WSL2, Docker Desktop, stack Wazuh, hardening password) sampai ke implementasi lanjutan: agent monitoring PC sendiri, custom FIM, Vulnerability Detection, Sysmon, validasi deteksi pakai Atomic Red Team, Active Response, Threat Intel Enrichment, dan Index Lifecycle Management.

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

Daripada patokan ke satu model hardware tertentu, lebih akurat menyesuaikan ke **skala kebutuhan** — jumlah agent, retensi, dan volume event per detik (EPS):

| Skala | CPU | RAM | Disk | Cocok untuk |
|---|---|---|---|---|
| Minimal / Home-lab | 4 core | 8GB | 50GB | 1-10 agent, retensi 30-90 hari, EPS rendah |
| Small office | 8 core | 16GB | 100-200GB | 10-50 agent |
| Medium | 16 core+ | 32GB+ | 500GB+ | 50-200 agent |

Kebutuhan resmi minimum Wazuh all-in-one untuk skala "Minimal/Home-lab" di atas: **4 CPU, 8GB RAM, 50GB disk** (±25 agent, retensi 90 hari). Contoh implementasi di panduan ini (i3-10105F, 4 core/8 thread, 16GB RAM) ada di kelas "Small office" — lebih dari cukup untuk monitoring 1 PC plus beberapa endpoint tambahan, dengan margin aman.

**Batasan realistis:** dengan 16GB RAM total (dipakai bersama Windows + Docker Desktop), jangan jalankan beban indexing berat (ratusan EPS atau ratusan agent) — itu butuh tier server terpisah.

---

## 2. Prasyarat

- Windows 10 versi 2004+ (build 19041+) atau Windows 11
- Virtualization (Intel VT-x) aktif di BIOS — cek di Task Manager → tab Performance → CPU → "Virtualization: Enabled"
- Ruang disk kosong minimal 50GB (disarankan di drive SSD/NVMe, bukan HDD)
- Koneksi internet stabil
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

> **Catatan lokasi drive:** secara default, distro Ubuntu ini ter-install di drive **C:**. Kalau mau pindah ke drive lain, lihat **Section 16**.

### Batasi resource WSL2 agar tidak "memakan" semua RAM Windows

Buat/edit file `C:\Users\<USERNAME>\.wslconfig` (ganti `<USERNAME>` dengan username Windows kamu — cek lewat `$env:USERNAME` di PowerShell):

```ini
[wsl2]
memory=11GB
processors=6
swap=4GB
kernelCommandLine = "sysctl.vm.max_map_count=262144"
```

**Kenapa setting ini:**
- `memory=11GB` — menyisakan ±5GB untuk Windows + Docker Desktop UI + aplikasi lain.
- `processors=6` — sisakan 2 thread untuk Windows, dari total 8 thread i3-10105F.
- `kernelCommandLine` — **wajib**, karena Wazuh indexer (berbasis OpenSearch) butuh `vm.max_map_count ≥ 262144`. Tanpa ini, container indexer akan gagal start dengan error `max virtual memory areas vm.max_map_count too low`.

Restart WSL agar setting berlaku:

```powershell
wsl --shutdown
```

---

## 4. Langkah 2 — Install Docker Desktop

1. Download dari https://www.docker.com/products/docker-desktop/
2. Saat instalasi, **centang "Use WSL 2 instead of Hyper-V"**
3. Settings → **Resources → WSL Integration** → aktifkan toggle untuk `Ubuntu-22.04`
4. Verifikasi dari dalam WSL:

```bash
docker --version
docker compose version
```

---

## 5. Langkah 3 — Clone Repo & Generate Sertifikat

Semua langkah berikut dijalankan **di dalam terminal WSL (Ubuntu)**, bukan PowerShell.

```bash
cd ~
git clone https://github.com/wazuh/wazuh-docker.git -b v4.14.5
cd wazuh-docker/single-node
```

Generate sertifikat:

```bash
docker compose -f generate-indexer-certs.yml run --rm generator
```

Tersimpan otomatis di `config/wazuh_indexer_ssl_certs/`.

---

## 6. Langkah 4 — Sesuaikan Resource Indexer

```bash
nano single-node/docker-compose.yml
```

Cari bagian `wazuh.indexer` → `environment`:

```yaml
environment:
  - "OPENSEARCH_JAVA_OPTS=-Xms2g -Xmx2g"
```

Heap JVM idealnya setengah dari RAM yang dialokasikan ke WSL2. Dengan WSL2 dibatasi 11GB, alokasi 2GB untuk indexer heap cukup untuk skala home-lab.

---

## 7. Langkah 5 — Jalankan Stack

```bash
docker compose up -d
docker compose ps
```

Log seperti `Failed to connect to Wazuh indexer port 9200` saat startup itu normal — tunggu sampai semua service status `running`.

---

## 8. Langkah 6 — Akses Dashboard & Ganti Password Default

```
https://localhost
```

Login default: `admin` / `SecretPassword`.

**Segera ganti password ini** — proses lengkapnya ada di **Section 17**, termasuk catatan penting soal command yang sering gagal silent kalau tidak hati-hati.

---

## 9. Langkah 7 — Buka Port di Firewall Windows (kalau mau monitor device lain)

| Port | Fungsi |
|---|---|
| 443/tcp | Akses dashboard via browser |
| 1514/tcp | Komunikasi data agent → manager |
| 1515/tcp | Enrollment/registrasi agent baru |
| 55000/tcp | Wazuh REST API (opsional) |

Port 9200 (indexer) **sebaiknya tidak** dibuka ke jaringan luar.

> Untuk lab ini, bagian ini tidak diterapkan — tidak ada device lain di jaringan rumah. Disimpan sebagai referensi.

---

## 10. Langkah 8 — Monitor PC Windows Ini Sendiri

Agent native Windows dipasang di PC yang sama dengan manager untuk memonitor dirinya sendiri. Komunikasi agent↔manager lewat `127.0.0.1` (loopback) — rule firewall di Section 9 tidak diperlukan untuk konfigurasi ini.

**1. Download installer (PowerShell sebagai Administrator):**
```powershell
Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.14.5-1.msi -OutFile wazuh-agent.msi
```

**2. Install, arahkan ke manager lokal:**
```powershell
msiexec.exe /i wazuh-agent.msi /q WAZUH_MANAGER='127.0.0.1' WAZUH_AGENT_NAME='PC-Windows-Utama'
```

**3. Start service:**
```powershell
NET START WazuhSvc
```

> ⚠️ `NET STOP`/`NET START WazuhSvc` itu khusus **agent**, dijalankan di **PowerShell Windows**. Manager (container di WSL2) pakai `docker compose restart wazuh.manager` dari WSL — dua sistem ini jangan dicampur.

**4. Verifikasi dari sisi agent:**
```powershell
Get-Service WazuhSvc
Get-Content "C:\Program Files (x86)\ossec-agent\ossec.log" -Tail 30
```
Cari baris `Connected to the server`.

**5. Verifikasi di dashboard:** `https://localhost` → **Agents** → status `PC-Windows-Utama` harus `active`.

---

## 11. Custom File Integrity Monitoring (FIM)

Konfigurasi FIM dilakukan lewat **Group** di manager (bukan edit `ossec.conf` lokal di agent) — scalable kalau nanti nambah endpoint lain.

**1. Buat group via dashboard:**
`Management → Groups` → buat group `agent_windows` → assign agent ke group ini.

**2. Edit `agent.conf` group tersebut:**
```xml
<agent_config>
  <syscheck>
    <directories realtime="yes" report_changes="yes">C:\Users\GANTI_DENGAN_USERNAME\Downloads</directories>
    <directories realtime="yes" report_changes="yes">C:\Users\GANTI_DENGAN_USERNAME\Desktop</directories>
    <directories realtime="yes" report_changes="yes">C:\Users\GANTI_DENGAN_USERNAME\Documents\Projects</directories>
    <directories realtime="no" report_changes="no">C:\Program Files</directories>
    <directories realtime="no" report_changes="no">C:\Program Files (x86)</directories>

    <ignore>.tmp</ignore>
    <ignore>.crdownload</ignore>
    <ignore>.log</ignore>

    <windows_registry arch="both">HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Run</windows_registry>
    <windows_registry arch="both">HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\RunOnce</windows_registry>
  </syscheck>
</agent_config>
```

> ⚠️ Jangan tulis placeholder pakai tanda `<` `>` (misal `<USERNAME>`) — parser Wazuh membacanya sebagai tag XML baru, bukan teks biasa, error "unclosed tag". Pakai `GANTI_DENGAN_USERNAME` (tanpa kurung siku), ganti manual ke username asli (`$env:USERNAME`) sebelum save.

**Kenapa `realtime` dibedakan:** `Downloads`/`Desktop`/`Documents` paling sering kena modifikasi tidak sah, jadi `realtime="yes"`. `Program Files` jarang berubah, real-time di sana lebih mahal resource untuk value kecil.

**3. Validasi config sebelum restart:**
```bash
docker exec -it single-node-wazuh.manager-1 /var/ossec/bin/verify-agent-conf -f /var/ossec/etc/shared/agent_windows/agent.conf
```
> Flag `-g <nama_group>` **tidak ada** di tool ini — pakai `-f <path_lengkap>`, atau jalankan tanpa flag untuk scan semua group.

**4. Restart agent:**
```powershell
NET STOP WazuhSvc
NET START WazuhSvc
```

**5. Verifikasi:** `Management → Groups → agent_windows` — status agent harus "Synced". Cek `FIM → Events` setelah trigger perubahan test.

---

## 12. Vulnerability Detection

Sejak Wazuh 4.8.0, modul ini **enabled by default**.

**1. Cek status di manager:**
```bash
docker exec -it single-node-wazuh.manager-1 cat /var/ossec/etc/ossec.conf | grep -A 5 "vulnerability-detection"
```
Harus muncul `<enabled>yes</enabled>`.

**2. Modul ini butuh akses internet** dari container manager — sync feed dari `cti.wazuh.com`:
```bash
docker exec -it single-node-wazuh.manager-1 curl -sI https://cti.wazuh.com
```

**3. Cek progress sync feed:**
```bash
docker exec -it single-node-wazuh.manager-1 grep -i "vulnerability-scanner" /var/ossec/logs/ossec.log | tail -30
```
Urutan normal: `Starting vulnerability_scanner module` → `Database decompression finished` → `Vulnerability scanner module started` → `Initiating update feed process`. Modul sudah bisa korelasi CVE begitu status "started" muncul.

**4. Tuning Syscollector di agent** (data inventory paket yang dikorelasikan ke CVE) — di `agent.conf` group `agent_windows`:
```xml
<agent_config>
  <wodle name="syscollector">
    <disabled>no</disabled>
    <interval>1h</interval>
    <scan_on_start>yes</scan_on_start>
    <packages>yes</packages>
    <os>yes</os>
  </wodle>
</agent_config>
```

**5. Cek hasil:** Dashboard → **Vulnerability Detection → Inventory**, filter agent.

**Triase severity:** CVSS ≥7 (High/Critical) prioritas tindak lanjut, di bawah itu masuk backlog patch rutin.

---

## 13. Maintenance & Update

```bash
docker compose down   # stop tanpa hapus data
docker compose up -d  # start lagi
```

**Update versi:** ikuti panduan resmi di https://documentation.wazuh.com/current/deployment-options/docker/upgrading-wazuh-docker.html

**Backup data:**
```bash
docker run --rm -v <volume_name>:/data -v $(pwd):/backup busybox tar czf /backup/backup.tar.gz /data
```

---

## 14. Troubleshooting Cepat

| Gejala | Penyebab umum | Solusi |
|---|---|---|
| Container indexer restart terus / error `max virtual memory areas` | `vm.max_map_count` belum ke-set | Cek `.wslconfig`, `wsl --shutdown` lalu buka ulang WSL, verifikasi `sysctl vm.max_map_count` (harus ≥262144) |
| Dashboard tidak bisa diakses | Stack belum fully up, atau port bentrok | `docker compose ps`, pastikan tidak ada app lain pakai port 443 |
| Docker Desktop terasa berat / Windows lag | Alokasi memory WSL2 kebesaran | Turunkan `memory=` di `.wslconfig` |
| Agent tidak muncul di dashboard | Firewall blokir 1514/1515, atau salah `WAZUH_MANAGER` IP | Cek firewall rule, pastikan IP/hostname benar |
| Error "unclosed tag" di `agent.conf`/`ossec.conf` | Placeholder ditulis dengan `<` `>` | Pakai placeholder tanpa kurung siku, validasi pakai `verify-agent-conf` sebelum restart |
| Log vulnerability-scanner diam di "Initiating update feed process" | Normal, feed pertama besar | Tunggu, cek dengan `grep` (bukan `tail -f`) |
| Salah jalankan `NET STOP/START WazuhSvc` di WSL | Itu service Windows untuk agent | Wajib di PowerShell Windows; manager pakai `docker compose restart` |
| PowerShell error `'<' operator is reserved for future use` | Placeholder command ditulis literal dengan `<` `>` | `<` dan `>` di PowerShell itu operator redirect — ganti placeholder dengan nilai asli langsung, tanpa kurung |
| `Install-Module`/script PowerShell gagal: "running scripts is disabled" | Execution Policy default `Restricted` | `Set-ExecutionPolicy -ExecutionPolicy Bypass -Scope Process -Force` (scope session saja, lebih aman dari ubah policy permanen) |
| Installer PowerShell: `Cannot convert null to type "System.DateTime"` | Module dependency (`powershell-yaml` dll) ke-install partial/corrupt, atau `PowerShellGet` versi lama | Hapus folder module yang corrupt, install ulang `NuGet` provider + `PowerShellGet`, **buka PowerShell window baru**, baru retry |
| `securityadmin.sh`: `which: command not found` | Image indexer tidak punya binary `which`, `JAVA_HOME` tidak ke-set | Set `JAVA_HOME` manual: `export JAVA_HOME=/usr/share/wazuh-indexer/jdk/` sebelum jalankan script (dalam satu `bash -c '...'` yang sama) |
| `securityadmin.sh`: `FileNotFoundException` di path cert/config | Path default di dokumentasi tidak match struktur image versi ini | Cari path asli dengan `find / -name "root-ca.pem"` dan `find / -name "internal_users.yml"` di dalam container — jangan asumsikan path dari versi lama |
| Active Response tidak pernah trigger, command custom (`firewall-drop`) tidak dikenal di Windows agent | Command itu cuma ada di agent Linux | Cek isi folder `active-response/bin/` di agent — Windows agent pakai `netsh.exe`, bukan `firewall-drop` |
| Atomic Red Team test ter-flag Windows Defender | Folder `atomics` sengaja berisi pattern yang menyerupai malware nyata untuk testing | **Verifikasi dulu** di Protection History bahwa path-nya memang di `C:\AtomicRedTeam\...` sebelum exclude — jangan exclude folder lebih luas dari yang perlu |

---

## 15. Drive Mana yang Sebaiknya Dipakai?

Setup ini mengasumsikan 4 drive internal:

| Drive | Tipe | Cocok untuk Wazuh? |
|---|---|---|
| 256GB SSD M.2 SATA (OS) | SSD | Bisa, tapi sebaiknya jangan — OS makin padat |
| 1TB HDD 7200 2.5" (data) | HDD | ❌ Tidak disarankan — indexer butuh random I/O cepat |
| 3TB HDD 7200 3.5" (data) | HDD | ❌ Tidak disarankan — alasan sama |
| 128GB SSD M.2 SATA (kosong) | SSD | ✅ **Pilihan terbaik** |

**Rekomendasi: drive #4.** Drive internal tetap, bukan removable.

### Setup

**1. Format NTFS & assign drive letter** lewat `diskmgmt.msc`.

**2. Arahkan Docker Desktop disk image ke drive ini:**
Quit Docker Desktop → Settings → **Resources → Advanced** → "Disk image location" → `D:\DockerData` → Apply & restart.

**3. (Opsional) Pindahkan distro WSL Ubuntu juga:**
```powershell
wsl --shutdown
wsl --export Ubuntu-22.04 D:\WSL-Backup\ubuntu-export.tar
wsl --unregister Ubuntu-22.04
wsl --import Ubuntu-22.04 D:\WSL\Ubuntu-22.04 D:\WSL-Backup\ubuntu-export.tar --version 2
wsl --shutdown
```

### Soal kapasitas & dua HDD lama
128GB cukup lega untuk skala home-lab (minimum resmi 50GB). HDD (1TB/3TB) tetap berguna untuk backup berkala dan cold storage index lama — bukan untuk lokasi aktif indexer (random I/O-nya terlalu lambat). Retensi index aktif sekarang dikelola otomatis lewat **Section 20 (Index Lifecycle Management)**.

---

## 16. Hardening Keamanan — Password Default

Ada beberapa password terpisah yang perlu diganti dari default:

| Password | Dipakai untuk | Default |
|---|---|---|
| `admin` (indexer + dashboard) | Login dashboard, akses penuh data indexer | `admin` / `SecretPassword` |
| API user `wazuh-wui` | Komunikasi dashboard ↔ manager REST API | `wazuh-wui` / `MyS3cr37P450r.*-` |
| `kibanaserver`, `kibanaro`, `logstash`, `readall`, `snapshotrestore` | Account internal OpenSearch Security lain | masing-masing punya default sendiri |

Jalankan semua perintah dari WSL, di folder `~/wazuh-docker/single-node`.

### Bagian A — Ganti password `admin` indexer

**1. Generate hash:**
```bash
docker run --rm -ti wazuh/wazuh-indexer:4.14.5 bash /usr/share/wazuh-indexer/plugins/opensearch-security/tools/hash.sh -p 'PASSWORD_BARU_KAMU'
```

**2. Edit `internal_users.yml`:**
```bash
nano config/wazuh_indexer/internal_users.yml
```
Cari section `admin:`, timpa `hash:` dengan hash baru.

**3. Apply ke indexer yang sedang jalan:**

> ⚠️ **Catatan penting:** versi command resmi di dokumentasi Wazuh sering tidak match dengan struktur path di image container yang sebenarnya — `which` juga tidak tersedia di image ini. Kalau command versi "standar" gagal silent atau error path, **cari dulu lokasi asli file-nya** sebelum jalankan:
> ```bash
> docker exec -it single-node-wazuh.indexer-1 find / -name "root-ca.pem" -o -name "internal_users.yml" 2>/dev/null
> ```
> Command final yang terbukti jalan di setup ini (path bisa beda di versi image lain — selalu cross-check pakai `find` di atas dulu):

```bash
docker exec -it single-node-wazuh.indexer-1 bash -c 'export JAVA_HOME=/usr/share/wazuh-indexer/jdk/ && bash /usr/share/wazuh-indexer/plugins/opensearch-security/tools/securityadmin.sh -cd /usr/share/wazuh-indexer/config/opensearch-security -nhnv -cacert /usr/share/wazuh-indexer/config/certs/root-ca.pem -cert /usr/share/wazuh-indexer/config/certs/admin.pem -key /usr/share/wazuh-indexer/config/certs/admin-key.pem -p 9200 -icl'
```

Harus berakhir dengan `Done with success`.

**4. VERIFIKASI LANGSUNG — jangan skip:**
```bash
curl -k -u admin:PASSWORD_BARU_KAMU "https://localhost:9200/_cluster/health"
curl -k -u admin:SecretPassword "https://localhost:9200/_cluster/health"
```
Baris pertama harus sukses, baris kedua harus `Unauthorized`. Kalau password lama masih bisa login, perubahan **belum benar-benar ter-apply** — jangan lanjut ke langkah berikutnya sebelum ini benar.

**5. Update `docker-compose.yml`:**
```bash
nano docker-compose.yml
```
Ganti semua baris `INDEXER_PASSWORD=SecretPassword` (di `wazuh.manager` dan `wazuh.dashboard`) ke password baru.

**6. Restart & force logout sesi dashboard lama:**
```bash
docker compose restart wazuh.manager wazuh.dashboard
```
Restart container otomatis clear semua session state di server — tidak ada sesi lama yang bisa "nyangkut" pakai password default setelah ini. Login ulang di `https://localhost` pakai password baru.

### Bagian B — Ganti password API `wazuh-wui`

Beda sistem dari indexer (port 55000, bukan 9200).

**1. Auth pakai credential lama:**
```bash
TOKEN=$(curl -s -u wazuh-wui:MyS3cr37P450r.*- -k -X POST "https://localhost:55000/security/user/authenticate?raw=true")
```

**2. Cari ID user:**
```bash
curl -s -k -X GET "https://localhost:55000/security/users?username=wazuh-wui" -H "Authorization: Bearer $TOKEN" | python3 -m json.tool
```

**3. Set password baru** (ganti `<id>` sesuai hasil langkah 2):
```bash
curl -s -k -X PUT "https://localhost:55000/security/users/<id>" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"password":"PASSWORD_API_BARU"}'
```

**4. Update `docker-compose.yml`** — `API_PASSWORD=MyS3cr37P450r.*-` di service `wazuh.dashboard`.

**5. Restart dashboard:**
```bash
docker compose restart wazuh.dashboard
```

### Bagian C — Password user internal lain (`kibanaserver`, dll)

⚠️ `kibanaserver` satu-satunya yang **aktif dipakai** stack (dashboard pakai ini auth ke indexer). Empat lainnya tidak dipakai fungsional, rotate password-nya pure hygiene.

**1-2. Generate hash & edit `internal_users.yml`** (sama pola dengan Bagian A, ulangi per account).

**3. Apply** dengan command final yang sama di Bagian A (path `config/opensearch-security` + `JAVA_HOME`).

**4. WAJIB — update referensi `kibanaserver` di dashboard:**
```bash
nano config/wazuh_dashboard/opensearch_dashboards.yml
```
Cari `opensearch.password: kibanaserver` → ganti ke password baru.

**5. Restart dashboard.**

### Catatan keamanan tambahan

- **Jangan paste password/API key ke tempat yang ter-log/tersimpan** (chat, issue tracker, dokumen yang dibagikan) — anggap selalu exposed begitu sudah diketik di tempat seperti itu, dan rotate lagi.
- Simpan password final di password manager, bukan di file teks biasa.
- Jangan expose port 443/9200/55000 ke internet tanpa VPN/reverse-proxy tambahan.

---

## 17. Custom Rules & Tuning

Dengan 1 endpoint, tuning berbasis baseline behavior PC ini sendiri — perlu di-review ulang nanti begitu ada endpoint kedua.

Rule custom **selalu** ditulis di `local_rules.xml` (bukan edit default rules — ketimpa tiap update) dengan ID custom di range **100000–120000**.

**1. Identifikasi kandidat tuning** — Dashboard → `Security Events` → cari alert yang sering muncul tapi tidak actionable. Catat `rule.id`.

**2. Edit local rules:**
```bash
docker exec -it single-node-wazuh.manager-1 nano /var/ossec/etc/rules/local_rules.xml
```

**3. Turunkan severity:**
```xml
<group name="local,">
  <rule id="100001" level="3">
    <if_sid>GANTI_DENGAN_RULE_ID_ASLI</if_sid>
    <description>Override: severity diturunkan — normal behavior di home-lab ini</description>
    <options>no_full_log</options>
  </rule>
</group>
```

**Atau suppress total (false positive):**
```xml
<rule id="100002" level="0">
  <if_sid>GANTI_DENGAN_RULE_ID_ASLI</if_sid>
  <description>Ignore: confirmed false positive, [alasan spesifik]</description>
</rule>
```

**4. Test sebelum deploy:**
```bash
docker exec -it single-node-wazuh.manager-1 /var/ossec/bin/wazuh-logtest
```

**5. Apply:**
```bash
docker compose restart wazuh.manager
```

**6. Monitor 3-5 hari, satu rule per sesi tuning.**

---

## 18. Alerting Eksternal — Telegram

Wazuh tidak punya integrasi Telegram bawaan — custom integration script.

### A. Buat Bot & Dapatkan Chat ID
1. Telegram → **@BotFather** → `/newbot` → dapat bot token.
2. Kirim 1 pesan ke bot itu.
3. Dapatkan chat ID:
```bash
curl -s "https://api.telegram.org/botGANTI_DENGAN_TOKEN/getUpdates" | python3 -m json.tool
```

### B. Siapkan Folder Persistent

```bash
cd ~/wazuh-docker/single-node
mkdir -p custom-integrations
```

**Wrapper script** (`custom-integrations/custom-telegram`):
```sh
#!/bin/sh
WPYTHON_BIN="framework/python/bin/python3"
SCRIPT_PATH_NAME="$0"
DIR_NAME="$(cd $(dirname ${SCRIPT_PATH_NAME}); pwd -P)"
SCRIPT_NAME="$(basename ${SCRIPT_PATH_NAME})"
if [ -z "${WAZUH_PATH}" ]; then
    WAZUH_PATH="$(cd ${DIR_NAME}/..; pwd)"
fi
PYTHON_SCRIPT="${DIR_NAME}/${SCRIPT_NAME}.py"
${WAZUH_PATH}/${WPYTHON_BIN} ${PYTHON_SCRIPT} "$@"
```

**Python script** (`custom-integrations/custom-telegram.py`):
```python
#!/usr/bin/env python3
import json, sys, requests

def main(args):
    alert_file = args[1]
    chat_id = args[2]
    bot_url = args[3]
    with open(alert_file) as f:
        alert = json.load(f)
    rule = alert.get("rule", {})
    agent = alert.get("agent", {})
    text = (
        f"🚨 *Wazuh Alert*\n"
        f"Level: {rule.get('level')}\n"
        f"Rule: {rule.get('description')}\n"
        f"Agent: {agent.get('name')}\n"
        f"Time: {alert.get('timestamp')}"
    )
    requests.post(bot_url, data={"chat_id": chat_id, "text": text, "parse_mode": "Markdown"}, timeout=10)

if __name__ == "__main__":
    main(sys.argv)
```

### C. Mount ke Container
```yaml
      - ./custom-integrations/custom-telegram:/var/ossec/integrations/custom-telegram
      - ./custom-integrations/custom-telegram.py:/var/ossec/integrations/custom-telegram.py
```

### D. Set Permission
```bash
docker compose up -d wazuh.manager
docker exec -u root -it single-node-wazuh.manager-1 chown root:wazuh /var/ossec/integrations/custom-telegram /var/ossec/integrations/custom-telegram.py
docker exec -u root -it single-node-wazuh.manager-1 chmod 750 /var/ossec/integrations/custom-telegram /var/ossec/integrations/custom-telegram.py
```

### E. Tambah Block Integration di `wazuh_manager.conf`

> Lokasi file: `~/wazuh-docker/single-node/config/wazuh_cluster/wazuh_manager.conf` — file ini yang persistent (di-mount ke container dan otomatis di-copy ulang ke `/var/ossec/etc/ossec.conf` setiap restart). Jangan edit `ossec.conf` langsung di dalam container, perubahan akan hilang saat container di-recreate.

```xml
<integration>
  <name>custom-telegram</name>
  <level>10</level>
  <hook_url>https://api.telegram.org/botGANTI_DENGAN_TOKEN_BOT/sendMessage</hook_url>
  <api_key>GANTI_DENGAN_CHAT_ID</api_key>
  <alert_format>json</alert_format>
</integration>
```

`level=10` dulu (critical-only) sampai rules tuning di Section 17 lebih stabil — turunkan bertahap ke level 7 setelah dipantau beberapa hari.

### F. Restart & Test Manual
```bash
docker compose restart wazuh.manager
```
```bash
docker exec -it single-node-wazuh.manager-1 bash -c '
echo "{\"rule\":{\"level\":10,\"description\":\"Test alert\"},\"agent\":{\"name\":\"PC-Windows-Utama\"},\"timestamp\":\"test\"}" > /tmp/test_alert.json
/var/ossec/integrations/custom-telegram /tmp/test_alert.json GANTI_DENGAN_CHAT_ID "https://api.telegram.org/botGANTI_DENGAN_TOKEN_BOT/sendMessage"
'
```
Cek log kalau gagal: `docker exec -it single-node-wazuh.manager-1 tail -50 /var/ossec/logs/integrations.log`

---

## 19. Sysmon Integration

Windows Event Log default tidak menangkap process creation tree, command-line argument, network connection per-process. Sysmon mengisi blind spot ini, dan melengkapi FIM registry (Section 11) dengan konteks proses pelakunya.

### A. Install Sysmon
```powershell
Invoke-WebRequest -Uri https://download.sysinternals.com/files/Sysmon.zip -OutFile Sysmon.zip
Expand-Archive Sysmon.zip -DestinationPath C:\Sysmon
Invoke-WebRequest -Uri https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml -OutFile C:\Sysmon\sysmonconfig.xml
cd C:\Sysmon
.\Sysmon64.exe -accepteula -i sysmonconfig.xml
```
Config baseline pakai **SwiftOnSecurity** (standar industri, well-maintained, filtering noise sudah wajar).

### B. Konfigurasi Wazuh Agent — Ingest Channel Sysmon

Tambahkan di `agent.conf` group `agent_windows`:
```xml
<agent_config>
  <localfile>
    <location>Microsoft-Windows-Sysmon/Operational</location>
    <log_format>eventchannel</log_format>
  </localfile>
</agent_config>
```
Validasi (`verify-agent-conf -f ...`), lalu restart agent.

### C. Decoder & Rules

Wazuh sudah punya built-in ruleset Sysmon, aktif otomatis:
```bash
docker exec -it single-node-wazuh.manager-1 ls /var/ossec/ruleset/rules/ | grep -i sysmon
```
Coverage utama: `sysmon_id_1` (process creation), `_3` (network connection), `_7` (image load), `_8` (create remote thread — indikasi injection, T1055), `_11` (file create), `_13` (registry set — komplemen FIM registry).

### D. Validasi
```powershell
notepad.exe
```
Cek dashboard `Security Events` filter `Sysmon`, atau:
```bash
docker exec -it single-node-wazuh.manager-1 grep -i "sysmon" /var/ossec/logs/alerts/alerts.json | tail -5
```
(`archives.json` tidak ada secara default — itu cuma muncul kalau `logall_json` di-enable manual, bukan error.)

---

## 20. Validasi Deteksi — Atomic Red Team

⚠️ Lab ini jalan di PC harian, bukan VM terisolasi. Hanya jalankan test yang ditandai aman, selalu `-Cleanup` setelah test, satu test per sesi.

### A. Install
```powershell
Set-ExecutionPolicy -ExecutionPolicy Bypass -Scope Process -Force
IEX (IWR 'https://raw.githubusercontent.com/redcanaryco/invoke-atomicredteam/master/install-atomicredteam.ps1' -UseBasicParsing); Install-AtomicRedTeam -getAtomics -Force
```

Kalau Windows Defender flag file di folder `atomics` — itu **diharapkan** (folder ini sengaja berisi pattern menyerupai malware nyata untuk testing). Verifikasi dulu di Protection History bahwa path-nya memang `C:\AtomicRedTeam\...`, baru exclude scope sekecil mungkin:
```powershell
Add-MpPreference -ExclusionPath "C:\AtomicRedTeam"
```

### B. Pilih Test yang Selaras dengan Visibility

| Teknik | Sysmon Event ID Terkait | Catatan |
|---|---|---|
| T1087.001 (Account Discovery) | 1 | Aman — cuma enumerasi lokal |
| T1547.001 (Registry Run Key) | 13 | Aman, ada cleanup resmi — validasi FIM registry juga |
| T1059.001 test #1 | — | **Skip** — test ini literally Mimikatz, bukan simulasi ringan |

**Selalu cek detail test sebelum run:**
```powershell
Invoke-AtomicTest T1087.001 -ShowDetailsBrief
```
Kalau ada nama tool malware nyata (Mimikatz, Cobalt Strike) atau `IEX`/download dari URL eksternal di detailnya — skip, cari test number lain.

### C. Workflow Eksekusi
```powershell
Import-Module "C:\AtomicRedTeam\invoke-atomicredteam\Invoke-AtomicRedTeam.psd1" -Force
Invoke-AtomicTest T1087.001 -TestNumbers 1 -GetPrereqs
Invoke-AtomicTest T1087.001 -TestNumbers 1
```
Cek dashboard `Security Events` filter teknik/Sysmon, lalu:
```powershell
Invoke-AtomicTest T1087.001 -TestNumbers 1 -Cleanup
```

### D. Interpretasi Hasil

- Sysmon event muncul + Wazuh alert fire → detection confirmed.
- Sysmon event muncul tapi Wazuh **tidak** alert → gap nyata, bahan custom rule (Section 17).
- Sysmon event sendiri tidak muncul → cek config Sysmon, bukan masalah Wazuh.

---

## 21. Active Response

Manager bisa otomatis jalankan command di agent saat rule tertentu fire. Logic-nya di sisi **manager** (`wazuh_manager.conf`), bukan agent.

⚠️ Untuk single-PC lab, value ini lebih ke latihan skill daripada defense yang teruji terhadap ancaman nyata — hindari trigger yang terlalu sensitif sampai bisa nge-block diri sendiri.

### A. Cari Rule Trigger

Generate event logon gagal untuk dapat rule ID dari alert riil (bukan tebakan):
```powershell
net use \\localhost\C$ /user:testuser SalahPasswordSengaja
```
Cek dashboard `Security Events` filter `authentication_fail`, catat `rule.id`.

> Pakai username fiktif seperti `testuser` — tetap generate Event ID 4625 (logon failure) yang valid untuk trigger rule, dan menghindari risiko self-lockout kalau pakai akun asli dengan account lockout policy aktif.

### B. Command yang Tersedia di Windows Agent

Cek dulu isi folder ini — **jangan asumsikan nama command dari dokumentasi generik** (banyak contoh di internet pakai `firewall-drop`, yang sebenarnya cuma ada di agent Linux):
```powershell
Get-ChildItem "C:\Program Files (x86)\ossec-agent\active-response\bin\"
```
Windows agent menyediakan `netsh.exe`, `restart-wazuh.exe`, `route-null.exe`.

### C. Konfigurasi di `wazuh_manager.conf`

```bash
nano ~/wazuh-docker/single-node/config/wazuh_cluster/wazuh_manager.conf
```
Tambahkan di dalam `<ossec_config>` yang sudah ada:
```xml
<command>
  <name>netsh</name>
  <executable>netsh.exe</executable>
  <timeout_allowed>yes</timeout_allowed>
</command>

<active-response>
  <disabled>no</disabled>
  <command>netsh</command>
  <location>local</location>
  <rules_id>GANTI_DENGAN_RULE_ID_DARI_LANGKAH_A</rules_id>
  <timeout>60</timeout>
</active-response>
```
`<location>local</location>` — block hanya di agent yang trigger alert. `timeout=60` untuk testing awal, jangan langsung pasang lama.

```bash
docker compose restart wazuh.manager
```

### D. Verifikasi Siklus Penuh

Ulangi trigger dari langkah A, lalu cek:
```powershell
netsh advfirewall firewall show rule name="WAZUH ACTIVE RESPONSE BLOCKED IP" verbose
Get-Content "C:\Program Files (x86)\ossec-agent\active-response\active-responses.log" -Tail 20
```
Log harus tunjukkan siklus `add` → `check_keys` → `continue` → (setelah `timeout` detik) `delete`. Kalau dicek setelah window timeout lewat, `netsh` akan bilang "No rules match" — itu **benar**, bukan gagal, karena rule sudah auto-expire sesuai desain.

---

## 22. Threat Intel Enrichment (VirusTotal)

Built-in integration Wazuh, otomatis cek hash file yang ke-flag FIM ke VirusTotal.

### A. Konfigurasi

Di `wazuh_manager.conf` (path sama seperti Section 21):
```xml
<integration>
  <name>virustotal</name>
  <api_key>GANTI_DENGAN_VT_API_KEY</api_key>
  <group>syscheck</group>
  <alert_format>json</alert_format>
</integration>
```
API key gratis dari virustotal.com (rate limit 4 req/menit, cukup untuk home-lab).

```bash
docker compose restart wazuh.manager
```

### B. Validasi

> Catatan: kalau file test dibuat **selama** FIM baseline scan pertama di folder itu masih berjalan, tidak akan generate alert — baseline scan cuma "mendaftarkan" state awal. Tunggu baseline selesai (cek `Syscheck last ended at` via `agent_control -i <id>`), baru test.

```powershell
"test content $(Get-Date)" | Out-File C:\Users\GANTI_DENGAN_USERNAME\Downloads\test-vt.txt
```
```bash
docker exec -it single-node-wazuh.manager-1 grep -i "virustotal" /var/ossec/logs/alerts/alerts.json | tail -5
```
Hasil yang diharapkan: dua alert berurutan — FIM "File added" (`rule.id: 554`), lalu "VirusTotal: Alert" dengan field `data.virustotal` berisi hash dan status `found`/`malicious`. Untuk file biasa, hasilnya "No records in VirusTotal database" — itu normal, bukan gagal.

Kalau `integrations.log` kosong total:
```bash
docker exec -it single-node-wazuh.manager-1 tail -30 /var/ossec/logs/integrations.log
docker exec -it single-node-wazuh.manager-1 grep -i "integratord" /var/ossec/logs/ossec.log | tail -20
```

---

## 23. Index Lifecycle Management (ISM)

Tanpa retention policy, Wazuh bikin index baru setiap hari dan tidak pernah hapus sendiri. Default limit shard per node OpenSearch (1000) bisa kena dalam hitungan bulan dengan storage 128GB, dan kalau itu terjadi indexer berhenti terima data baru sama sekali.

### A. Cek Baseline
```bash
curl -k -u admin:PASSWORD_INDEXER_KAMU "https://localhost:9200/_cat/indices/wazuh-alerts-*?v"
```

### B. Buat ISM Policy

```bash
curl -k -u admin:PASSWORD_INDEXER_KAMU -X PUT "https://localhost:9200/_plugins/_ism/policies/wazuh-alert-retention-policy" \
  -H 'Content-Type: application/json' \
  -d '{"policy": {"policy_id": "wazuh-alert-retention-policy", "default_state": "retention_state", "states": [{"name": "retention_state", "actions": [], "transitions": [{"state_name": "delete_alerts", "conditions": {"min_index_age": "30d"}}]}, {"name": "delete_alerts", "actions": [{"retry": {"count": 3, "backoff": "exponential", "delay": "1m"}, "delete": {}}], "transitions": []}], "ism_template": [{"index_patterns": ["wazuh-alerts-*"], "priority": 1}]}}'
```
Retensi 30 hari — diturunkan dari contoh resmi 90 hari, disesuaikan kapasitas SSD 128GB.

### C. Apply ke Index yang Sudah Ada

`ism_template` cuma berlaku otomatis untuk index baru. Index lama perlu attach manual:
```bash
curl -k -u admin:PASSWORD_INDEXER_KAMU -X POST "https://localhost:9200/_plugins/_ism/add/wazuh-alerts-*" \
  -H 'Content-Type: application/json' \
  -d '{"policy_id": "wazuh-alert-retention-policy"}'
```
(Kalau ISM background job sudah lebih dulu auto-attach lewat `ism_template`, command ini akan return "already has a policy" — itu juga tanda sukses, bukan gagal.)

### D. Verifikasi
```bash
curl -k -u admin:PASSWORD_INDEXER_KAMU "https://localhost:9200/_plugins/_ism/explain/wazuh-alerts-*?pretty"
```
`policy_id` harus terisi, `enabled: true`.

> Status `yellow` dengan `unassigned_shards` di `_cluster/health` itu normal untuk single-node cluster — replica shard butuh node fisik berbeda yang tidak ada di setup ini. Tidak ada data yang hilang/rusak.

---

## 24. Langkah Lanjutan

- Selesaikan password user internal lain (Section 16 Bagian C) kalau belum.
- Test end-to-end Telegram alerting (Section 18) kalau belum pernah dapat notifikasi nyata.
- Lanjutkan custom rules tuning (Section 17) berdasarkan hasil Atomic Red Team — terutama kalau ada teknik yang Sysmon-nya capture tapi Wazuh tidak alert.
- Pertimbangkan Backup/restore drill nyata (bukan cuma punya command-nya) — terutama setelah ISM aktif, pastikan backup juga cover index yang masih ada sebelum auto-delete.
- Network visibility (Suricata/Zeek) — effort besar, butuh VM/appliance terpisah, untuk jangka panjang kalau mau nutup blind spot level network/traffic (saat ini lab 100% host-based).
- Threat Intel Enrichment lanjutan — hubungkan ke IOC Checker pribadi (refactor fungsi lookup OSINT jadi module Python terpisah, dipakai bersama Streamlit UI dan integration script Wazuh).

---

## 25. Single-node vs Multi-node — Kapan Pindah?

Panduan ini seluruhnya pakai **single-node**: manager, indexer, dashboard dalam satu stack container. Itu pilihan yang tepat untuk skala home-lab/single-host — tapi penting paham kapan arsitektur ini sudah tidak cukup, dan apa bedanya dengan **multi-node**.

### Perbedaan arsitektur, bukan cuma "lebih banyak langkah"

| Aspek | Single-node | Multi-node |
|---|---|---|
| Indexer | 1 node | Cluster — beberapa data node untuk redundansi & horizontal scaling |
| Manager | 1 instance | 1 master + beberapa worker, butuh load balancer (HAProxy/nginx) di depan untuk distribusi koneksi agent |
| Dashboard | 1 instance, sama host | Bisa terpisah, connect ke indexer cluster |
| Repo/config | `wazuh-docker/single-node/` | `wazuh-docker/multi-node/` — `docker-compose.yml` dan `instances.yml` beda struktur total |
| Target skala | 1-50 agent | Ratusan-ribuan agent, kebutuhan high-availability |
| Resilience | Single point of failure | Tahan kalau satu node mati |

Multi-node **bukan modifikasi** dari setup single-node yang sudah ada — itu deployment dari nol dengan topologi berbeda.

### Kenapa multi-node tidak masuk akal di satu PC fisik

Value utama multi-node adalah **redundansi**: kalau satu node mati, yang lain tetap jalan. Tapi kalau semua "node" itu jalan di **satu PC fisik yang sama**, redundansi itu fiktif — PC mati, semua node mati bersamaan. Hasilnya cuma overhead resource lebih besar tanpa benefit nyata. Untuk monitoring PC sendiri di rumah, **single-node tetap pilihan yang benar**, bukan langkah awal yang nanti "naik level" — beda use case sepenuhnya.

### Kapan pindah ke multi-node (di luar konteks home-lab ini)

- Jumlah agent sudah ratusan, satu indexer node kehabisan kapasitas storage/compute
- Butuh SLA uptime tinggi — tidak bisa terima downtime kalau satu komponen gagal
- Beban EPS tinggi sampai satu manager node jadi bottleneck

### Belajar topologinya tanpa korbankan stack yang sedang jalan

Karena relevan untuk konteks kerja MSSP, simulasi multi-node tetap punya value belajar — tapi sebaiknya jadi **lab terpisah** (beberapa VM di satu PC fisik, semacam rencana Proxmox yang pernah dieksplor sebelumnya), bukan modifikasi stack yang sedang aktif monitoring PC ini. Kalau eksperimen di lab terpisah itu rusak, tidak mengganggu visibility keamanan yang sudah berjalan.


