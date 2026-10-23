## 🗂️ Project Snapshot — Elastic Stack Home Lab

> Deskripsi project lengkap — siap dipakai sebagai context di LinkedIn, CV, resume, atau web portofolio.

**Nama Project:** Elastic Stack Home Lab (Multi-SIEM Monitoring)
**Periode:** Oktober 2026 — masih berjalan (dokumentasi tuntas, repo belum di-push)
**Kategori:** Blue Team / SIEM Deployment / Intrusion Detection
**Stack/Tools:** Elasticsearch 9.5.5, Kibana 9.5.5, Elastic Agent & Fleet, VirtualBox, Ubuntu Server 24.04 LTS, Kali Linux (Nmap, Hydra, Crowbar), Windows 10

**Deskripsi:**
Membangun Elastic Stack (Elasticsearch, Kibana, Fleet Server) dari nol secara manual di lingkungan home lab VirtualBox, memasang Elastic Agent pada endpoint Linux dan Windows, lalu mensimulasikan serangan brute force (SSH dan RDP) dari mesin Kali Linux untuk menguji kemampuan deteksi end-to-end — investigasi manual via KQL, pembuatan Detection Rule otomatis, hingga dashboard visualisasi. Project ini juga mendokumentasikan proses debugging infrastruktur nyata, bukan skenario yang selalu berjalan mulus.

**Yang sudah dikerjakan:**
- Infrastruktur Elastic Stack lengkap terpasang manual (bukan *unattended install*) dari ISO Ubuntu Server baru
- Dua endpoint (Linux & Windows) terpasang Elastic Agent dan terkonfirmasi mengirim log aktif
- Satu insiden troubleshooting infrastruktur besar ditemukan dan diselesaikan (output Fleet salah arah ke IP NAT)
- Troubleshooting Elastic Agent Windows yang kembali `Offline` setelah VM lama tidak aktif — diagnosis berjenjang (ping → port → service), ditemukan StartType sudah `Automatic` namun tetap perlu di-start manual (akar penyebab pasti belum terkonfirmasi), dimitigasi lewat penambahan SOP startup harian (Langkah 7)
- Simulasi serangan brute force terhadap kedua endpoint (SSH via Hydra, RDP via Hydra setelah Crowbar/rdesktop gagal karena tidak mendukung NLA/CredSSP)
- Investigasi manual menemukan field log yang reliabel untuk masing-masing OS (`message` di Linux, `event.code`/`winlog.logon.type` di Windows)
- Dua Detection Rule dibangun dengan pendekatan berbeda (Custom query vs Threshold) sebagai perbandingan metodologi, lengkap dengan pemetaan MITRE ATT&CK (Credential Access → Brute Force → Password Guessing/T1110.001) pada keduanya
- Dua dashboard visualisasi (satu per endpoint) tersimpan permanen
- Dokumentasi lengkap: writeup 5 fase, SOP troubleshooting (8 insiden), KQL cheatsheet, struktur repo rapi

**Angka & bukti konkret:**
- 2 endpoint di-monitor penuh (Linux + Windows) + 1 Fleet Server self-monitoring
- 2 Detection Rule aktif (Custom query & Threshold)
- 39 alert otomatis tervalidasi (Linux) + 26 kejadian threshold tervalidasi (Windows)
- 181 alert historis dari rule preview (Linux)
- 2 dashboard (4 panel total) tersimpan permanen
- 8 insiden troubleshooting infrastruktur nyata terdokumentasi lengkap dengan root cause
- ~250+ screenshot bukti teknis terorganisir di 4 kategori folder evidence
- 6 file writeup terstruktur (README + 5 fase) dengan navigasi antar-dokumen

**5 Screenshot Terpenting:**

| No | Nama File (dengan path folder) | Apa yang ditunjukkan |
|---|---|---|
| 1 | `evidence/03-detection/linux/49-dashboard-2-panel-lengkap.png` | Dashboard final brute force SSH Linux — tren waktu + top sumber IP dalam satu tampilan |
| 2 | `evidence/03-detection/linux/16-ketemu-field-message-login-berhasil.png` | Momen investigasi kunci: field `message` ditemukan sebagai sumber kebenaran log setelah field hasil parsing menyesatkan |
| 3 | `evidence/03-detection/linux/35-alert-muncul-39-alerts-bruteforce.png` | Bukti Detection Rule bekerja otomatis — 39 alert muncul tanpa intervensi manual saat serangan diulang |
| 4 | `evidence/03-detection/windows/19-event-4624-rdp-success-confirmed-logontype-10.png` | Analisis forensik Windows Event Log — LogonType 10 mengonfirmasi akses RDP berhasil |
| 5 | `evidence/03-detection/windows/38-dashboard-preview-2-panels-before-save.png` | Dashboard brute force RDP Windows, melengkapi visibilitas kedua endpoint |

---

## 📋 Project Progress Checklist — Elastic Stack Home Lab

**Terakhir diupdate:** 23 Oktober 2026

### A. Infrastructure Setup

| Status | Item | Catatan |
|---|---|---|
| ✅ | VM `elastic-server` dibuat (8GB RAM/4CPU/50GB disk) | ISO baru, terpisah dari VM Wazuh Manager |
| ✅ | Network NAT + Host-only dikonfigurasi | `192.168.56.0/24` internal, `10.0.2.0/24` NAT |
| ✅ | Ubuntu Server 24.04 terinstall manual, IP statis | `192.168.56.10` di `enp0s8` |
| ✅ | Elasticsearch terinstall & verified running | TLS + security aktif otomatis |
| ✅ | Kibana terinstall & dapat diakses dari browser | `server.host` dikonfigurasi ke IP Host-only |
| ✅ | Fleet Server terpasang & Healthy | Sempat terkendala encryption key, teratasi |
| ✅ | Agent policy Linux & Windows dibuat | Dipisah sejak awal untuk integrasi berbeda |

