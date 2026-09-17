# Perbandingan Metode Onboarding Elastic Defend — Laptop BYOD di Cluster Air-Gapped

## Konteks

- Cluster Elastic bersifat **air-gapped** (Elasticsearch/Kibana tidak reachable langsung dari internet). Cluster on-prem sudah berlisensi **Enterprise**.
- Laptop BYOD secara default hanya punya koneksi **internet**, tidak punya akses ke jaringan internal.
- Koneksi ke cluster hanya tersedia setelah **VPN aktif**.
- VPN mensyaratkan **device posturing**: salah satu syaratnya adalah Elastic Defend sudah terinstall.
- Ini menimbulkan masalah *chicken-and-egg*: Elastic Defend butuh enroll ke Fleet (butuh akses ke cluster) → akses ke cluster butuh VPN aktif → VPN aktif butuh Elastic Defend sudah terinstall.
- **Data laptop BYOD boleh berada di luar on-prem** (tidak ada requirement data residency ketat untuk data ini).

Tiga metode berikut membahas cara memutus siklus tersebut.

---

## Metode 1: Onboarding Awal di Jaringan Lokal/Kantor

Laptop BYOD wajib berada di jaringan lokal kantor untuk instalasi & enrollment awal Elastic Defend. Setelah itu, VPN akan bisa aktif (device posturing lolos) dan operasional harian berjalan lewat VPN.

### Cara Kerja
1. Laptop terhubung ke jaringan kantor (Wi-Fi/LAN internal) yang punya akses langsung ke Fleet Server.
2. Elastic Agent + integrasi Elastic Defend diinstall dan enroll ke Fleet Server menggunakan enrollment token.
3. Setelah enroll, Elastic Defend aktif → device posturing mendeteksi Defend terinstall → VPN bisa diaktifkan.
4. Selanjutnya, laptop dipakai di luar kantor: VPN aktif dulu → baru dapat jalur ke cluster untuk update policy/artifact & kirim alert. Proteksi tetap enforce lokal dari cache walau VPN mati sementara.

### Kelebihan
- Tidak perlu membuka celah baru di perimeter jaringan — cluster tetap sepenuhnya tertutup dari internet.
- Arsitektur paling sederhana; tidak perlu reverse proxy, mTLS, atau komponen tambahan.
- Risiko keamanan terendah — permukaan serangan (attack surface) tidak bertambah.

### Kekurangan
- **Butuh kehadiran fisik** laptop di kantor — tidak berjalan untuk BYOD yang sepenuhnya remote/tidak pernah ke kantor.
- Proses onboarding bergantung pada logistik (device harus dibawa/dijadwalkan ke kantor).
- Enrollment token bisa kedaluwarsa jika proses onboarding memakan waktu lama.
- Kurang scalable untuk organisasi dengan banyak laptop baru atau tim yang tersebar secara geografis.

### Cocok Untuk
- Organisasi dengan kebijakan wajib onboarding on-site.
- Skenario di mana mayoritas laptop BYOD memang rutin ke kantor.

---

## Metode 2: Expose Fleet Server (+ Jalur Elasticsearch) via Reverse Proxy

Publish port/endpoint tertentu dari cluster Elastic on-prem ke internet lewat reverse proxy, dengan hardening keamanan, sehingga laptop bisa enroll & terhubung tanpa harus ke kantor dulu.

### Cara Kerja
1. Fleet Server on-prem (port default **8220**) di-publish ke internet lewat reverse proxy/load balancer.
2. Fleet Server hanya menangani *control plane* (enrollment, policy, check-in) — bukan *data plane*. Jalur terpisah ke Elasticsearch tetap diperlukan agar Elastic Defend bisa kirim alert/event, lewat reverse proxy terpisah atau fitur *proxy output* Elastic Agent.
3. Laptop BYOD bisa langsung enroll & install Elastic Defend lewat internet ke endpoint yang di-publish ini.
4. Setelah Defend terinstall, device posturing lolos, VPN bisa aktif untuk trafik selanjutnya.

