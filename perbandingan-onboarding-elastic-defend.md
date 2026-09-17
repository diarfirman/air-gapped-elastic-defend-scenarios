# Perbandingan Metode Onboarding Elastic Defend — Laptop BYOD di Cluster Air-Gapped

## Konteks

- Cluster Elastic bersifat **air-gapped** (Elasticsearch/Kibana tidak reachable langsung dari internet).
- Laptop BYOD secara default hanya punya koneksi **internet**, tidak punya akses ke jaringan internal.
- Koneksi ke cluster hanya tersedia setelah **VPN Netskope aktif**.
- VPN Netskope mensyaratkan **device posturing**: salah satu syaratnya adalah Elastic Defend sudah terinstall.
- Ini menimbulkan masalah *chicken-and-egg*: Elastic Defend butuh enroll ke Fleet (butuh akses ke cluster) → akses ke cluster butuh VPN aktif → VPN aktif butuh Elastic Defend sudah terinstall.

Dua metode berikut membahas cara memutus siklus tersebut.

---

## Metode 1: Onboarding Awal di Jaringan Lokal/Kantor

Laptop BYOD wajib berada di jaringan lokal kantor untuk instalasi & enrollment awal Elastic Defend. Setelah itu, VPN akan bisa aktif (device posturing lolos) dan operasional harian berjalan lewat VPN.

### Cara Kerja
1. Laptop terhubung ke jaringan kantor (Wi-Fi/LAN internal) yang punya akses langsung ke Fleet Server.
2. Elastic Agent + integrasi Elastic Defend diinstall dan enroll ke Fleet Server menggunakan enrollment token.
3. Setelah enroll, Elastic Defend aktif → device posturing Netskope mendeteksi Defend terinstall → VPN bisa diaktifkan.
4. Selanjutnya, laptop dipakai di luar kantor: VPN aktif dulu → baru dapat jalur ke cluster untuk update policy/artifact & kirim alert. Proteksi tetap enforce lokal dari cache walau VPN mati sementara.

### Kelebihan
- Tidak perlu membuka celah baru di perimeter jaringan — cluster tetap sepenuhnya tertutup dari internet.
- Arsitektur paling sederhana; tidak perlu reverse proxy, mTLS, atau komponen tambahan.
- Risiko keamanan terendah — permukaan serangan (attack surface) tidak bertambah.
- Selaras dengan prinsip air-gapped yang sudah ada (tidak ada pengecualian akses).

### Kekurangan
- **Butuh kehadiran fisik** laptop di kantor — tidak berjalan untuk BYOD yang sepenuhnya remote/tidak pernah ke kantor.
- Proses onboarding bergantung pada logistik (device harus dibawa/dijadwalkan ke kantor).
- Enrollment token bisa kedaluwarsa jika proses onboarding memakan waktu lama.
- Kurang scalable untuk organisasi dengan banyak laptop baru atau tim yang tersebar secara geografis.

### Cocok Untuk
- Organisasi dengan kebijakan wajib onboarding on-site (hari pertama kerja di kantor, dsb).
- Prioritas keamanan/kepatuhan air-gapped yang ketat, tidak ingin menambah exposure sama sekali.

---

## Metode 2: Expose Fleet Server (+ Jalur Elasticsearch) via Reverse Proxy

Publish port/endpoint tertentu dari cluster Elastic ke internet lewat reverse proxy, dengan hardening keamanan, sehingga laptop bisa enroll & terhubung tanpa harus ke kantor dulu.

### Cara Kerja
1. Fleet Server (port default **8220**) di-publish ke internet lewat reverse proxy/load balancer — ini pola resmi yang didukung Elastic untuk deployment on-prem/hybrid.
2. **Penting**: Fleet Server hanya menangani *control plane* (enrollment, policy, check-in) — bukan *data plane*. Jalur terpisah ke Elasticsearch tetap diperlukan agar Elastic Defend bisa kirim alert/event. Ini bisa lewat reverse proxy terpisah atau fitur *proxy output* Elastic Agent.
3. Laptop BYOD (walau baru pertama kali dan belum VPN) bisa langsung enroll & install Elastic Defend lewat internet ke endpoint yang di-publish ini.
4. Setelah Defend terinstall, device posturing lolos, VPN bisa aktif, dan selanjutnya trafik bisa lewat VPN seperti biasa (opsional — expose publik bisa tetap dipertahankan sebagai jalur cadangan).

