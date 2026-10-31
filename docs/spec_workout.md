# JointFit --- Software Specification

**Dokumen:** `docs/spec_workout.md`\
**Versi:** 1.2\
**Status:** Draft untuk MVP\
**Bahasa:** Indonesia\
**Produk:** Personal Strength & Joint Stability Training App\
**Referensi integrasi:** `docs/reference/openapi.json` --- WhatsApp
Notification Service API v1.0.0

**Status konten:** seluruh seed latihan, media, teks instruksi, dan cue
keselamatan berstatus `pending_review` sampai ditinjau tenaga kesehatan.
Program tidak boleh ditawarkan sebagai program aktif sebelum
`clinical_review_status` menjadi `reviewed` (lihat §15 butir 14 dan §17).

------------------------------------------------------------------------

## 1. Ringkasan Produk

JointFit adalah aplikasi latihan personal untuk membantu pengguna
membangun kekuatan otot, kontrol gerakan, keseimbangan, dan kebugaran
secara bertahap. Program awal berfokus pada pengguna dewasa pemula
dengan hipermobilitas sendi dan bilateral flat foot.

Aplikasi bukan alat diagnosis, bukan pengganti dokter/fisioterapis, dan
tidak boleh menjanjikan pencegahan cedera atau koreksi struktural flat
foot. Rekomendasi latihan merupakan program umum yang harus dapat
ditinjau dan disesuaikan oleh tenaga kesehatan yang memahami kondisi
pengguna.

### 1.1 Tujuan MVP

1.  Menyimpan profil dan kondisi fungsional pengguna.
2.  Menampilkan program latihan 12 minggu.
3.  Menjadwalkan sesi latihan dan pengingat.
4.  Memandu pengguna melalui latihan, set, repetisi, dan waktu
    istirahat.
5.  Melakukan pemeriksaan kondisi sebelum dan sesudah latihan.
6.  Merekam riwayat dan perkembangan latihan.
7.  Menyediakan mekanisme regresi, jeda, atau penghentian latihan saat
    muncul gejala.

### 1.2 Bukan tujuan MVP

-   Mendiagnosis hypermobility spectrum disorder atau Ehlers-Danlos
    syndrome.
-   Menilai atau mengubah diagnosis flat foot.
-   Menggantikan penilaian fisioterapis.
-   Menggunakan AI untuk meresepkan latihan atau menaikkan intensitas
    secara otomatis.
-   Mengukur kekuatan otot secara klinis tanpa perangkat atau
    pemeriksaan yang tervalidasi.
-   Sinkronisasi lintas perangkat dan mode multi-user --- di luar
    cakupan MVP; semua data tetap terikat `user_id` agar tidak menutup
    jalur migrasi nanti (§18).

------------------------------------------------------------------------

## 2. Keputusan Produk dan Profil Awal

Pilihan platform dan teknologi berikut sudah diputuskan (§18) dan hanya
bila diubah melalui revisi dokumen ini:

  Parameter             Keputusan
  --------------------- ----------------------------------------------
  Platform              PWA (web responsif untuk desktop dan mobile)
  Frontend              React + TypeScript + Vite
  Styling               Tailwind CSS
  Backend               Go
  HTTP router           Chi
  Database              PostgreSQL
  Konfigurasi           `config.json` (base URL + credential gateway);
                        tanpa `.env`
  Autentikasi           PIN lokal (tanpa akun/registrasi), di-hash
                        argon2id; unlock berlaku selama aplikasi terbuka;
                        5 percobaan gagal → kunci 5 menit
  Pengingat             WhatsApp via gateway eksternal (opt-in) untuk
                        seluruh hari terjadwal + follow-up check; fallback
                        notifikasi in-app
  Offline               Baca via cache service worker + antrean log sesi
                        lokal (IndexedDB) yang disinkronkan
  Bahasa konten         Indonesia (`id-ID`), multi-bahasa ditunda
  Nama latihan           Istilah Inggris + alias Indonesia (`name_id`)
  Media latihan         Ilustrasi vektor SVG + `alt_text`, status review
                        menyusul
  Deployment            Docker lokal (localhost); HTTPS dan backup
                        terjadwal menyusul
  Tujuan penggunaan     Personal use, single user, satu perangkat
  Zona waktu            Asia/Jakarta
  Jam latihan default   07:00 WIB
  Hari latihan          Senin dan Kamis
  Aktivitas ringan      Rabu dan Sabtu
  Durasi program        12 minggu

### 2.1 Profil pengguna awal

  -----------------------------------------------------------------------
  Parameter                           Nilai
  ----------------------------------- -----------------------------------
  Usia                                30 tahun

  Gender                              Laki-laki

  Level aktivitas                     Pemula / jarang olahraga

  Hypermobility                       Lutut, pergelangan kaki,
                                      pergelangan tangan, jari tangan

  Flat foot                           Kedua kaki

  Nyeri saat ini                      Tidak ada, menurut keterangan
                                      pengguna

  Peralatan                           Tanpa alat

  Tujuan                              Kekuatan dan stabilitas seluruh
                                      tubuh
  -----------------------------------------------------------------------

Profil ini adalah konfigurasi awal, bukan kesimpulan medis. Pengguna
dapat mengubah profil dan kondisi kapan saja.

------------------------------------------------------------------------

## 3. Prinsip Keselamatan dan Batasan Klinis

Keselamatan harus menjadi prioritas tertinggi aplikasi.

1.  Latihan dilakukan dengan rentang gerak yang nyaman dan terkontrol.
2.  Pengguna tidak diarahkan untuk mengunci lutut, siku, atau
    pergelangan pada ujung rentang gerak.
3.  Hindari gerakan memantul, mendadak, atau memaksa fleksibilitas.
4.  Mulai dengan volume dan tingkat kesulitan rendah.
5.  Aplikasi tidak boleh menganggap semua pengguna dengan diagnosis yang
    sama memiliki toleransi latihan yang sama.
6.  Tidak ada progression otomatis hanya karena minggu program
    bertambah.
7.  Jika muncul nyeri sendi baru/meningkat, bengkak, sensasi sendi
    bergeser, ketidakstabilan, mati rasa, atau gejala yang
    mengkhawatirkan, hentikan latihan terkait dan sarankan evaluasi
    tenaga kesehatan.
8.  Nyeri dada, sesak napas yang tidak biasa, pingsan, atau gejala akut
    berat harus memunculkan instruksi menghentikan aktivitas dan mencari
    pertolongan medis segera.
9.  Pemeriksaan awal dan persetujuan fisioterapis dianjurkan sebelum
    pengguna memulai program mandiri, terutama untuk menilai teknik dan
    modifikasi yang sesuai.
10. Aplikasi tidak boleh menyatakan bahwa latihan akan menyembuhkan
    hypermobility atau mengembalikan lengkungan kaki.

### 3.1 Check-in sebelum latihan

Sebelum sesi dimulai, aplikasi menanyakan enam pertanyaan **ya/tidak**
(tanpa skor):