### Kelebihan
- Tidak perlu kehadiran fisik di kantor — onboarding bisa dari mana saja.
- Scalable untuk organisasi besar/tersebar dan BYOD yang sepenuhnya remote.

### Kekurangan
- **Menambah attack surface** — Fleet Server & jalur Elasticsearch jadi reachable dari internet, walau via reverse proxy.
- Kompleksitas setup tinggi: reverse proxy, TLS dari CA publik, konfigurasi timeout long-polling, dan idealnya **mutual TLS (mTLS)**.
- mTLS butuh versi Fleet Server tertentu (8.19.19+ / 9.3.8+ / 9.4.4+ / 9.5.0+) — lisensi Enterprise sudah tersedia di kasus ini, jadi bukan blocker biaya.
- Fleet Server **tidak punya IP allowlisting/rate limiting native** — harus dibangun di layer reverse proxy/WAF.
- Enrollment token adalah API key Elasticsearch tanpa expiry otomatis — perlu proses manual rotasi/revoke.
- Perlu maintenance berkelanjutan: sertifikat, proxy, monitoring akses dari internet ke komponen cluster.

### Cocok Untuk
- Organisasi dengan banyak BYOD remote yang tidak realistis diminta ke kantor, tapi tetap ingin semua infrastruktur (termasuk Fleet Server) dikelola sendiri di on-prem.

---

## Metode 3: Elastic Cloud Hosted (ECH) untuk Endpoint + Cross-Cluster Search (CCS) Outgoing-Only dari On-Prem

Buat deployment Elastic Cloud Hosted (ECH) yang reachable dari mana pun untuk menampung enrollment & data Elastic Defend BYOD. Cluster on-prem air-gapped dikonfigurasi outgoing-only untuk query data dari ECH via Cross-Cluster Search (CCS).

### Cara Kerja
1. Buat deployment ECH — otomatis punya **Fleet Server bawaan** (bagian dari Integrations Server) yang internet-facing dengan TLS terkelola Elastic, tanpa perlu bangun reverse proxy sendiri.
2. Laptop BYOD enroll & kirim data Elastic Defend langsung ke ECH lewat internet — proses ini semulus SaaS security tool pada umumnya.
3. Cluster on-prem air-gapped dikonfigurasi sebagai **remote cluster client** untuk CCS ke ECH — cluster on-prem yang inisiasi koneksi keluar (outgoing-only), ECH tidak pernah menghubungi balik ke on-prem. Tidak ada port inbound baru yang perlu dibuka di jaringan internal.
4. SOC/analyst di on-prem melakukan query/investigasi data endpoint BYOD lewat CCS, sementara data operasional sehari-hari (server internal, dsb.) tetap di cluster on-prem seperti biasa.

### Kelebihan
- **Setup Fleet Server paling ringan** — tidak perlu reverse proxy atau mTLS custom karena ECH sudah menyediakannya secara default.
- Cluster air-gapped tetap **outgoing-only**, tidak ada exposure baru ke jaringan internal.
- CCS murni federated query saat search time (bukan replikasi) — cocok karena data BYOD memang boleh berada di luar on-prem.
- Lisensi Enterprise yang sudah dimiliki cluster on-prem sudah cukup untuk fitur CCS lanjutan (ES|QL cross-cluster search), asal sisi ECH juga di tier yang mendukung.
- Skalabel untuk BYOD remote tanpa menambah beban operasional reverse proxy di sisi on-prem.