### B. Agent Deployment

| Status | Item | Catatan |
|---|---|---|
| ✅ | Agent Linux terpasang & enrolled | — |
| ✅ | Insiden output Fleet salah arah (IP NAT) diperbaiki | Insiden troubleshooting terbesar project ini |
| ✅ | Agent Windows terpasang & enrolled | Langsung Healthy karena fix sudah diterapkan duluan |
| ✅ | Troubleshooting Agent Windows `Offline` pasca-VM dormant | StartType sudah `Automatic`, tetap perlu di-start manual as Administrator; root cause pasti belum terkonfirmasi |
| ✅ | RDP diaktifkan di endpoint Windows | Registry + Firewall + Service, 3 lapisan |
| ✅ | SOP startup harian disusun & diperbarui | Awalnya mencegah salah diagnosis status basi (Linux); ditambah Langkah 7 khusus cek Elastic Agent Windows setelah temuan di atas |

### C. Detection — Endpoint Linux

| Status | Item | Catatan |
|---|---|---|
| ✅ | Port scan (nmap) terhadap endpoint Linux | Hanya port 22/SSH terbuka |
| ✅ | Simulasi brute force SSH (Hydra) | Wordlist custom, bukan rockyou.txt |
| ✅ | Investigasi KQL — field `message` ditemukan | Field hasil parsing (`event.action/outcome`) terbukti tidak reliabel |
| ✅ | Detection Rule (Custom query) dibuat & enabled | Severity Medium, risk score 47 |
| ✅ | Rule tervalidasi otomatis | 39 alert real-time, 181 alert historis preview |
| ✅ | Dashboard 2 panel dibuat & tersimpan permanen | Tren waktu + breakdown source.ip |

### D. Detection — Endpoint Windows

| Status | Item | Catatan |
|---|---|---|
| ✅ | Port scan RDP (nmap) | Port 3389 terbuka |
| ✅ | Percobaan Crowbar/rdesktop (gagal) → Hydra (berhasil) | NLA/CredSSP jadi penghambat tool lama (backend rdesktop); Hydra pakai backend freerdp yang kompatibel |
| ✅ | Investigasi Event ID 4625 (gagal) & 4624 (berhasil) | LogonType 10 (RemoteInteractive) = konfirmasi akses RDP; field `source.ip` tervalidasi reliable |
| ✅ | Validasi login sukses via xfreerdp (bukan sekadar Hydra) | Bukti independen kredensial valid secara fungsional |
| ✅ | Detection Rule (Threshold) + mapping MITRE ATT&CK | Pendekatan berbeda dari Linux sebagai perbandingan; MITRE: Credential Access (TA0006) → Brute Force (T1110) → Password Guessing (T1110.001) |
| ✅ | Rule tervalidasi otomatis | 26 kejadian threshold teragregasi dari satu source IP dalam 1 alert |
| ✅ | Dashboard 2 panel dibuat & tersimpan permanen | — |

### E. Dokumentasi & Struktur Repo

| Status | Item | Catatan |
|---|---|---|
| ✅ | Struktur folder direncanakan (`writeup/`, `evidence/`, `configs/`, `learning-notes/`) | — |
| ✅ | `sop-dan-troubleshooting.md` dibuat & diperbarui | 8 insiden terdokumentasi lengkap |
| ✅ | `kql-cheatsheet.md` dibuat | Mencakup query tervalidasi Linux & Windows, plus workflow validasi field baru |
| ✅ | File context handoff untuk kelanjutan project | Dipakai sukses untuk lanjut ke fase Windows |
| ✅ | Writeup 5 fase + README index dibuat | Format Incident Report, navigasi antar-file |
| ✅ | Backup konfigurasi (elasticsearch.yml, kibana redacted) | Password/token sudah di-redact |
| ✅ | `.gitignore` dibuat | — |
| ✅ | `README.md` root repo | Selesai digenerate |
| ⬜ | `learning-notes/` diisi (konsep umum di luar narasi project) | Folder masih kosong |
| ⬜ | Push repository ke GitHub | Masih lokal di `Downloads\cyber\projek\` |
| ⬜ | Upgrade Detection Rule Linux ke tipe Threshold | Opsional, belum dilakukan (Linux masih Custom query) |
| ⬜ | Notifikasi/aksi lanjutan saat alert (Slack/email) | Sengaja di luar scope project |

---

### 📊 Summary

| Metric | Jumlah |
|---|---|
| ✅ Selesai | 33 item |
| 🔄 Sedang berjalan | 0 item |
| ⬜ Belum dimulai | 4 item |
| **Total** | **37 item** |
| **Progress** | **≈89%** |

**Terakhir selesai di:** Troubleshooting Elastic Agent Windows (`Offline` → `Healthy` pasca-VM dormant) dan validasi penuh fase deteksi Windows (RDP brute force, Detection Rule Threshold, dashboard) tervalidasi dan terdokumentasi; `README.md` root repo serta `project-snapshot` ini selesai digenerate.

**Selanjutnya:** Mengisi `learning-notes/` dengan rangkuman konsep umum, lalu push repository ke GitHub sebagai portofolio publik.