1.  Apakah ada nyeri baru atau nyeri yang lebih berat dari biasanya?
2.  Apakah ada bengkak?
3.  Apakah ada rasa sendi bergeser, hampir terlepas, atau tidak stabil?
4.  Apakah ada cedera atau kelelahan yang membuat gerakan hari ini tidak
    aman?
5.  Apakah ada nyeri dada, sesak napas yang tidak biasa, pusing sampai
    hampir pingsan, atau jantung berdebar tidak beraturan?
6.  Apakah ada gejala lain yang memburuk dibanding sebelum latihan
    terakhir?

Hasil ditentukan oleh **tabel aturan deterministik** berikut. Tabel ini
disimpan sebagai data (bukan hanya logika di UI), berlaku meski offline,
dan tidak memakai skor otomatis:

  #   Kondisi                                                   Tier    Tindakan aplikasi
  --- --------------------------------------------------------- ------- -----------------------------------------------------------
  1   Pertanyaan 5 dijawab ya                                   Akut    Hentikan sesi, tampilkan anjuran mencari pertolongan medis segera, jangan tawarkan latihan lagi hari ini
  2   Pertanyaan 3 atau 4 dijawab ya                            Berat   Hentikan latihan terkait; sarankan evaluasi tenaga kesehatan sebelum lanjut
  3   Pertanyaan 1 atau 2 atau 6 dijawab ya                     Ringan  Tawarkan lewati latihan terkait atau batalkan sesi; tidak memberikan diagnosis
  4   Tidak ada pertanyaan yang dijawab ya                      Aman    Pengguna dapat melihat sesi yang direncanakan

Aturan penggunaan tabel:

-   Jika beberapa kondisi terpenuhi, berlaku **tier tertinggi**.
-   Aplikasi tidak menentukan kelayakan medis hanya dari skor otomatis.
-   Aplikasi tidak memberikan diagnosis; tindakan hanya berupa lewati,
    ubah dosis, batalkan, atau hentikan.
-   Pilihan akhir pengguna (lanjut / lewati / hentikan) selalu dicatat ke
    `BodyCheck.action_taken`, apa pun tier-nya.
-   Jika pengguna memilih lanjut saat tier Ringan, tampilkan konfirmasi
    eksplisit bahwa pengguna memilih melanjutkan dengan gejala yang
    dilaporkan.
-   Tier Akut selalu memunculkan teks yang sama, tanpa variasi.

### 3.2 Check-in setelah latihan

Pada `WorkoutSession` dicatat **RPE (perceived exertion) 0--10**; inilah
satu-satunya sumber nilai RPE, tidak disimpan ganda di entitas lain.

Pada `BodyCheck` (tipe `post`) dicatat:

-   Nyeri sendi selama/setelah latihan, 0--10.
-   Kelelahan, 0--10.
-   Ada/tidaknya bengkak atau sensasi tidak stabil.
-   Jawaban pertanyaan gejala yang sama dengan §3.1.
-   Tier dan aturan yang terpicu (disimpan sebagai data, bukan dihitung
    ulang saat dibaca).
-   Catatan bebas.

**Aturan follow-up:** jika tier check-in Akut atau Berat, aplikasi
menjadwalkan `BodyCheck.check_type = follow_up` untuk 6--24 jam
berikutnya dan menampilkannya sebagai pekerjaan berikutnya di Dashboard.
Follow-up tidak boleh dipakai untuk menaikkan latihan, hanya untuk
mencatat kondisi.

Tampilkan pengingat untuk mengevaluasi kondisi beberapa jam kemudian
atau hari berikutnya. Jangan menganggap nyeri otot dan nyeri sendi
sebagai hal yang sama.

------------------------------------------------------------------------

## 4. Jadwal Mingguan

Jadwal default, dapat diedit pengguna:

  Hari       Waktu Aktivitas                              Durasi target
  -------- ------- ------------------------------------ ---------------
  Senin      07:00 Full Body A                             20--30 menit
  Selasa       --- Recovery / istirahat                             ---
  Rabu       07:00 Jalan santai atau aktivitas ringan      10--15 menit
  Kamis      07:00 Full Body B                             20--30 menit
  Jumat        --- Recovery / istirahat                             ---
  Sabtu      07:00 Jalan santai / keseimbangan ringan      10--15 menit
  Minggu       --- Istirahat                                        ---

Waktu 07:00 adalah preferensi jadwal, bukan klaim bahwa pagi lebih
unggul secara medis. Pengguna dapat mengubah waktu, hari, durasi, dan
pengingat.

Minggu 9--12 tidak otomatis menambah sesi ketiga. Sesi tambahan hanya
boleh ditampilkan sebagai opsi jika pengguna telah toleran terhadap
jadwal saat ini dan/atau program telah ditinjau oleh fisioterapis.

------------------------------------------------------------------------

## 5. Struktur Program 12 Minggu

  -----------------------------------------------------------------------
  Fase              Minggu            Fokus             Prinsip
  ----------------- ----------------- ----------------- -----------------
  1                 1--2              Adaptasi          Belajar teknik,
                                                        volume sangat
                                                        rendah

  2                 3--4              Kontrol gerakan   Konsistensi dan
                                                        kontrol rentang
                                                        gerak

  3                 5--6              Kekuatan dasar    Tambah volume
                                                        bila toleran

  4                 7--8              Daya tahan otot   Pertahankan
                                                        teknik, tambah
                                                        repetisi/set
                                                        bertahap

  5                 9--10             Stabilitas        Variasi sederhana
                                      fungsional        bila aman

  6                 11--12            Konsolidasi       Evaluasi
                                                        kapasitas dan
                                                        rencana
                                                        berikutnya
  -----------------------------------------------------------------------

### 5.1 Aturan progression

-   Minggu program tidak otomatis menaikkan beban, set, atau repetisi.
-   Progression hanya terjadi lewat aksi eksplisit pengguna (memilih
    "gunakan dosis berikutnya" atau target lebih tinggi) yang
    dikonfirmasi; tidak pernah dijalankan sistem saat sesi dibuka.
-   Naikkan satu variabel pada satu waktu.
-   Dahulukan teknik yang konsisten sebelum menambah repetisi.
-   Jika target tidak tercapai, pengguna dapat mengulang dosis yang
    sama.
-   Jika gejala meningkat, tawarkan pengurangan repetisi/set, regresi
    gerakan yang sudah ditentukan, atau penghentian sesi.
-   Jangan menaikkan kesulitan ketika pengguna melaporkan nyeri sendi,
    bengkak, atau ketidakstabilan.
-   Perubahan program klinis harus dapat ditinjau oleh fisioterapis.

Rentang di bawah merupakan target awal yang dapat dikonfigurasi, bukan
resep wajib untuk semua pengguna.

  -----------------------------------------------------------------------
  Periode                             Dosis awal yang ditargetkan
  ----------------------------------- -----------------------------------
  Minggu 1--2                         1 set, sekitar 5--8 repetisi sesuai
                                      latihan

  Minggu 3--4                         1--2 set, sekitar 6--10 repetisi

  Minggu 5--8                         Umumnya 2 set, sekitar 8--12
                                      repetisi bila toleran

  Minggu 9--12                        Umumnya 2 set; set ketiga hanya
                                      jika sesuai dan ditinjau
  -----------------------------------------------------------------------

