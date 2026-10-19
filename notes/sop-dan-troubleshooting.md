# SOP & Troubleshooting — Elastic Home Lab

> Dokumen ini merangkum Standard Operating Procedure (SOP) harian dan log troubleshooting yang ditemui selama membangun Elastic Stack home lab. Simpan di `notes/sop-dan-troubleshooting.md` di repo `elastic-homelab`.

---

## 1. Arsitektur Singkat

```
[Kali Linux] --(serangan)--> [Endpoint Linux / Windows] --(Elastic Agent)-->
[Fleet Server @ elastic-server :8220] --> [Elasticsearch :9200] <-- [Kibana :5601]
```

- Jaringan internal lab: VirtualBox **Host-only** (`192.168.56.0/24`)
- Jaringan internet tiap VM: VirtualBox **NAT** (`10.0.2.0/24`)
- Elastic Server: `192.168.56.10` (static)
- Endpoint Linux: `192.168.56.106` (DHCP, cenderung stabil)
- Endpoint Windows: `192.168.56.109` (DHCP)
- Kali: `192.168.56.102` (DHCP)

---

## 2. SOP Startup Harian

Checklist ini dijalankan **setiap kali** VM lab dinyalakan ulang setelah mati (semalaman/berhari-hari), untuk menghindari kesalahan diagnosis berulang.

### Langkah 1 — Nyalakan `elastic-server` dulu, tunggu

- Start VM `elastic-server` di VirtualBox.
- **Tunggu 3–5 menit** sebelum mencoba akses apa pun. Elasticsearch butuh waktu booting (proses Java berat).

### Langkah 2 — Cek service inti (jangan langsung restart)

```bash
sudo systemctl status elasticsearch kibana --no-pager -l
```

- `--no-pager`: cetak langsung tanpa mode scroll (tidak perlu tekan `q`).
- `-l`/`--full`: tampilkan baris log tanpa dipotong `...`.

**Baca hasilnya:**
| Kondisi | Tindakan |
|---|---|
| `active (running)`, sudah beberapa menit | Lanjut, tidak perlu apa-apa |
| `activating` | Tunggu 1–2 menit lagi, cek ulang |
| `failed` | Lanjut ke Langkah 3 (bersihkan & restart) |

### Langkah 3 — Kalau `failed`: bersihkan proses nyangkut, lalu restart

```bash
sudo systemctl kill elasticsearch
sudo systemctl reset-failed elasticsearch
sudo systemctl start elasticsearch
```

- `kill`: paksa hentikan semua proses yang masih menggantung dari percobaan start sebelumnya.
- `reset-failed`: bersihkan status "failed" supaya systemd mau mencoba start lagi.

Pantau live kalau perlu:
```bash
sudo journalctl -u elasticsearch -f
```
(`-f` = follow, log baru tampil real-time; tekan `Ctrl+C` untuk keluar)

### Langkah 4 — Cek RAM

```bash
free -h
```

Pastikan `available` tidak terlalu kecil (idealnya > 1 GB), dan tidak ada VM lain (Wazuh/Splunk/Kali) menyala kalau tidak sedang dipakai.

### Langkah 5 — Baru buka Kibana di browser

`http://192.168.56.10:5601` — hanya setelah Langkah 2–4 aman. Kalau masih loading lama di titik ini padahal service sudah lama `active`, baru itu tanda ada masalah lain.

### Langkah 6 — Nyalakan endpoint, cek status agent

```bash
sudo elastic-agent status
```

**Penting:** kalau muncul `FAILED`/`DEGRADED` dengan pesan error jaringan (misalnya `no route to host`, `connection refused`), **jangan langsung panik**. Itu sering kali **status basi** (hasil checkin terakhir sebelum jaringan stabil saat boot), bukan kondisi real-time. `elastic-agent status` tidak melakukan tes baru setiap dipanggil — ia melaporkan hasil checkin terakhir yang tersimpan di memori proses.

**Verifikasi dulu sebelum restart:**
```bash
ping -c 3 192.168.56.10
nc -zv 192.168.56.10 8220
curl -k -s -w "\nHTTP: %{http_code}\n" https://192.168.56.10:8220/api/status
```

Kalau ketiganya sukses tapi `elastic-agent status` masih error → paksa checkin ulang:
```bash
sudo systemctl restart elastic-agent
```
Tunggu 30–60 detik, cek lagi.

### Langkah 7 — Nyalakan endpoint Windows, cek service Elastic Agent

Berbeda dengan Linux (Elastic Agent jalan sebagai systemd service, auto-start saat boot), 
service Elastic Agent di Windows **tidak selalu ikut start otomatis** meski StartType 
sudah `Automatic` — khususnya setelah VM lama tidak dinyalakan. Jangan asumsi otomatis sehat.

