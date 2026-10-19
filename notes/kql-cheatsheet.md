# KQL Cheatsheet — Elastic Home Lab

> Kumpulan query KQL yang terbukti dipakai & tervalidasi sepanjang project `elastic-homelab`.
> Simpan di `notes/kql-cheatsheet.md`.

---

## 1. Dasar — Melihat Data per Dataset

Cek semua log dari satu integrasi tertentu, buat eksplorasi awal sebelum nyari spesifik.

```
data_stream.dataset : "system.auth"        // Linux - auth log
data_stream.dataset : "system.syslog"      // Linux - syslog
data_stream.dataset : "system.security"    // Windows - Security Event Log
data_stream.dataset : "system.system"      // Windows - System Event Log
data_stream.dataset : "system.application"  // Windows - Application Event Log
data_stream.dataset : "elastic_agent.*"    // Log internal agent & Fleet Server
```

---

## 2. Linux — SSH Brute Force

**Field yang RELIABLE** (hasil validasi manual, field hasil parsing seperti
`event.action`/`event.outcome` TIDAK bisa dipercaya langsung — lihat Insiden #5 di SOP):

```
message : "Failed password*"       // Login SSH gagal
message : "Accepted password*"     // Login SSH berhasil
```

**Dipakai di Detection Rule "Potential SSH Brute Force - Endpoint Linux":**
```
data_stream.dataset : "system.auth" and message : "Failed password*"
```

**Cross-check manual di endpoint (kalau KQL meragukan):**
```bash
sudo grep -a "Failed password" /var/log/auth.log
sudo grep -a "Accepted password" /var/log/auth.log
```

---

## 3. Windows — RDP Brute Force

**Field yang RELIABLE** (tervalidasi lewat cross-check ke `message` mentah):

```
event.code : "4625"      // Logon gagal (Failed logon)
event.code : "4624"      // Logon berhasil (Successful logon) — HATI-HATI, noisy!
source.ip : "<IP>"       // Field terstruktur, sudah tervalidasi akurat (beda dari kasus Linux)
winlog.logon.type : "RemoteInteractive"  // Login via RDP (Logon Type 10)
winlog.logon.type : "Network"            // Login via jaringan non-RDP (Logon Type 3)
winlog.logon.type : "Interactive"        // Login langsung di depan mesin (Logon Type 2)
winlog.logon.type : "Service"            // Logon otomatis oleh service Windows (Logon Type 5)
```

**Dipakai di Detection Rule "Potential RDP Brute Force - Endpoint Windows" (Threshold):**
```
data_stream.dataset : "system.security" and event.code : "4625"
```
→ Group by `source.ip`, Threshold `>= 5` per 5 menit.

**Mencari login RDP sukses dari IP tertentu (buat korelasi manual):**
```
data_stream.dataset : "system.security" and event.code : "4624" and winlog.logon.type : "RemoteInteractive"
```

**PENTING — kenapa `event.code: 4624` saja TIDAK CUKUP buat deteksi:**
`4624` menangkap SEMUA jenis logon sukses (interactive lokal, service, RDP, dll).
Login manual kamu sendiri di VirtualBox = `Logon Type: 2`, bukan `10`.
Selalu kombinasikan dengan `winlog.logon.type` biar presisi.

**Full-text search di `message` (dipakai saat field terstruktur diragukan/belum tervalidasi):**
```
message : "192.168.56.102"   // Cari IP tertentu di dalam raw text
```

---

## 4. Pola Umum — Menyempitkan Hasil

```
... and source.ip : "192.168.56.102"        // Filter per IP penyerang
... and @timestamp >= "2026-10-19T08:00:00"  // Filter custom time range (biasanya lebih gampang pakai time picker)
```

---

## 5. Pelajaran Penting Soal Field (Jangan Lupa!)

| Pelajaran | Insiden Terkait |
|---|---|
| Field hasil parsing (`event.action`, `event.outcome`) **tidak selalu reliable** — turun ke `message` kalau meragukan | Insiden #5 (Linux) |
| Field terstruktur **bisa** reliable (contoh: `source.ip` di Windows) — tapi tetap WAJIB divalidasi dulu, jangan asumsi | Validasi Windows |
| Satuan waktu interval di Lens **case-sensitive**: `m` = menit, `M` = bulan | Insiden #7 |
| `event.code: 4624` itu noisy — selalu kombinasikan dengan `winlog.logon.type` buat presisi | Investigasi RDP sukses |
| `Workstation Name` di Windows event **tidak reliable** buat identifikasi asal serangan (bisa beda-beda tergantung client) — pakai `Source Network Address` / `source.ip` | Validasi event 4624 RDP |

---

## 6. Workflow Validasi Field Baru (SOP)

Setiap kali nemu field baru yang belum pernah divalidasi, ikuti urutan ini:

1. Coba asumsi standar dulu (misal Event ID resmi Microsoft/field umum)
2. Buka 1 dokumen contoh di Discover, cari field `message` / `event.original` (raw text)
3. Bandingkan: apakah narasi di `message` match sama apa yang diharapkan dari field terstruktur?
4. Kalau match → field terstruktur aman dipakai buat rule/dashboard
5. Kalau TIDAK match / meragukan → pakai full-text search ke `message` sebagai sumber kebenaran