**Pengecualian latihan isometric / tahanan statis** (Supine Abdominal
Brace, Wrist Isometric, Short Foot): dosisnya dihitung sebagai **set ×
durasi tahan**, bukan repetisi. Dosis awal **2 set × tahan 5 detik**
(dengan istirahat 5--10 detik antar set), maksimal **3 set × tahan 8
detik** setelah minggu ke-4 dan hanya bila toleran. Durasi tahan tidak
pernah dinaikkan bersamaan dengan jumlah set. Angka pada tabel §6.3
mengikuti aturan ini.

------------------------------------------------------------------------

## 6. Katalog Latihan

Setiap latihan pada katalog memiliki data:

-   ID, nama (istilah Inggris), dan alias nama Indonesia.
-   Deskripsi serta instruksi langkah demi langkah.
-   Kelompok otot utama dan sekunder.
-   Tujuan fungsional.
-   Peralatan.
-   Level kesulitan.
-   Set, repetisi/durasi, waktu tahan, dan istirahat.
-   Kesalahan umum.
-   Cue keselamatan dan kondisi untuk menghentikan gerakan.
-   Ilustrasi/video demonstrasi yang telah ditinjau, lengkap dengan
    teks alternatif (`alt_text`).
-   Regresi dan progresi yang disetujui.
-   Status review klinis dan versi konten.

### 6.1 Pemanasan

  Latihan                   Target                     Dosis awal
  ------------------------- -------------------------- ---------------------
  Diaphragmatic breathing   Kontrol napas              5--8 napas
  Shoulder rolls            Gerak bahu ringan          8 repetisi
  Ankle circles             Kontrol pergelangan kaki   5 putaran tiap arah
  Wrist circles             Gerak pergelangan tangan   5 putaran tiap arah

Pemanasan sekitar 4--5 menit. Jangan memaksa ujung rentang gerak.

### 6.2 Workout A

| Latihan | Target utama | Dosis awal | Cue keselamatan | Regresi (jika tidak mampu teknik/dosis) |
|---|---|---|---|---|
| Glute Bridge | Gluteus, hamstring | 1 × 8 | Jangan melengkungkan pinggang berlebihan | Bridge rentang seperempat + tahan 2 detik; kaki lebih rapat |
| Dead Bug versi mudah | Core | 1 × 5 per sisi | Mulai dengan heel slide; jaga batang tubuh stabil | Heel slide satu kaki dengan kedua lutut ditekuk, kaki tetap di lantai |
| Wall Push-up | Dada, triceps, bahu | 1 × 8 | Pergelangan netral; siku tidak dikunci | Posisi tangan lebih tinggi, rentang dorong separuh |
| Prone Scapular Retraction | Punggung atas | 1 × 8 | Gerak kecil; jangan mengangkat bahu ke telinga | Duduk, scapular squeeze tanpa mengangkat lengan |
| Sit to Stand | Quadriceps, gluteus | 1 × 6 | Gunakan kursi kokoh; lutut searah jari kaki; jangan mengunci lutut | Kursi lebih tinggi, tumpu tangan di paha, turun terkendali |
| Seated Calf Raise | Betis | 1 × 10 | Angkat dan turunkan tumit secara terkontrol | Ankle pump duduk tanpa beban penuh, rentang seperempat |

Istirahat 45--90 detik sesuai kebutuhan.

### 6.3 Workout B

| Latihan | Target utama | Dosis awal | Cue keselamatan | Regresi (jika tidak mampu teknik/dosis) |
|---|---|---|---|---|
| Clamshell | Gluteus medius | 1 × 8 per sisi | Panggul tidak berguling ke belakang | Rentang buka lebih kecil; pastikan panggul tetap lurus |
| Supine Abdominal Brace | Core dalam | 2 × tahan 5 detik | Bernapas normal; jangan mengejan | Brace ringan 3 detik tanpa menahan napas |
| Wall Angel, rentang nyaman | Bahu dan punggung atas | 1 × 6 | Jangan memaksa bahu ke atas | Gerak separuh rentang, siku tetap di bawah 90 derajat |
| Wrist Isometric | Otot pergelangan/lengan bawah | 2 × 5 detik per arah | Tahanan sangat ringan; pergelangan netral | Tangan disangga meja, tahanan mendekati nol |
| Short Foot | Otot intrinsik kaki | 2 × tahan 5 detik per kaki | Jangan mencengkeram lantai dengan jari | Latihan duduk saja; angkat lengkung tanpa menekuk jari |
| Toe Spreading | Otot kecil kaki | 5 repetisi | Rentang nyaman, tanpa memaksa jari | Gerak jari dalam rentang nyaman tanpa penekanan |
| Finger Intrinsic / buka-tutup tangan | Otot intrinsik tangan | 5--8 repetisi | Hindari mengunci atau mendorong jari ke hyperextension | Finger glide dengan jari tetap lurus, tanpa tekanan |

Latihan pergelangan tangan dan jari harus dapat diganti dengan versi
lebih sederhana jika pengguna kesulitan mengontrol posisi sendi. Hindari
tumpuan berat pada tangan/jari pada tahap awal.

Dosis isometric pada tabel di atas mengikuti §5.1 (2 set × 5 detik,
maksimal 3 set × 8 detik setelah minggu ke-4).

### 6.4 Aktivitas ringan dan keseimbangan

| Aktivitas | Target | Dosis awal |
|---|---|---|
| Jalan santai | Kebugaran umum | 10--15 menit |
| Supported Standing Balance | Proprioception kaki/pergelangan | 2 × 15--20 detik |

Hari aktivitas ringan (Rabu dan Sabtu) tetap diawali pemanasan ringan
2--3 menit (napas diafragma, ankle/wrist circles, shoulder rolls) dan
ditutup napas diafragma 5--8 tarikan, agar pengguna tidak masuk ke
aktivitas dalam keadaan dingin.

Untuk keseimbangan, pengguna berdiri dengan kaki selebar pinggul dan
memegang kursi/dinding. Jangan langsung menggunakan permukaan tidak
stabil atau latihan berdiri satu kaki tanpa penilaian dan progresi yang
sesuai.

### 6.5 Ruang lingkup bilateral flat foot

Program awal menyebut bilateral flat foot sebagai kondisi yang
dilaporkan pengguna. Batasannya:

-   Aplikasi hanya menampilkan latihan penguatan umum kaki (Short Foot,
    Toe Spreading, Seated Calf Raise, kontrol pergelangan kaki) sebagai
    bagian program umum.
-   Aplikasi **tidak boleh** menampilkan skor lengkungan, grafik
    perubahan bentuk kaki, atau klaim "perbaikan"/koreksi flat foot.
-   Aplikasi **tidak boleh** memberi rekomendasi sepatu, ortosis, atau
    latihan gait sebagai saran medis; bila pengguna bertanya, tampilkan
    anjuran menemui tenaga kesehatan.