```powershell
Get-Service -Name "Elastic Agent"
```

| Status | Tindakan |
|---|---|
| `Running` | Lanjut, cek Fleet → Agents untuk konfirmasi `Healthy` |
| `Stopped` | Start manual (lihat di bawah) |

**Kalau `Stopped`:** `Start-Service` HARUS dijalankan dari PowerShell **as Administrator** 
— kalau tidak, akan gagal dengan error `Cannot open Elastic Agent service`.

1. Buka PowerShell via **Run as administrator**
2. ```powershell
   Start-Service -Name "Elastic Agent"
   ```
3. Verifikasi ulang di Kibana → **Fleet → Agents**, tunggu sampai status `Healthy` (biasanya <1 menit).

**Catatan diagnosis sebelum sampai ke fix ini** (urutan dari luar ke dalam, jangan langsung restart):
```powershell
ping 192.168.56.10
Test-NetConnection -ComputerName 192.168.56.10 -Port 8220
```
Kalau keduanya sukses tapi Fleet tetap `Offline`, baru cek status service seperti di atas — 
itu tandanya bukan masalah jaringan, tapi service lokal di Windows.

---

## 3. Log Troubleshooting (Kronologis)

### Insiden #1 — Fleet output salah arah ke IP NAT

**Gejala:** Agent endpoint Linux terus `DEGRADED`, semua komponen error:
```
Elasticsearch request failed: dial tcp 10.0.2.15:9200: connect: connection refused
```
Padahal status `fleet: Connected` tampak sehat.

**Root cause:** Saat enrollment Fleet Server dilakukan di `elastic-server` dengan `--fleet-server-es=https://10.0.2.15:9200`, Kibana menyimpan alamat itu sebagai **output default** di `xpack.fleet.outputs` (`kibana.yml`). Alamat itu valid untuk Fleet Server sendiri (satu mesin dengan Elasticsearch), tapi ikut dibagikan ke semua agent lain — dan `10.0.2.15` di endpoint menunjuk ke **dirinya sendiri** (setiap VM NAT dapat IP yang sama), bukan ke `elastic-server`.

**Cara menemukan:**
```bash
# Cek alamat yang tersimpan di Kibana
sudo grep -E '^xpack\.fleet' /etc/kibana/kibana.yml

# Cek alamat yang benar-benar dipegang agent
sudo elastic-agent inspect | grep -n -A2 "hosts"
```

**Fix:**
1. Edit `/etc/kibana/kibana.yml`, ubah `hosts` di blok `xpack.fleet.outputs` dari `https://10.0.2.15:9200` ke `https://192.168.56.10:9200` (IP Host-only). **Jangan ubah `elasticsearch.hosts`** — itu untuk koneksi Kibana sendiri dan tetap boleh `10.0.2.15` karena satu mesin.
2. `sudo systemctl restart kibana`
3. Naikkan revisi agent policy (edit Description sedikit lalu Save) supaya Fleet mendorong konfigurasi baru.
4. Restart agent di endpoint: `sudo systemctl restart elastic-agent`

**Pelajaran:** Sertifikat Elasticsearch perlu dicek dulu apakah mencantumkan IP Host-only di SAN-nya sebelum pindah alamat:
```bash
echo | openssl s_client -connect 192.168.56.10:9200 2>/dev/null | openssl x509 -noout -ext subjectAltName
```

---

### Insiden #2 — Elasticsearch timeout saat start (21+ menit, gagal)

**Gejala:**
```
Active: failed (Result: timeout) since ...
elasticsearch.service: start operation timed out. Terminating.
```
CPU time terpakai 21+ menit, memory peak 4.3G — proses benar-benar jalan tapi tidak kunjung selesai sebelum systemd membatalkannya.

**Kemungkinan penyebab:** Resource host (CPU/RAM/disk I/O) tertekan saat proses start, membuat Elasticsearch yang biasanya start 1–2 menit jadi merangkak. Proses lama juga bisa meninggalkan child process (`java`, `controller`) yang masih menggantung setelah service dianggap gagal.

**Fix:**
```bash
sudo systemctl kill elasticsearch       # bersihkan proses nyangkut
sudo systemctl reset-failed elasticsearch
sudo systemctl start elasticsearch
```
Biasanya percobaan kedua berhasil normal (~1 menit) karena beban awal (cold start/cache) sudah terlewati di percobaan pertama.

**Pelajaran:** Jangan restart berulang tanpa `kill` + `reset-failed` dulu — proses lama yang menggantung bisa mengganggu percobaan start berikutnya.

