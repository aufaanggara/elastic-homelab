# Elastic Stack Home Lab

Membangun Elastic Stack (Elasticsearch, Kibana, Fleet) dari nol di home lab VirtualBox, lalu mensimulasikan dan mendeteksi serangan brute force SSH & RDP secara end-to-end.

---

## Deskripsi

Project ini membangun SIEM berbasis Elastic Stack secara manual — bukan *unattended install* — di lingkungan VirtualBox dengan dua endpoint (Linux & Windows) yang dipasangi Elastic Agent. Setelah infrastruktur berjalan, dilakukan simulasi serangan brute force dari mesin Kali Linux terhadap kedua endpoint, lalu dibangun kemampuan deteksinya: investigasi manual via KQL, Detection Rule otomatis, hingga dashboard visualisasi. Repo ini juga mendokumentasikan insiden troubleshooting infrastruktur nyata yang ditemukan sepanjang proses, bukan hanya hasil akhir yang sudah rapi.

## Tech Stack & Tools

- **Elasticsearch & Kibana 9.5.5** — penyimpanan dan analisis log/SIEM
- **Elastic Agent & Fleet** — pengumpulan log dari endpoint secara terpusat
- **VirtualBox** — virtualisasi seluruh lab (server, endpoint, attacker)
- **Ubuntu Server 24.04 LTS** — OS server Elastic & endpoint Linux
- **Kali Linux** — Nmap (reconnaissance), Hydra (brute force SSH & RDP), Crowbar (percobaan brute force RDP)
- **Windows 10** — endpoint target kedua, RDP sebagai vektor serangan

## Struktur Folder

```
elastic-homelab/
├── README.md                 # file ini
├── writeup/                  # laporan investigasi terstruktur per fase
│   ├── README.md
│   ├── 01-setup-dan-instalasi.md
│   ├── 02-agent-linux-dan-windows.md
│   ├── 03-detection-linux.md
│   ├── 04-detection-windows.md
│   ├── 05-hasil-dan-kesimpulan.md
│   ├── referensi/
│   │   ├── sop-dan-troubleshooting.md
│   │   └── kql-cheatsheet.md
│   └── assets/
├── evidence/                  # seluruh screenshot bukti teknis
│   ├── 01-setup/
│   ├── 02-agents/
│   ├── 03-detection/
│   │   ├── linux/
│   │   └── windows/
│   └── 04-maintenance/
├── learning-notes/            # catatan belajar konsep umum (di luar narasi project)
├── configs/                   # backup konfigurasi (kredensial sudah di-redact)
│   ├── elasticsearch/
│   └── kibana/
└── .gitignore
```

## Progress

| Fase | Status |
|---|---|
| Infrastructure Setup | ✅ Selesai |
| Agent Deployment (Linux & Windows) | ✅ Selesai |
| Detection — Endpoint Linux | ✅ Selesai |
| Detection — Endpoint Windows | ✅ Selesai |
| Dokumentasi & Struktur Repo | 🔄 Hampir selesai |

**Progress keseluruhan: ≈89%** — lihat [`project-snapshot`](project-snapshot-elastic-homelab-2026-10-23.md) untuk checklist detail per item.

## Hasil Utama

- 2 endpoint termonitor penuh, 2 Detection Rule aktif (Custom query & Threshold), keduanya dilengkapi pemetaan MITRE ATT&CK (Credential Access → Brute Force → Password Guessing/T1110.001)
- 39 alert (Linux) + 26 kejadian threshold (Windows) tervalidasi otomatis dari simulasi serangan nyata
- 2 dashboard visualisasi tersimpan permanen
- 8 insiden troubleshooting infrastruktur nyata terdokumentasi lengkap dengan root cause — termasuk temuan operasional bahwa service Windows dengan `StartType: Automatic` tetap bisa gagal start otomatis setelah VM lama tidak aktif, dimitigasi lewat SOP startup harian

## Dokumentasi

Writeup lengkap tersedia di folder `writeup/` — lihat [`writeup/README.md`](writeup/README.md) untuk navigasi per fase, mulai dari setup infrastruktur hingga deteksi dan kesimpulan.

## Disclaimer

Project ini dibuat untuk tujuan pembelajaran dan portofolio pribadi di lingkungan lab terisolasi (VirtualBox, jaringan internal, tanpa koneksi ke sistem produksi atau pihak ketiga). Seluruh simulasi serangan (brute force SSH/RDP) dijalankan terhadap mesin virtual milik sendiri. Teknik dan tooling yang didokumentasikan di sini tidak boleh digunakan terhadap sistem apa pun tanpa izin eksplisit dari pemiliknya.