-   Halaman Progress tidak boleh memuat metrik "koreksi flat foot".
-   Yang boleh ditampilkan: latihan yang dilakukan, konsistensi, dan
    kondisi yang dilaporkan pengguna sendiri.

------------------------------------------------------------------------

## 7. Target Otot dan Target Fungsional

Aplikasi harus membedakan **target otot** dari **target hasil
fungsional**. Jangan menampilkan persentase kekuatan otot jika tidak
diukur secara valid.

  -----------------------------------------------------------------------
  Kelompok otot           Tujuan fungsional       Latihan terkait
  ----------------------- ----------------------- -----------------------
  Core                    Kontrol batang tubuh    Dead Bug, Abdominal
                                                  Brace

  Gluteus maximus         Ekstensi pinggul        Glute Bridge

  Gluteus medius          Kontrol panggul         Clamshell

  Quadriceps              Kontrol gerak lutut     Sit to Stand

  Hamstring               Dukungan gerak          Glute Bridge
                          pinggul/lutut           

  Calf                    Kapasitas otot betis    Seated Calf Raise

  Intrinsic foot muscles  Kontrol kaki dan        Short Foot, Toe
                          lengkungan dinamis      Spreading

  Chest/triceps           Kekuatan dorong tubuh   Wall Push-up
                          atas                    

  Upper back/scapular     Kontrol tulang belikat  Prone Scapular
  muscles                                         Retraction, Wall Angel

  Wrist/forearm           Kontrol pergelangan     Wrist Isometric

  Finger intrinsic        Kontrol jari            Finger Intrinsic
  muscles                                         Exercise
  -----------------------------------------------------------------------

### 7.1 Metrik keberhasilan

-   Konsistensi sesi yang diselesaikan.
-   Repetisi/set yang dilakukan dengan teknik baik.
-   Kemampuan menyelesaikan gerakan pada dosis yang sama dengan RPE yang
    sesuai.
-   Durasi keseimbangan dengan dukungan yang sama.
-   Kondisi sendi sebelum dan setelah latihan.
-   Toleransi aktivitas sehari-hari yang dilaporkan pengguna.

Tidak boleh mengklaim peningkatan stabilitas sendi secara klinis hanya
dari jumlah repetisi.

------------------------------------------------------------------------

## 8. Fitur Fungsional

### 8.1 User Profile

-   Lihat dan edit profil.
-   Usia, gender, level aktivitas.
-   Kondisi yang dilaporkan pengguna.
-   Area sendi yang perlu diperhatikan.
-   Peralatan yang tersedia.
-   Tujuan latihan.
-   Preferensi hari dan waktu latihan.
-   Zona waktu.

### 8.2 Dashboard

Menampilkan: - Workout berikutnya. - Hari dan jam latihan. - Fase
program dan minggu berjalan. - Durasi estimasi. - Ringkasan sesi minggu
ini. - Tombol mulai sesi. - Peringatan jika ada check-in yang belum
diselesaikan.

### 8.3 Workout Session

-   Menampilkan satu latihan pada satu waktu.
-   Instruksi dan media demonstrasi.
-   Target otot.
-   Set dan repetisi/durasi.
-   Timer istirahat.
-   Tandai set selesai.
-   Edit repetisi aktual.
-   Lewati latihan dengan alasan.
-   Pause dan akhiri sesi.
-   Tampilkan cue keselamatan secara jelas.

Sesi tidak boleh dianggap selesai hanya karena timer habis. Pengguna
harus mengonfirmasi set atau latihan yang benar-benar dilakukan.

### 8.4 Body Condition Check

-   Check-in sebelum (6 pertanyaan §3.1) dan sesudah latihan.
-   Tier hasil dan daftar aturan yang terpicu.
-   Pilihan tindakan pengguna: lanjut, lewati latihan terkait, batalkan
    sesi, atau hentikan.
-   Skor nyeri 0--10.
-   Skor kelelahan 0--10.
-   Bengkak: ya/tidak.
-   Sensasi tidak stabil/bergeser: ya/tidak.
-   Catatan bebas.
-   Riwayat check-in, termasuk follow-up terjadwal (§3.2).

### 8.5 Progress Tracking

-   Kalender latihan.
-   Riwayat sesi.
-   Set dan repetisi aktual.
-   RPE.
-   Grafik konsistensi mingguan.
-   Perbandingan performa latihan yang sama, dibatasi pada latihan
    dengan `exercise_content_version` yang sama; jika versi konten
    berbeda, tampilkan penanda perbedaan versi, bukan garis tren
    tunggal.
-   Ringkasan kondisi yang dilaporkan pengguna.
-   Tidak ada metrik "koreksi flat foot" (lihat §6.5).

### 8.6 Scheduler & Reminder

-   Pengguna dapat mengubah jam dan hari.
-   Pengingat sebelum sesi.
-   Pengingat aktivitas ringan jika diaktifkan.
-   Tidak mengirim pengingat latihan ketika sesi dijeda.
-   Zona waktu mengikuti profil/perangkat dengan default Asia/Jakarta.

**Kanal pengiriman: WhatsApp** melalui gateway eksternal (kontrak:
`docs/reference/openapi.json`). Aturan wajib:

1.  **Cakupan.** Seluruh hari terjadwal dikirim via WhatsApp: sesi utama
    (Senin/Kamis), aktivitas ringan (Rabu/Sabtu), dan pengingat
    `follow_up` check-in --- estimasi 4--5 pesan per minggu.
2.  **Opt-in.** Nomor WhatsApp hanya disimpan setelah pengguna memberi
    persetujuan eksplisit; dapat dicabut kapan saja di Settings.
3.  **Normalisasi nomor** ke format internasional (`62…`) satu kali saat
    disimpan; nilai itulah yang dikirim ke gateway.
4.  **Jendela kirim** default 06:00--20:00 waktu lokal. Di luar jendela,
    pesan ditahan sampai jendela berikutnya.
5.  **Isi pesan tanpa data medis.** Hanya jenis aktivitas, hari/jam, dan
    ajakan membuka aplikasi. Dilarang menyertakan nyeri, diagnosis,
    kondisi kaki, atau catatan check-in apa pun.
6.  **Tanpa duplikasi.** Setiap pesan memakai `idempotency_key` unik;
    retry dibatasi (maksimal 3) dengan backoff dan tidak pernah mengirim
    ulang pesan yang sudah berstatus `sent`.
7.  **Throttle** minimal 2 detik antar pengiriman (akun WhatsApp
    pengguna bersifat pribadi).
8.  **Preflight** `GET /api/v1/session/status` sebelum mengirim. Jika
    tidak terhubung (`503` atau status disconnect), pesan menjadi
    `deferred`, banner "WhatsApp belum terhubung" tampil di Settings, dan
    pengingat tetap dikirim in-app.
9.  **Batas status.** Status yang didukung hanya `queued`, `sending`,
    `sent`, `failed`, `deferred`. Gateway tidak menyediakan status
    delivered/read, jadi aplikasi tidak boleh mengklaim pesan sudah
    dilihat.
