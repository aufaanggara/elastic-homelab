[← Fase Sebelumnya](04-detection-windows.md) | [README](README.md)

---

# Fase 5 — Hasil, Kesimpulan & Rencana Lanjutan

---

## Angka-Angka Project

| Metrik | Nilai |
|---|---|
| Total endpoint ter-monitor | 2 (Linux + Windows) + 1 Fleet Server self-monitoring |
| Detection Rule dibuat | 2 (Custom query — Linux, Threshold — Windows) |
| Skenario serangan disimulasikan | 2 (brute force SSH, brute force RDP), plus reconnaissance (nmap) di masing-masing |
| Alert tervalidasi (Linux) | 39 alert real-time, 181 alert historis dari rule preview |
| Alert tervalidasi (Windows) | 26 kejadian threshold dari satu IP |
| Dashboard dibuat | 2 (satu per endpoint, masing-masing 2 panel) |
| Insiden troubleshooting besar terdokumentasi | 8 (lihat [`referensi/sop-dan-troubleshooting.md`](referensi/sop-dan-troubleshooting.md)) |
| Total screenshot bukti | ~250+ (folder `evidence/`) |

---

## Pemetaan MITRE ATT&CK

| Teknik | ID | Fase |
|---|---|---|
| Network Service Discovery | T1046 | Reconnaissance (nmap, kedua endpoint) |
| Brute Force: Password Guessing | T1110.001 | Deteksi Linux (Hydra SSH) |
| Brute Force: Password Guessing | T1110.001 | Deteksi Windows (Hydra RDP) — sama dengan Linux, diisi eksplisit saat pembuatan rule |
| Valid Accounts | T1078 | Setelah kredensial berhasil didapat (kedua endpoint) |
| Remote Services: RDP | T1021.001 | Akses RDP berhasil, dikonfirmasi via xfreerdp |

---

## Temuan Paling Menarik

1. **Status "sehat" di dashboard SIEM tidak selalu berarti data benar-benar mengalir.** Pola ini muncul berulang kali di project ini — pada Fleet Agent ("Connected" tapi data tidak sampai karena salah alamat output) dan pada `elastic-agent status` (melaporkan status basi, bukan kondisi real-time). Pelajaran utamanya: selalu validasi di titik penerimaan data (Discover), bukan hanya percaya indikator status.

2. **Field hasil parsing SIEM bisa menyesatkan, log mentah adalah sumber kebenaran.** Ditemukan secara eksplisit saat field `event.action`/`event.outcome` tidak bisa diandalkan untuk membedakan login sukses/gagal di Linux — solusinya menelusuri langsung ke `auth.log` di endpoint.

3. **Perbandingan dua pendekatan Detection Rule dalam satu project**: Custom query manual (Linux) vs Threshold otomatis (Windows) — memberi pengalaman langsung membandingkan trade-off keduanya, bukan hanya membaca dokumentasi.

4. **Kegagalan tool pentest bisa jadi sinyal positif, bukan negatif** — kegagalan `crowbar`/`rdesktop` menghadapi NLA/CredSSP pada RDP Windows modern menunjukkan kontrol keamanan bekerja, bukan sekadar hambatan teknis yang harus "dikalahkan".

---

## Kendala Terbesar

1. **Output Fleet salah arah ke IP NAT** (`10.0.2.15` vs `192.168.56.10`) — insiden troubleshooting terbesar dan paling memakan waktu, melibatkan penelusuran dari gejala di endpoint sampai ke root cause di `kibana.yml`. Detail lengkap di [`referensi/sop-dan-troubleshooting.md`](referensi/sop-dan-troubleshooting.md), Insiden #1.

2. **Pola kegagalan startup berulang** setiap VM dinyalakan ulang setelah mati beberapa hari (Elasticsearch timeout, status agent basi) — diselesaikan dengan menyusun SOP startup harian yang sistematis, bukan sekadar restart berulang tanpa diagnosis.

3. **Konektivitas RDP Windows** — RDP tidak aktif secara default, dan Elastic Agent Windows sempat perlu di-start manual setelah VM lama tidak aktif, membuat fase Windows jauh lebih kompleks dibanding Linux. Jalan buntu sempat muncul dari profil jaringan `Public` (dicurigai jadi penyebab, ternyata bukan) sebelum akar masalah sebenarnya ditemukan: NLA/CredSSP yang tidak didukung tool `crowbar`/`rdesktop`, baru teratasi setelah beralih ke Hydra.

---

## Perbandingan Metodologi: Linux vs Windows

| Aspek | Linux | Windows |
|---|---|---|
| Log source | File teks (`auth.log`) | Structured Event Log (Security) |
| Field andalan | `message` (raw text) | `event.code`, `winlog.logon.type` (structured) |
| Tool serangan | Hydra (SSH) — langsung berhasil | Crowbar gagal → rdesktop gagal → Hydra (RDP) berhasil |
| Tipe Detection Rule | Custom query | Threshold |
| Kompleksitas prasyarat | Minimal (SSH aktif default) | Tinggi (RDP perlu diaktifkan manual, firewall, NLA) |

---

## Rencana Lanjutan

- [ ] Standardisasi tipe Detection Rule antara Linux dan Windows (pilih salah satu pendekatan sebagai baku, atau dokumentasikan alasan perbedaan secara eksplisit)
- [ ] Tambahkan notifikasi/aksi lanjutan saat alert terpicu (Slack/email) — sengaja dilewati karena di luar scope pembelajaran inti
- [ ] Lengkapi `learning-notes/` dengan rangkuman konsep umum (arsitektur Elastic Stack, KQL vs Lucene, dasar MITRE ATT&CK) yang terpisah dari narasi project-specific ini
- [ ] Dokumentasi perbandingan multi-SIEM (Wazuh vs Splunk vs Elastic) dari endpoint yang sama, memanfaatkan setup multi-forwarding yang sudah berjalan
- [ ] Push repository ke GitHub sebagai portofolio publik (saat ini masih lokal)
- [ ] Investigasi lebih lanjut root cause pasti kenapa Elastic Agent Windows (StartType: Automatic) tetap gagal start otomatis setelah VM lama tidak aktif — saat ini baru dimitigasi lewat SOP manual, belum ditemukan akar penyebab pastinya

---

[← Fase Sebelumnya](04-detection-windows.md) | [README](README.md)
