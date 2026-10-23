# Elastic Stack Home Lab — Laporan Investigasi & Portofolio

**Tanggal:** Oktober 2026
**Platform:** Home Lab (VirtualBox, bukan platform CTF)
**Tingkat Kesulitan:** Menengah–Sulit
**Kategori:** SIEM Deployment, Intrusion Detection, Brute Force (SSH & RDP)
**Tools Utama:** Elasticsearch, Kibana, Elastic Agent/Fleet, Kali Linux (Nmap, Hydra, Crowbar), VirtualBox

---

## Ringkasan Eksekutif

Project ini membangun Elastic Stack (Elasticsearch, Kibana, Fleet Server) dari nol di lingkungan home lab VirtualBox, memasang Elastic Agent pada dua endpoint (Linux dan Windows), lalu melakukan simulasi serangan brute force terhadap keduanya (SSH dan RDP) untuk menguji dan membangun kemampuan deteksi secara end-to-end — mulai dari investigasi manual lewat KQL, pembuatan Detection Rule otomatis, hingga dashboard visualisasi. Sepanjang proses, ditemukan dan diselesaikan beberapa insiden infrastruktur nyata (bukan skenario buatan), termasuk kesalahan konfigurasi output Fleet yang membuat data gagal terkirim meski status terlihat sehat, dan pola kegagalan startup berulang pada lab yang sering dimatikan-nyalakan.

Seluruh endpoint juga dikonfigurasi untuk mengirim log secara paralel ke dua SIEM lain (Wazuh dan Splunk) sebagai bagian dari eksperimen multi-SIEM log forwarding, meski dokumentasi ini berfokus khusus pada implementasi Elastic.

---

## Daftar Fase

| # | Fase | Highlight |
|---|---|---|
| 1 | [Setup & Instalasi Infrastruktur](01-setup-dan-instalasi.md) | Elasticsearch, Kibana, Fleet Server terpasang manual; IP statis; TLS & encryption key dikonfigurasi dari nol |
| 2 | [Agent Linux & Windows + Troubleshooting](02-agent-linux-dan-windows.md) | Insiden besar: output Fleet salah arah ke IP NAT, diselesaikan lewat investigasi berlapis |
| 3 | [Deteksi Brute Force SSH (Linux)](03-detection-linux.md) | 39 alert otomatis tervalidasi; investigasi field menemukan `message` sebagai sumber kebenaran log |
| 4 | [Deteksi Brute Force RDP (Windows)](04-detection-windows.md) | RDP diaktifkan dari nol; Crowbar gagal, Hydra berhasil; rule tipe Threshold dengan mapping MITRE ATT&CK |
| 5 | [Hasil & Kesimpulan](05-hasil-dan-kesimpulan.md) | Angka-angka project, perbandingan metodologi Linux vs Windows, rencana lanjutan |

**Dokumen referensi pendukung** (dirujuk dari fase-fase di atas, bukan bagian narasi utama):
- [`referensi/sop-dan-troubleshooting.md`](referensi/sop-dan-troubleshooting.md) — SOP startup harian + log lengkap 8 insiden troubleshooting
- [`referensi/kql-cheatsheet.md`](referensi/kql-cheatsheet.md) — kumpulan query KQL tervalidasi dari investigasi

---

## Highlight Hasil Utama

- **2 endpoint** (Linux & Windows) terpasang Elastic Agent dan menerima log secara aktif
- **2 Detection Rule** dibangun dengan pendekatan berbeda (Custom query vs Threshold) sebagai perbandingan metodologi
- **39 alert** (Linux) dan **26 kejadian threshold** (Windows) tervalidasi secara otomatis dari simulasi serangan nyata
- **2 dashboard** visualisasi tren dan sumber serangan, tersimpan permanen
- **8 insiden troubleshooting infrastruktur nyata** terdokumentasi lengkap dengan root cause dan solusinya — bukan hanya "project berjalan mulus", tapi proses debugging yang representatif dengan pengalaman SOC sungguhan

---

## Arsitektur Singkat

```
[Kali Linux :192.168.56.102] --(nmap, hydra)--> [Endpoint Linux :192.168.56.106]
                                                  [Endpoint Windows :192.168.56.109]
                                                              |
                                                     (Elastic Agent)
                                                              v
                                                  [Fleet Server @ elastic-server :8220]
                                                              |
                                                              v
                                                  [Elasticsearch :9200] <-- [Kibana :5601]

Jaringan internal (Host-only): 192.168.56.0/24
Jaringan internet tiap VM (NAT): 10.0.2.0/24
```

---

## Struktur Repository

```
elastic-homelab/
├── README.md              # README utama repository (root)
├── writeup/                # Dokumen ini — laporan investigasi terstruktur
│   ├── README.md
│   ├── 01-05-*.md
│   ├── referensi/
│   └── assets/
├── evidence/                # Seluruh screenshot bukti teknis per fase
├── learning-notes/          # Catatan belajar konsep umum (terpisah dari narasi project ini)
└── configs/                 # Backup konfigurasi (kredensial sudah di-redact)
```

---

*Dokumentasi ini disusun mengikuti standar incident report blue team — setiap klaim teknis dilengkapi bukti screenshot dan command yang dapat ditelusuri ulang di folder `evidence/`.*