10. **Fallback in-app** selalu aktif, terlepas dari status WhatsApp.
11. Endpoint `send-media` tidak digunakan pada MVP.

### 8.7 Settings

-   Profil.
-   Jadwal.
-   **Keamanan:** atur/ganti PIN, kunci sesi berjalan.
-   **Notifikasi:** opt-in nomor WhatsApp, jendela kirim, pengingat
    in-app, pengingat aktivitas ringan.
-   **Status WhatsApp:** indikator koneksi, tampilan QR pairing
    (`GET /api/v1/session/qr`), putuskan sambungan.
-   Satuan dan bahasa.
-   Ekspor data (JSON).
-   Hapus seluruh data beserta konsekuensinya.
-   Disclaimer dan informasi keselamatan.

------------------------------------------------------------------------

## 9. User Flow

``` mermaid
flowchart TD
    A[Open App] --> A0[Unlock PIN]
    A0 --> B[Dashboard]
    B --> C[Pre-workout Check: 6 pertanyaan]
    C --> D{Tier hasil?}
    D -->|Aman| E[View Workout]
    D -->|Ringan| F[Pilih: lanjut / lewati / batalkan]
    D -->|Berat atau Akut| F2[Stop & panduan tenaga kesehatan]
    F --> H[Simpan check-in + action_taken]
    F2 --> H
    E --> I[Start Session]
    I --> J[Exercise Instructions]
    J --> K[Perform and Record Set]
    K --> L{More exercises?}
    L -->|Yes| J
    L -->|No| M[Post-workout Check + RPE]
    M --> N[Save Session]
    N --> O[Update History and Progress]
    O --> P{Tier Akut/Berat?}
    P -->|Ya| Q[Jadwalkan Follow-up Check]
    P -->|Tidak| B
    Q --> B
```

------------------------------------------------------------------------

## 10. Halaman / Screen Requirements

  Screen               Isi utama
  -------------------- ----------------------------------------------
  Onboarding           Penjelasan aplikasi, disclaimer, profil awal
  Unlock               Pertanyaan PIN (sekali per sesi aplikasi)
  Profile              Data pengguna dan kondisi yang dilaporkan
  Dashboard            Jadwal dan sesi berikutnya, termasuk follow-up
  Weekly Schedule      Kalender dan pengaturan waktu
  Program Detail       Fase, minggu, Workout A/B
  Exercise Detail      Instruksi, target, dosis, cue keselamatan, regresi
  Active Session       Latihan aktif, set, repetisi, timer
  Pre-workout Check    Pemeriksaan kondisi (6 pertanyaan + tier)
  Post-workout Check   RPE, nyeri, kelelahan, gejala
  History              Riwayat latihan
  Progress             Tren konsistensi dan performa
  Settings             Profil, PIN, jadwal, notifikasi, status WhatsApp,
                       data, disclaimer

------------------------------------------------------------------------

## 11. Data Model

Rancangan entitas awal:

### User

-   id
-   display_name
-   birth_year atau age_band
-   gender (opsional)
-   timezone
-   locale (default `id-ID`)
-   created_at
-   updated_at

### Auth

Satu baris untuk MVP (single user).

-   user_id
-   pin_hash (argon2id; PIN tidak pernah disimpan plain)
-   failed_attempt_count
-   locked_until (nullable; 5 percobaan gagal → kunci 5 menit)
-   pin_changed_at
-   updated_at

Unlock menghasilkan token sesi yang berlaku **selama aplikasi terbuka**
(per sesi aplikasi, bukan timeout tetap); tutup/buka ulang aplikasi
berarti PIN ditanya lagi.

### UserPreference

-   user_id
-   wa_opt_in (boolean)
-   wa_number (nullable, sudah dinormalisasi `62…`)
-   wa_consent_at (nullable)
-   reminder_window_start (default `06:00`)
-   reminder_window_end (default `20:00`)
-   session_reminder_offset_minutes (default 30)
-   light_day_reminder_enabled
-   in_app_reminder_enabled
-   unit_system
-   updated_at

### UserCondition

-   id
-   user_id
-   condition_type
-   body_area
-   laterality
-   user_reported_status
-   notes
-   created_at
-   updated_at

### MuscleGroup

-   id
-   code
-   name
-   functional_goal
-   description

### Exercise

-   id
-   code
-   name (istilah Inggris)
-   name_id (alias Bahasa Indonesia untuk tampilan)
-   description
-   category
-   difficulty
-   equipment (array/relasi)
-   instructions (JSON)
-   safety_cues (JSON)
-   contraindication_notes (JSON)
-   regression_exercise_id (nullable)
-   progression_exercise_id (nullable)
-   clinical_review_status
-   content_version
-   is_active

### ExerciseTarget

-   exercise_id
-   muscle_group_id
-   target_role (primary/secondary)

### ExerciseMedia

-   id
-   exercise_id
-   media_type (image/video)
-   url
-   alt_text (wajib untuk gambar/video)
-   clinical_review_status
-   content_version
-   is_active

Media yang belum `reviewed` tidak ditampilkan pada sesi aktif.

### WorkoutProgram

-   id
-   name
-   duration_weeks
-   description
-   status

### ProgramPhase

-   id
-   program_id
-   phase_number
-   week_start
-   week_end
-   focus
-   progression_rules (JSON)

### WorkoutTemplate

-   id
-   program_id
-   phase_id
-   name
-   day_type
-   estimated_duration_minutes

### WorkoutTemplateExercise

-   id
-   template_id
-   exercise_id
-   order_index
-   sets_target
-   reps_min
-   reps_max
-   hold_seconds
-   rest_seconds
-   side_mode
-   notes

### WorkoutSchedule

-   id
-   user_id
-   day_role (workout_a / workout_b / light / rest)
-   weekday
-   local_time
-   timezone
-   reminder_enabled
-   is_active

`day_role` **tidak** menunjuk template tetap: saat sesi dibuka, backend
me-resolve `day_role` ke `WorkoutTemplate` pada fase aktif program
berjalan (lihat §13). Dengan begitu pergantian fase otomatis mengubah
isi sesi tanpa mengubah jadwal.

### WorkoutSession

-   id
-   user_id
-   template_id
-   day_role
-   program_id
-   phase_number (snapshot)
-   program_version (snapshot)
-   template_version (snapshot)
-   started_at
-   completed_at
-   status (draft/active/paused/completed/cancelled)
-   perceived_exertion (RPE 0--10; satu-satunya sumber RPE, §3.2)
-   notes

### ExerciseLog

-   id
-   session_id
-   exercise_id
-   exercise_content_version (snapshot)
-   set_number
-   reps_completed
-   hold_seconds_completed
-   rest_seconds_actual
-   skipped
-   skip_reason (wajib diisi bila skipped = true)
-   technique_self_rating

### BodyCheck