---

### Insiden #3 — Lupa password Kibana

**Fix:**
```bash
sudo /usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic
```
Password baru ditampilkan sekali di terminal, simpan di luar repo.

---

### Insiden #4 — Hydra "0 valid password found" padahal password benar ada di wordlist

**Gejala:** `hydra -L users.txt -P passwords.txt ssh://...` selesai dengan `0 valid password found`.

**Root cause (gabungan 2 hal):**
1. Username yang dipakai login SSH sebenarnya berbeda dari yang diasumsikan (salah isi `users.txt`).
2. SSH server membatasi jumlah koneksi autentikasi paralel (`MaxStartups`), dan hydra default memakai 16 task paralel — sebagian percobaan ditolak sebelum sempat dicoba.

**Fix:**
- Pastikan username di `users.txt` benar-benar cocok dengan akun SSH asli di endpoint.
- Turunkan task paralel: `hydra -L users.txt -P passwords.txt -t 4 ssh://<IP>`

---

### Insiden #5 — Field KQL membingungkan saat mencari "login berhasil"

**Gejala:** Query `event.action : "authentication_success"` → kosong. Field `event.action: "ssh_login"` yang ditemukan sebagai alternatif ternyata **bukan** penanda sukses — baik `authentication_failure` maupun `ssh_login` sama-sama bisa punya `event.outcome: failure`.

**Fix — turun ke sumber paling mentah:**
```bash
sudo grep -a "Accepted password" /var/log/auth.log
```
(`-a`: paksa grep baca file sebagai teks meski terdeteksi "binary file matches")

Setelah ketemu baris aslinya (`Accepted password for ... from <IP> port ... ssh2`), pakai kata kuncinya untuk full-text search di Kibana:
```
"Accepted password" and source.ip : "<IP>"
```
Ternyata field yang menyimpan teks itu adalah **`message`** — field yang selalu ada secara default untuk log berbasis file di integrasi System (Elastic Agent), berisi baris log mentah apa adanya.

**Query final presisi:**
```
message : "Accepted password*"     // login berhasil
message : "Failed password*"        // login gagal
```

**Pelajaran utama:** Field hasil parsing (`event.action`, `event.outcome`) **tidak selalu bisa langsung dipercaya**. Kalau investigasi KQL mentok, turun ke log mentah (`message`, atau file log asli di endpoint via `grep`) adalah sumber kebenaran paling dasar.

---

### Insiden #6 — Rule preview timeout pada rentang waktu lebar

**Gejala:**
```
Error: Search has been aborted due to cancelled execution
Preview timed out after 60 seconds
```
Saat preview timeframe di-set ke "Last 7 days".

**Root cause:** Host dengan RAM terbatas kesulitan memproses pencarian pada rentang data besar dalam waktu 60 detik.

**Fix:** Persempit preview timeframe (mis. "Last 1 day" atau custom range sempit di sekitar waktu kejadian). Tidak memengaruhi rule yang sudah jalan — hanya soal preview di form pembuatan rule.

---

### Insiden #7 — Interval histogram salah karena huruf besar/kecil

**Gejala:** Mengisi "Minimum interval" di Lens dengan `1M` menghasilkan interval **1 bulan**, bukan 1 menit — chart jadi satu blok besar menutupi seluruh rentang waktu.

**Root cause:** Satuan waktu di Elastic **case-sensitive**: `m` (kecil) = menit, `M` (besar) = bulan, `h` = jam, `d` = hari, `w` = minggu.

**Fix:** Ketik ulang dengan huruf kecil yang benar (`1m` untuk 1 menit).

---

### Insiden #8 — Service & agent sudah sehat, tapi Kibana tetap tidak bisa diakses dari browser

**Gejala:** `sudo systemctl status elasticsearch kibana` menunjukkan **`active (running)`** untuk keduanya, dan `sudo elastic-agent status` di semua mesin (server + endpoint) menunjukkan **`HEALTHY`** — tapi `http://192.168.56.10:5601` tetap tidak terbuka di browser Windows host (loading terus, timeout, atau "site can't be reached").

**Kenapa ini membingungkan:** Semua indikator "normal" di level service dan agent sudah hijau, jadi penyebabnya **bukan** di Elasticsearch/Kibana/Fleet itu sendiri — harus dicari di lapisan lain (jaringan, firewall, atau browser).

**Cara diagnosis — telusuri dari dalam ke luar, jangan tebak:**