### Kekurangan
- Menambah komponen infrastruktur baru (deployment ECH terpisah) dan biaya langganan cloud.
- Traffic CCS ditagih sebagai **data-out** di sisi ECH — perlu diperhitungkan untuk volume query yang sering.
- Bukan reference architecture resmi Elastic yang "dijamin" — kombinasi dua fitur (CCS + Fleet Server ECH) yang masing-masing didukung, tapi kombinasinya perlu divalidasi/diuji sendiri di lingkungan Anda.
- Kalau butuh alerting/dashboard real-time yang setara data lokal (bukan cuma query on-demand), mungkin perlu tambahan setup alerting yang jalan langsung di ECH.
- Latensi tambahan untuk query cross-cluster dibanding data yang tersimpan lokal (tidak dikuantifikasi resmi oleh Elastic, tapi merupakan konsekuensi inheren arsitektur federated search).

### Cocok Untuk
- Kasus ini secara spesifik: data BYOD boleh di luar on-prem, lisensi Enterprise sudah ada, dan ingin menghindari kompleksitas membangun reverse proxy + mTLS sendiri.

---

## Tabel Perbandingan Ringkas

| Kriteria | Metode 1: On-site Provisioning | Metode 2: Reverse Proxy Expose | Metode 3: ECH + CCS Outgoing-Only |
|---|---|---|---|
| Kehadiran fisik ke kantor | Wajib | Tidak perlu | Tidak perlu |
| Kompleksitas setup | Rendah | Tinggi | Sedang (tanpa perlu bangun reverse proxy/mTLS sendiri untuk Fleet) |
| Penambahan attack surface di sisi on-prem | Tidak ada | Ada (perlu mitigasi) | Tidak ada (on-prem tetap outgoing-only) |
| Kebutuhan lisensi tambahan | Tidak ada | Tidak ada (Enterprise sudah ada) | Tidak ada tambahan besar (Enterprise sudah ada); ada biaya langganan ECH + data-out |
| Skalabilitas untuk remote BYOD | Rendah | Tinggi | Tinggi |
| Data residency BYOD | Di on-prem | Di on-prem | Di ECH (cloud) — **sesuai kebutuhan kasus ini** |
| Maintenance berkelanjutan | Minimal | Signifikan (proxy, TLS, monitoring) | Sedang (kelola deployment ECH, biaya data-out, validasi CCS) |
| Kesesuaian dengan prinsip air-gapped (isolasi jaringan on-prem) | Tinggi | Sebagian (ada exception terkontrol) | Tinggi (on-prem tetap tanpa inbound baru) |

## Rekomendasi

Berdasarkan kondisi terbaru (data BYOD boleh di luar on-prem, lisensi Enterprise sudah tersedia): **Metode 3 (ECH + CCS outgoing-only) adalah pilihan yang paling direkomendasikan** untuk kasus ini.

Alasan utama:
- Menghilangkan kebutuhan membangun & memelihara reverse proxy + mTLS custom (beban operasional Metode 2).
- Tidak menambah attack surface di jaringan on-prem — prinsip air-gapped tetap terjaga dari sisi on-prem.
- Tidak ada isu data residency karena data BYOD memang diizinkan berada di luar on-prem.

Langkah lanjutan yang disarankan sebelum implementasi penuh:
1. Uji coba (POC) enrollment BYOD ke Fleet Server ECH dan verifikasi alur device-posturing Netskope berjalan seperti yang diharapkan.
2. Konfigurasi remote cluster (CCS) dari on-prem ke ECH menggunakan **API-key security model** (bukan cert-based) karena cukup trust satu arah, sesuai desain outgoing-only.
3. Hitung estimasi biaya data-out ECH berdasarkan proyeksi volume query CCS dari tim SOC/analyst.
4. Tentukan apakah cukup query on-demand (CCS) atau perlu alerting real-time tambahan yang berjalan langsung di ECH.

Metode 1 tetap relevan sebagai jalur onboarding cadangan untuk laptop yang kebetulan sedang berada di kantor, dan Metode 2 bisa disimpan sebagai opsi jika ke depannya kebijakan data residency berubah (data BYOD wajib tetap di on-prem).