### Kelebihan
- Tidak perlu kehadiran fisik di kantor — onboarding bisa dilakukan dari mana saja, kapan saja.
- Scalable untuk organisasi besar/tersebar, dan cocok untuk BYOD yang sepenuhnya remote.
- Mendukung update policy/artifact lebih rutin karena laptop tidak harus menunggu VPN aktif untuk terhubung.

### Kekurangan
- **Menambah attack surface** — ada komponen cluster (Fleet Server, jalur Elasticsearch) yang jadi reachable dari internet, walau melalui reverse proxy.
- Kompleksitas setup jauh lebih tinggi: perlu reverse proxy, sertifikat TLS dari CA publik, konfigurasi timeout untuk long-polling, dan idealnya **mutual TLS (mTLS)** agar hanya device sah yang bisa connect.
- **mTLS butuh lisensi Enterprise** dan versi Fleet Server tertentu (8.19.19+ / 9.3.8+ / 9.4.4+ / 9.5.0+) — ada biaya tambahan jika belum punya lisensi ini.
- Fleet Server **tidak punya IP allowlisting/rate limiting native** — kontrol ini harus dibangun sendiri di layer reverse proxy/WAF.
- Enrollment token adalah API key Elasticsearch tanpa expiry otomatis — perlu proses manual untuk rotasi/revoke agar tidak jadi celah keamanan jangka panjang.
- Perlu maintenance berkelanjutan: sertifikat, proxy, monitoring akses dari internet ke komponen cluster.

### Cocok Untuk
- Organisasi dengan banyak BYOD remote yang tidak realistis diminta ke kantor.
- Tim yang sudah punya kapasitas untuk mengelola reverse proxy, TLS/mTLS, dan lisensi Enterprise Elastic.

---

## Tabel Perbandingan Ringkas

| Kriteria | Metode 1: On-site Provisioning | Metode 2: Reverse Proxy Expose |
|---|---|---|
| Kehadiran fisik ke kantor | Wajib | Tidak perlu |
| Kompleksitas setup | Rendah | Tinggi |
| Penambahan attack surface | Tidak ada | Ada (perlu mitigasi) |
| Kebutuhan lisensi tambahan | Tidak ada | Enterprise (untuk mTLS) |
| Skalabilitas untuk remote BYOD | Rendah | Tinggi |
| Kecepatan onboarding | Bergantung logistik kantor | Bisa langsung, dari mana saja |
| Maintenance berkelanjutan | Minimal | Signifikan (proxy, TLS, monitoring) |
| Kesesuaian dengan prinsip air-gapped murni | Tinggi | Sebagian (ada exception terkontrol) |

## Rekomendasi

Untuk kasus di mana **sebagian besar laptop BYOD rutin ke kantor**, Metode 1 lebih sederhana dan aman — jadikan default, dan gunakan hanya untuk kasus reguler.

Untuk laptop yang **benar-benar tidak pernah ke kantor** (remote-first hire, dsb.), Metode 2 bisa jadi jalur khusus (bukan default untuk semua), dengan syarat minimal:
- mTLS aktif untuk autentikasi device di level Fleet Server.
- Jalur Elasticsearch untuk data plane turut di-hardening, bukan cuma Fleet Server.
- Proses rotasi/revoke enrollment token yang jelas.
- Monitoring akses reverse proxy sebagai bagian dari deteksi anomali.

Pendekatan **hybrid** (Metode 1 sebagai default, Metode 2 sebagai jalur khusus untuk kasus remote murni) kemungkinan paling realistis untuk kebanyakan organisasi.