**Langkah 1 — Buktikan Kibana benar-benar menjawab dari dalam VM-nya sendiri**
```bash
curl -I http://localhost:5601
```
- `-I`: hanya ambil header response, cukup untuk tes cepat.
- Kalau ini **gagal**, masalahnya ada di Kibana sendiri meski statusnya "running" (misal proses hang, port tidak benar-benar listen). Cek `sudo ss -tlnp | grep 5601` untuk pastikan port benar-benar listen, dan di IP mana (harus `192.168.56.10:5601`, bukan `127.0.0.1:5601` saja).
- Kalau ini **berhasil**, lanjut ke langkah 2 — masalahnya ada di luar VM.

**Langkah 2 — Tes dari host Windows, bukan browser dulu**
```powershell
ping 192.168.56.10
Test-NetConnection 192.168.56.10 -Port 5601
```
- Kalau `ping` gagal: adapter Host-only bermasalah (sama seperti kasus endpoint yang pernah terjadi) — cek `ip -4 a` di elastic-server, pastikan `enp0s8` statusnya `UP` dan IP `192.168.56.10` masih ada.
- Kalau `ping` berhasil tapi `Test-NetConnection` ke port 5601 `False`: ada yang memblokir port tersebut di antara host dan VM — lanjut ke langkah 3.

**Langkah 3 — Cek firewall di dalam VM**
```bash
sudo ufw status
```
- Kalau UFW aktif dan tidak ada rule untuk port 5601, port itu akan diblokir meski Kibana sudah listen. Buka kalau perlu:
```bash
sudo ufw allow 5601/tcp
```

**Langkah 4 — Cek ulang `server.host` di kibana.yml**
```bash
sudo grep server.host /etc/kibana/kibana.yml
```
- Pastikan masih `"192.168.56.10"`, bukan ter-revert ke `"localhost"` (bisa terjadi kalau file config sempat ditimpa ulang saat reinstall/upgrade paket).

**Langkah 5 — Baru curigai sisi browser**
Kalau langkah 1–4 semua lolos (Kibana menjawab lokal, port terbuka dari host, firewall tidak memblokir, config benar) tapi browser tetap gagal:
- Pastikan mengetik `http://` bukan `https://` (Kibana di setup ini belum pakai TLS, beda dengan Elasticsearch yang wajib HTTPS).
- Coba **incognito/private window** atau browser lain — kemungkinan cache/cookie lama dari sesi sebelumnya (apalagi kalau IP atau port sempat berubah).
- Cek apakah ada VPN/proxy aktif di Windows host yang mengganggu akses ke jaringan Host-only.

**Pelajaran:** "Service running" dan "agent healthy" cuma membuktikan proses di dalam VM sehat — tidak membuktikan **jalur sampai ke browser kamu** juga sehat. Urutan diagnosis yang benar selalu dari titik paling dekat ke sumber masalah (proses lokal) baru melebar ke luar (jaringan → firewall → client), supaya tidak menebak-nebak di lapisan yang salah.

---

## 4. Ringkasan Detection Rule yang Dibuat

### Potential SSH Brute Force - Endpoint Linux

| Field | Nilai |
|---|---|
| Rule type | Custom query (KQL) |
| Query | `data_stream.dataset : "system.auth" and message : "Failed password*"` |
| Index pattern | `logs-*` |
| Severity | Medium |
| Risk score | 47 |
| Schedule | Runs every 5m, additional look-back 1m |
| Tags | `brute-force`, `ssh`, `linux`, `T1110.001` |

**Validasi:** Rule preview menangkap 181 alert historis (Last 1 day). Setelah di-enable, serangan Hydra baru menghasilkan 39 alert otomatis di halaman Alerts dalam satu siklus (±5 menit).

---

## 5. Dashboard / Visualisasi

### Chart: Tren Login Gagal per Menit (Endpoint Linux)

- Tipe: Bar chart (Lens)
- Filter: `data_stream.dataset : "system.auth" and message : "Failed password"`
- Horizontal axis: `@timestamp`, Date histogram, minimum interval `1m`
- Vertical axis: Count of records
- Breakdown: `source.ip` (Top values)

Hasil: pola klasik brute force — baseline datar, lonjakan tajam (~38 kejadian) dalam rentang waktu sangat singkat.

---

## 6. Rencana Lanjutan

- [ ] Simulasi serangan ke endpoint Windows (RDP/SMB brute force, cari field Event ID yang tepat — ulangi proses investigasi dari nol seperti di Linux)
- [ ] Detection Rule untuk Windows
- [ ] (Opsional) Upgrade rule dari Custom query ke tipe Threshold
- [ ] (Opsional) Notifikasi/aksi lanjutan saat alert terpicu
- [ ] Dokumentasi perbandingan multi-SIEM (Wazuh vs Splunk vs Elastic)