-   id
-   user_id
-   session_id (nullable)
-   check_type (pre/post/follow_up)
-   symptom_new_pain (boolean)
-   symptom_swelling (boolean)
-   symptom_instability (boolean)
-   symptom_unsafe_today (boolean)
-   symptom_acute_red_flag (boolean)
-   symptom_worsening (boolean)
-   tier (aman/ringan/berat/akut)
-   matched_rules (JSON array; aturan yang terpicu)
-   action_taken (proceed/skip_exercise/cancel_session/stop_session)
-   pain_score
-   fatigue_score
-   swelling
-   instability
-   notes
-   recorded_at

`tier` dan `matched_rules` dihitung dari tabel §3.1 di backend saat
disimpan, lalu disimpan apa adanya; tidak dihitung ulang saat riwayat
dibaca, dan tidak pernah diubah oleh klien.

### Reminder

-   id
-   user_id
-   schedule_id
-   reminder_type (session/light_day/follow_up)
-   enabled
-   local_time

### OutboundMessage

Antrian pengiriman WhatsApp ke gateway (`docs/reference/openapi.json`).

-   id
-   user_id
-   reminder_id (nullable)
-   kind (session_reminder/light_day_reminder/followup_check)
-   idempotency_key (unique)
-   to_number (sudah dinormalisasi `62…`)
-   body_template_code
-   body_text (tanpa data medis)
-   status (queued/sending/sent/failed/deferred)
-   retry_count (maksimal 3)
-   last_error_code (nullable)
-   scheduled_for (wajib berada di dalam jendela kirim)
-   sent_at (nullable)
-   created_at
-   updated_at

### 11.1 Enum terpusat

Semua enum didefinisikan di satu tempat (migrasi + kontrak API) dan
divalidasi backend: `session.status`, `day_role`, `exercise.category`,
`exercise.difficulty`, `exercise.side_mode`, `body_check.check_type`,
`body_check.tier`, `body_check.action_taken`, `reminder.reminder_type`,
`outbound_message.status`, `clinical_review_status`,
`equipment`.

### 11.2 Aturan dasar dan retensi

-   Foreign key, indeks pada kolom pencarian utama (`user_id`,
    `session_id`, `recorded_at`, `scheduled_for`), dan timestamps wajib.
-   Data kesehatan/keluhan adalah data sensitif: minimalkan
    pengumpulan, batasi akses, dan sediakan penghapusan serta ekspor.
-   `body_text` pada `OutboundMessage` tidak pernah berisi data medis
    (aturan §8.6.4); token gateway dan nomor tujuan tidak masuk ke log
    aplikasi (§12.1).
-   Retensi: `body_check` dan `exercise_log` ikut penghapusan data
    pengguna (dihapus total saat pengguna meminta hapus). Riwayat sesi
    dan check-in **tidak** diarsipkan otomatis selama program berjalan;
    `outbound_message` yang berstatus `sent` dipertahankan 90 hari untuk
    audit, lalu `body_text` dihapus (metadata status dipertahankan 90
    hari juga).
-   Backup mengikuti §14 (terenkripsi, akses dibatasi).

------------------------------------------------------------------------

## 12. API Specification (Draft)

Prefix: `/api/v1`

  Method   Endpoint                Kegunaan
  -------- ----------------------- --------------------------------
  POST     `/auth/bootstrap`       Atur PIN pertama (sekali, saat onboarding)
  POST     `/auth/unlock`          Buka sesi dengan PIN
  POST     `/auth/pin`             Ganti PIN
  POST     `/auth/lock`            Kunci sesi berjalan
  GET      `/me`                   Profil pengguna
  PATCH    `/me`                   Perbarui profil
  GET      `/me/export`            Ekspor seluruh data (JSON)
  DELETE   `/me`                   Hapus seluruh data pengguna
  GET      `/conditions`           Daftar kondisi yang dilaporkan
  PUT      `/conditions`           Perbarui kondisi
  GET      `/preferences`          Preferensi (notifikasi, jendela kirim, satuan)
  PUT      `/preferences`          Perbarui preferensi
  GET      `/programs/active`      Program aktif
  GET      `/programs/{id}`        Detail program
  GET      `/workouts/today`       Sesi terjadwal hari ini (sudah di-resolve per fase)
  GET      `/workouts/{id}`        Detail template workout
  POST     `/sessions`             Mulai sesi
  GET      `/sessions/{id}`        Detail sesi
  PATCH    `/sessions/{id}`        Pause/selesaikan sesi
  POST     `/sessions/{id}/logs`   Simpan log latihan
  POST     `/body-checks`          Simpan check-in (pre/post/follow-up)
  GET      `/body-checks`          Riwayat check-in
  GET      `/history`              Riwayat sesi
  GET      `/progress/summary`     Ringkasan perkembangan
  GET      `/schedule`             Jadwal pengguna
  PUT      `/schedule`             Perbarui jadwal
  GET      `/reminders`            Daftar pengingat terjadwal
  PUT      `/reminders`            Perbarui pengingat
  GET      `/notifications/outbox` Status pengiriman WhatsApp (paginated)
  GET      `/notifications/wa/status`  Status koneksi WhatsApp (proxy gateway)
  POST     `/notifications/wa/qr`  Ambil QR pairing (proxy gateway)
  POST     `/notifications/wa/unlink`  Putuskan sambungan WhatsApp
  GET      `/exercises`            Katalog latihan
  GET      `/exercises/{id}`       Detail latihan

Endpoint gateway WhatsApp **tidak pernah** dipanggil dari browser: selalu
lewat backend, agar token gateway tidak berada di sisi klien dan tidak
masuk ke log (§12.1).

### 12.1 API conventions

-   JSON request/response.
-   ISO 8601 timestamp dengan timezone.
-   Validasi request di backend.
-   Error response konsisten (`code`, `message`, `details`).
-   Pagination untuk `history`, `progress`, `notifications/outbox`, dan
    `body-checks`.
-   Header `Idempotency-Key` wajib pada: `POST /sessions`,
    `POST /sessions/{id}/logs`, `POST /body-checks`, dan `POST
    /auth/pin`. Nilai disimpan minimal 24 jam; request dengan key sama
    mengembalikan hasil pertama, bukan memproses ulang.
-   Autentikasi: sesi unlock PIN menghasilkan token pendek (aturan
    keamanan lihat §14); semua resource terikat `user_id` pengguna yang
    sedang aktif. Tidak ada endpoint yang membaca data pengguna lain
    (relevan bila kelak menjadi multi-user).
-   Jangan memasukkan informasi medis sensitif ke log aplikasi, termasuk
    `notes`, `pain_score`, dan `body_text` pesan.

------------------------------------------------------------------------

## 13. Arsitektur Teknis

### Frontend

-   React
-   TypeScript
-   Vite
-   Tailwind CSS
-   React Router
-   TanStack Query
-   Zod untuk validasi form
-   PWA service worker untuk instalasi dan caching aset statis

### Backend

-   Go
-   Chi (router + middleware eksplisit untuk auth PIN dan
    `Idempotency-Key`)
-   PostgreSQL
-   SQL migrations
-   Repository/service/handler separation
-   Structured logging
-   Request validation
-   OpenAPI untuk kontrak API
-   Worker terpisah (`cmd/worker`) untuk menjadwalkan dan mengirim
    pengingat WhatsApp

### Integrasi WhatsApp

-   Klien gateway dibungkus di balik interface `NotificationSender`
    dengan satu implementasi `WAGatewaySender`.
-   Kontrak gateway: `docs/reference/openapi.json`.
-   Seluruh konfigurasi --- termasuk `wa_gateway.base_url` dan
    `wa_gateway.token` --- dibaca dari **`config.json`**, bukan `.env`.
    `config.json` tidak ikut di-commit; sediakan `config.example.json`
    berisi placeholder. Nilai itu tidak pernah dikirim ke klien dan tidak
    pernah ditulis ke log.
-   Preflight `GET /api/v1/session/status`; jika disconnect → status
    `deferred`, banner di Settings, fallback in-app (§8.6).
-   Kirim lewat transaksi: tulis `OutboundMessage` (`queued`) → worker
    memproses dengan throttle 2 detik → perbarui status. Retry maksimal
    3 dengan backoff; `sent` tidak pernah dikirim ulang.
-   Jendela kirim 06:00--20:00 menurut `timezone` pengguna.
-   QR pairing dan status koneksi hanya diakses lewat backend.

### Offline

-   Service worker meng-cache aplikasi shell, katalog latihan, dan
    program terakhir yang berhasil dimuat (stale-while-revalidate).
-   Log sesi, check-in, dan jawaban pre-check ditulis lebih dulu ke
    IndexedDB lalu disinkronkan saat daring; antrean memakai
    `client_uuid` unik agar aman dari pengiriman ganda.
-   Tabel aturan tier §3.1 ikut dikirim dalam payload program dan ikut
    di-cache, sehingga keputusan check-in tetap berfungsi offline.
-   Aplikasi menampilkan status sinkronisasi; data dianggap tersimpan
    setelah konfirmasi server, bukan hanya setelah tulis lokal.

### Deployment

-   Docker
-   Docker Compose untuk local development (Postgres + API + worker +
    gateway WhatsApp dalam satu compose, saling terhubung via jaringan
    internal)
-   Konfigurasi via `config.json` (tanpa `.env`)
-   Akses lokal `localhost`; HTTPS dan backup terjadwal menyusul setelah
    MVP (lihat §18)

### Prinsip arsitektur

-   Pisahkan katalog latihan, template program, jadwal, dan log aktual.
-   Program dan katalog harus versioned agar perubahan konten tidak
    mengubah riwayat sesi lama; sesi dan log menyimpan snapshot versi
    (§11).
-   Jangan hard-code jadwal dan dosis ke UI; `day_role` di-resolve ke
    template fase aktif di backend (§11).
-   Frontend menampilkan instruksi yang berasal dari backend.
-   Aturan keselamatan dasar harus tetap berlaku meski aplikasi offline;
    jangan mengandalkan AI atau jaringan untuk keputusan keselamatan.
-   Aturan tier §3.1 disimpan sebagai data yang versioned dan hanya boleh
    diubah lewat migrasi/review, bukan lewat UI.
-   Backend menolak penambahan set/repetisi di luar aturan §5.1: tidak
    ada endpoint yang menaikkan dosis otomatis.

------------------------------------------------------------------------

## 14. Non-Functional Requirements

### Usability

-   Mobile-first dan responsif.
-   Tombol besar dan mudah disentuh.
-   Instruksi singkat, jelas, dan mudah dibaca saat latihan.
-   Kontras warna memadai.
-   Timer terlihat jelas.

### Performance

-   Dashboard siap interaksi di bawah 2,5 detik pada koneksi 4G
    (metrik Lighthouse: LCP ≤ 2,5 dtk, TBT ≤ 300 ms).
-   Aset gambar/video dioptimalkan.
-   Tidak memutar video otomatis.

### Reliability

-   Log sesi tidak hilang saat halaman ter-refresh.
-   Log sesi tetap tersimpan walau koneksi putus di tengah latihan
    (antrean lokal → sinkron saat daring, §13 Offline).
-   Sesi yang belum selesai dapat dilanjutkan atau dibatalkan dengan
    konfirmasi.
-   Timer tidak menjadi satu-satunya sumber kebenaran durasi latihan.
-   Pengiriman WhatsApp boleh gagal tanpa menghilangkan pengingat
    in-app; kegagalan gateway tidak pernah menunda pemeriksaan keselamatan.

### Privacy & Security

-   HTTPS wajib begitu diakses di luar localhost (untuk MVP lokal tetap
    HTTP `localhost`, lihat §18).
-   PIN di-hash dengan argon2id; rate-limit percobaan (5 percobaan →
    kunci 5 menit) dan unlock hanya berlaku selama aplikasi terbuka.
-   Token sesi mengikuti umur sesi aplikasi; `wa_gateway.token` tidak
    pernah sampai ke klien.
-   Authorization pada setiap resource pengguna.
-   Data sensitif tidak ditampilkan pada analytics pihak ketiga tanpa
    persetujuan.
-   Pengguna dapat mengekspor (JSON) dan menghapus data.
-   Pesan WhatsApp tidak memuat data medis (§8.6.4); nomor telepon hanya
    disimpan dengan persetujuan dan dapat dihapus lewat Settings.
-   Backup dienkripsi dan akses dibatasi.

### Accessibility

-   Navigasi keyboard.
-   Label untuk kontrol.
-   Jangan hanya menggunakan warna untuk menunjukkan status.
-   Dukungan ukuran teks yang nyaman.
-   Alternatif teks untuk gambar demonstrasi.

------------------------------------------------------------------------

## 15. MVP Acceptance Criteria

MVP dinyatakan siap diuji ketika:

1.  Pengguna dapat melihat dan mengubah profil awal.
2.  Program 12 minggu dan fase latihan dapat ditampilkan.
3.  Jadwal default Senin/Kamis pukul 07:00 dapat diubah.
4.  Pengguna dapat membuka detail setiap latihan.
5.  Sesi menampilkan set, repetisi, cue, dan timer istirahat.
6.  Pengguna dapat mencatat repetisi aktual dan latihan yang dilewati.
7.  Pre-workout check (6 pertanyaan §3.1) muncul sebelum sesi dan tier
    dihitung dari tabel aturan, bukan skor.
8.  Gejala yang dilaporkan memicu panduan lewati/batalkan/hentikan pada
    tier sesuai §3.1, dan tidak pernah menawarkan peningkatan latihan.
9.  Post-workout check dapat disimpan.
10. Riwayat sesi dapat dilihat.
11. Progress menampilkan metrik aktual tanpa klaim klinis dan tanpa
    metrik "koreksi flat foot".
12. Data tetap tersimpan setelah reload.
13. Pengguna dapat mengubah jadwal dan pengingat, termasuk opt-in/out
    nomor WhatsApp dan jendela kirim.
14. Konten latihan memiliki status review dan versi; konten berstatus
    `pending_review` tidak bisa ditampilkan sebagai program aktif.
15. Disclaimer dan informasi keselamatan tersedia.
16. Setiap pertanyaan gejala menghasilkan tier dan tindakan yang benar
    (diuji otomatis untuk seluruh kombinasi §3.1).
17. Tidak ada endpoint atau alur yang menaikkan set/repetisi otomatis
    (dibuktikan dengan pengujian).
18. Aplikasi terkunci meminta PIN; rate-limit dan lockout berfungsi.
19. Pengingat WhatsApp terkirim sesuai jadwal dalam jendela kirim, tanpa
    duplikat saat retry, dan jatuh ke `deferred` ketika gateway tidak
    terhubung.
20. Sesi yang berjalan offline dapat dilanjutkan dan tersinkron tanpa
    duplikat setelah kembali daring.
21. Ekspor JSON dan penghapusan data berfungsi serta menghapus
    `body_check` dan `outbound_message` terkait.
22. Media latihan menampilkan `alt_text` dan media `pending_review`
    tidak muncul di sesi aktif.

------------------------------------------------------------------------

## 16. Roadmap Pengembangan

### Milestone 1 --- Foundation

-   Setup repository frontend dan backend (`config.example.json`,
    docker-compose).
-   Setup PostgreSQL dan migrations.
-   Onboarding + PIN (bootstrap, unlock, rate-limit, lockout).
-   Implementasi profil pengguna.
-   Implementasi katalog kelompok otot, latihan (`name` + `name_id`),
    dan ilustrasi SVG beserta `alt_text` (status `pending_review`).

### Milestone 2 --- Program & Schedule

-   Seed program 12 minggu.
-   Workout template A/B per fase + resolve `day_role`.
-   Jadwal dan preferensi waktu.
-   Dashboard.

### Milestone 3 --- Workout Session

-   Active session UI.
-   Set/repetition logging dengan snapshot versi konten.
-   Rest timer.
-   Pause, resume, skip, complete.
-   Antrean log offline (IndexedDB) dan sinkronisasi.

### Milestone 4 --- Safety Check

-   Pre-workout check 6 pertanyaan + tabel tier §3.1 (data, offline).
-   Post-workout check dan follow-up terjadwal.
-   Safety messaging dan teks darurat tier Akut.
-   Catatan gejala.

### Milestone 5 --- Progress & Notifications

-   Workout history.
-   Weekly consistency dan perbandingan per `content_version`.
-   Ekspor dan hapus data.
-   Interface `NotificationSender` + adapter `WAGatewaySender`
    (`docs/reference/openapi.json`), worker, outbox, QR pairing, dan
    fallback in-app.

### Milestone 6 --- QA & Release

-   Responsive testing.
-   Accessibility checks (termasuk `alt_text` media).
-   Unit test aturan tier keselamatan, PIN, dan aturan progression.
-   API/security testing, pengujian retry & idempotency notifikasi.
-   **Prasyarat rilis:** review konten latihan oleh fisioterapis →
    seluruh seed dan media di-flip dari `pending_review` ke `reviewed`.
    Tanpa langkah ini program tetap terkunci (gate §15 butir 14).
-   Deployment PWA (Docker lokal).

------------------------------------------------------------------------

## 17. Risiko dan Hal yang Perlu Divalidasi

  -----------------------------------------------------------------------
  Risiko                              Mitigasi
  ----------------------------------- -----------------------------------
  Latihan tidak cocok untuk kondisi   Review fisioterapis dan kemampuan
  individu                            menonaktifkan latihan

  Pengguna memaksakan target          Tidak ada streak yang menghukum;
                                      dukung skip/rest

  Instruksi visual salah              Gunakan media yang ditinjau tenaga
                                      kompeten

  Progression terlalu cepat           Aturan progression konservatif dan
                                      dapat ditahan

  Pengguna menganggap aplikasi alat   Disclaimer yang jelas
  diagnosis                           

  Data kondisi tubuh bocor            Minimalisasi data, authorization,
                                      enkripsi, penghapusan

  Gateway WhatsApp tidak terhubung    Preflight status, pesan `deferred`,
                                      fallback in-app, banner Settings

  Akun WhatsApp pribadi kena batas/   Throttle 2 detik, jendela kirim,
  pembatasan                          pesan singkat tanpa data medis

  Pengguna menganggap notifikasi      Teks hanya ajakan membuka aplikasi,
  sudah dibaca                        status berhenti di `sent`

  Duplikasi pesan saat retry          `idempotency_key` unik + batas
                                      retry + state machine status
  -----------------------------------------------------------------------

Sebelum program dipakai secara rutin, minta fisioterapis yang memahami
hypermobility untuk meninjau pilihan gerakan, teknik, dosis, dan
modifikasi khusus untuk lutut, pergelangan kaki, pergelangan tangan,
jari, serta bilateral flat foot.

------------------------------------------------------------------------

## 18. Status Keputusan

### Sudah diputuskan

1.  Platform: PWA (React + TypeScript + Vite, Go + Chi, PostgreSQL).
2.  Penggunaan: personal/single-user/single-device; autentikasi PIN lokal
    (argon2id), unlock berlaku selama aplikasi terbuka, lockout 5 gagal →
    5 menit.
3.  Konfigurasi: `config.json` (termasuk base URL dan token gateway
    WhatsApp); tanpa `.env`.
4.  Waktu latihan default 07:00 WIB; hari default Senin dan Kamis,
    aktivitas ringan Rabu dan Sabtu (dapat diedit pengguna).
5.  Pengingat: WhatsApp via gateway eksternal (kontrak
    `docs/reference/openapi.json`) untuk **seluruh hari terjadwal** plus
    follow-up check; opt-in, jendela kirim 06:00--20:00, fallback in-app.
6.  Offline: cache baca + antrean log lokal (§13 Offline).
7.  Bahasa konten: Indonesia (`id-ID`); nama latihan tetap Inggris +
    alias Indonesia (`name_id`).
8.  Media latihan: ilustrasi vektor SVG + `alt_text`.
9.  Deployment MVP: Docker lokal di `localhost`; HTTPS dan backup
    terjadwal ditunda (mutasi jadi kebutuhan begitu diakses dari luar
    localhost).
10. Gate review konten: tetap berlaku; review fisioterapis menjadi
    **prasyarat rilis di Milestone 6** (§15 butir 14).
11. Sinkronisasi lintas perangkat dan mode multi-user: **di luar
    cakupan MVP**, dicatat eksplisit di §1.2 agar tidak menutup jalur
    migrasi (seluruh data sudah terikat `user_id`).

### Masih terbuka

1.  Waktu dan hasil review fisioterapis (siapa, kapan, bentuk
    artefaknya) --- satu-satunya prasyarat rilis yang tersisa.
2.  Isi final pesan WhatsApp (template bahasa) dan apakah perlu pesan
    tambahan di luar empat hari terjadwal.
3.  Kemungkinan pengembangan menjadi aplikasi multi-user plus
    sinkronisasi lintas perangkat (membutuhkan autentikasi selain PIN
    lokal).
4.  Host/URL gateway WhatsApp final (komposisi awal mengasumsikan satu
    jaringan docker-compose).

**Dokumen ini adalah spesifikasi produk dan teknis MVP, bukan resep
medis individual.**
