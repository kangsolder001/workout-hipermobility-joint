# Plan — Frontend (SvelteKit + Svelte 5 + TypeScript + Tailwind + PWA)

Baca dulu `docs/plan/README.md` (aturan main & DoD global).
Rujukan spec: `docs/spec_workout.md` v1.3 — terutama §8 (fitur),
§9 (user flow), §10 (layar), §14 (NFR).

Bahasa UI: **Indonesia**. Nama latihan: Inggris + alias Indonesia
(`name_id`).

---

## F01 — Scaffold frontend & design system

**Spec:** §13 (Frontend), §14 Usability/Accessibility

**Scope:** `frontend/` (SvelteKit + `adapter-static`, Vite, Tailwind,
Zod, PWA), `frontend/public/`

**Kerjakan:**
- SvelteKit + Svelte 5 (runes) + TS + Tailwind + Zod; build sebagai SPA
  statis lewat `adapter-static` (hasil build berupa folder statis).
- Routing berbasis file (`src/routes/**/+page.svelte`) — tanpa router
  pihak ketiga.
- PWA: manifest + service worker (`@vite-pwa/sveltekit` atau
  `vite-plugin-pwa`) — minimal cache shell.
- Komponen dasar tombol besar, timer, kartu latihan, kontras memadai;
  navigasi keyboard + label untuk kontrol (§14 Accessibility).
- Struktur folder sesuai `docs/plan/README.md` §2.
- Proxy dev ke backend + baca konfigurasi lewat build config (tanpa
  `.env`; gunakan `config`/`import.meta.env` minimal — utamakan server
  yang menyediakan base URL via `/api/v1/config` bila perlu).

**Acceptance:**
- [ ] `npm run dev` + `npm run build` sukses (build `adapter-static`
      menghasilkan folder `build/` tanpa error); `npm run lint` hijau.
- [ ] Lighthouse PWA minimal "installable"; tidak ada autoplay video.
- [ ] Halaman bisa dinavigasi keyboard; teks tetap terbaca pada zoom 200%.
- [ ] Tidak ada dependensi React / router pihak ketiga / library data
      pihak ketiga di `package.json`.

**Depends:** —

---

## F02 — API client, tipe, dan error handling

**Spec:** §12, §12.1, §13 (Zod, client fetch berbasis rune)

**Scope:** `frontend/src/lib/api/`

**Kerjakan:**
- Client dari `backend/api/openapi.yaml` (codegen bila memungkinkan;
  jika tidak, tulis tangan tapi wajib sinkron).
- Zod schema untuk request/response utama.
- Lapisan data: client `fetch` + cache sederhana berbasis rune
  (`$state`/`$derived`) berupa helper `createQuery`/`createMutation`
  yang mengurus retry, error `code/message/details`, token sesi, dan
  header `Idempotency-Key` untuk endpoint wajib — **tanpa** TanStack
  Query atau library data sejenis.
- Penanganan 401 (kembali ke layar PIN) dan 423/429 (pesan lockout).

**Acceptance:**
- [ ] Semua endpoint §12 terpakai oleh client (ada wrapper-nya).
- [ ] Error backend ditampilkan sebagai pesan yang mengerti user (bukan
      stack/JSON mentah).
- [ ] Unit test schema validasi (masukkannya ke T06).

**Depends:** B12 (bisa dimulai dari draft spec)

---

## F03 — Onboarding & layar PIN

**Spec:** §10 (Onboarding, Unlock), §8.7, §11 `Auth`, §12 `/auth/*`

**Scope:** `frontend/src/routes/{onboarding,unlock}/+page.svelte`,
`frontend/src/lib/features/auth/`

**Kerjakan:**
- Onboarding: penjelasan aplikasi, **disclaimer keselamatan**, pengaturan
  profil awal (§2.1), bootstrap PIN.
- Unlock: minta PIN tiap kali aplikasi dibuka; pesan lockout (5 gagal →
  5 menit) dengan hitung mundur; ganti PIN di Settings.

**Acceptance:**
- [ ] App tertutup/buka ulang → PIN ditanya lagi.
- [ ] Lockout tampil dengan sisa waktu, tombol tidak bisa dipakai.
- [ ] Disclaimer tidak bisa dilewati tanpa konfirmasi.

**Depends:** F01, F02, B04

---

## F04 — Profil & kondisi

**Spec:** §8.1, §10 (Profile)

**Scope:** `frontend/src/routes/profile/+page.svelte`,
`frontend/src/lib/features/profile/`

**Kerjakan:** Lihat/edit profil, kondisi yang dilaporkan user (per area
sendi + laterality), area sendi yang perlu diperhatikan, peralatan,
tujuan, hari/waktu/zona waktu. Validasi Zod + pesan error jelas.

**Acceptance:**
- [ ] Perubahan tersimpan dan bertahan setelah reload.
- [ ] Tidak ada field yang meminta diagnosis medis (hanya "dilaporkan
      pengguna").

**Depends:** F02, B05

---

## F05 — Dashboard, jadwal mingguan, detail program

**Spec:** §8.2, §4, §10 (Dashboard, Weekly Schedule, Program Detail)

**Scope:** `frontend/src/routes/+page.svelte` (Dashboard),
`frontend/src/routes/{schedule,program/[id]}/+page.svelte`,
`frontend/src/lib/features/schedule/`

**Kerjakan:**
- Dashboard: sesi berikutnya, hari/jam, fase & minggu, durasi estimasi,
  ringkasan minggu ini, tombol mulai, **peringatan check-in belum
  selesai**, pekerjaan *follow-up* bila ada, banner status WhatsApp.
- Jadwal: edit hari/jam per `day_role` + pengingat.
- Program detail: 6 fase, minggu berjalan, Workout A/B, latihan
  (dengan regresi).

**Acceptance:**
- [ ] Mengubah jadwal mempengaruhi "hari ini" tanpa reload penuh.
- [ ] `day_role` di-resolve backend (frontend tidak menghitung fase).
- [ ] Tidak ada elemen "streak" yang menghukum (§17).

**Depends:** F02, B06

---

## F06 — Detail latihan

**Spec:** §6, §10 (Exercise Detail)

**Scope:** `frontend/src/routes/exercises/[id]/+page.svelte`,
`frontend/src/lib/features/exercise/`

**Kerjakan:** Instruksi langkah demi langkah, target otot fungsional,
dosis (set/reps/hold/rest), **cue keselamatan menonjol**, kesalahan
umum, **regresi & progresi**, ilustrasi SVG + `alt_text`, penanda status
review konten, nama `name` + `name_id`.

**Acceptance:**
- [ ] Cue keselamatan terlihat tanpa scroll panjang di mobile.
- [ ] Media `pending_review` tetap tampil hanya jika program sudah
      aktif (ikuti guard B14) dan selalu punya `alt_text`.
- [ ] Tidak ada teks klaim medis/koreksi flat foot.

**Depends:** F02, B03 (data) / B06 (endpoint)

---

## F07 — Sesi latihan aktif

**Spec:** §8.3, §9, §14 Reliability

**Scope:** `frontend/src/lib/features/session/`,
`frontend/src/routes/session/[id]/+page.svelte`

**Kerjakan:**
- Satu latihan pada satu waktu: instruksi, media, target, dosis, cue.
- Tandai set selesai + edit repetisi aktual; timer istirahat.
- Pause/resume/akhiri dengan konfirmasi; **lewati latihan dengan alasan
  wajib**.
- Sesi tidak dianggap selesai hanya karena timer habis (konfirmasi
  eksplit).
- Semua tulisan masuk antrean lokal dulu (integrasi F11) dengan indikator
  sinkronisasi.

**Acceptance:**
- [ ] Refresh di tengah sesi → sesi & log yang sudah tercatat pulih.
- [ ] Tanpa konfirmasi user, sesi tidak berstatus `completed`.
- [ ] Tombol akses keyboard dan target sentuh besar (≥44px).

**Depends:** F02, F11 (bisa paralel dengan kontrak lokal), B07

---

## F08 — Alur check-in pra/pasca latihan

**Spec:** §3.1, §3.2, §9, §8.4, §10 (Pre/Post-workout Check)

**Scope:** `frontend/src/lib/safety/`,
`frontend/src/routes/check/{pre,post}/+page.svelte`

**Kerjakan:**
- Pre-check: 6 pertanyaan ya/tidak; hasil tier diambil dari **payload
  safety rules** (murni data, bukan logika UI yang diduplikasi); layar
  per tier: Aman → lanjut; Ringan → konfirmasi lanjut/lewati/batalkan
  (dengan konfirmasi eksplisit bila tetap lanjut); Berat → stop + anjuran
  evaluasi; **Akut → layar darurat statis** (teks sama persis, tombol
  hanya "hentikan").
- Post-check: nyeri, kelelahan, bengkak, instabilitas, catatan + RPE
  0–10 (RPE disimpan ke sesi).
- Tampilan status "belum tersinkron" bila offline.

**Acceptance:**
- [ ] Tidak ada skor otomatis di UI; tier datang dari backend/payload.
- [ ] Layar Akut tidak bisa dilewati dan tidak menawarkan latihan.
- [ ] Semua 6 pertanyaan tampil sebelum sesi; jawaban tersimpan dengan
      `action_taken`.
- [ ] Dicakup test T06/T07.

**Depends:** F02, B08

---

## F09 — Riwayat & progress

**Spec:** §8.5, §10 (History, Progress)

**Scope:** `frontend/src/routes/{history,progress}/+page.svelte`

**Kerjakan:** Kalender latihan, riwayat sesi + log (reps aktual, RPE),
grafik konsistensi mingguan, perbandingan performa per latihan
(menghormati pemisahan `content_version`, tampilkan penanda versi),
ringkasan kondisi yang dilaporkan user, riwayat check-in + follow-up.

**Acceptance:**
- [ ] Tidak ada persentase kekuatan, klaim klinis, atau metrik koreksi
      flat foot (§6.5).
- [ ] Grafik tidak menyatukan dua versi konten tanpa penanda.
- [ ] Empty state jelas ketika belum ada data.

**Depends:** F02, B09

---

## F10 — Settings (PIN, notifikasi, WhatsApp, data)

**Spec:** §8.7, §12 (`/preferences`, `/notifications/*`, `/me/export`,
`DELETE /me`)

**Scope:** `frontend/src/routes/settings/+page.svelte`,
`frontend/src/lib/features/settings/`

**Kerjakan:**
- Profil, jadwal, satuan/bahasa.
- Keamanan: ganti PIN, kunci sesi.
- Notifikasi: opt-in nomor WhatsApp (normalisasi + konfirmasi), jendela
  kirim, pengingat in-app & hari ringan.
- **Status WhatsApp**: indikator koneksi, **tampil QR pairing**
  (dari `POST /notifications/wa/qr`), tombol putuskan sambungan, banner
  "WhatsApp belum terhubung" + fallback in-app.
- Ekspor data (unduh JSON) + hapus seluruh data dengan konfirmasi ganda.
- Disclaimer & informasi keselamatan.

**Acceptance:**
- [ ] Opt-out langsung menghentikan pengiriman (dicek terhadap B10).
- [ ] QR di-refresh bila kedaluwarsa; token gateway tidak pernah tampil.
- [ ] Hapus data membutuhkan konfirmasi dan menampilkan konsekuensi.

**Depends:** F02, B05, B10, B11

---

## F11 — Runtime offline (SW cache + antrean log)

**Spec:** §13 (Offline), §14 Reliability, §15 butir 20

**Scope:** `frontend/src/runtime/offline/`, service worker config

**Kerjakan:**
- Cache stale-while-revalidate untuk shell, katalog, program terakhir
  (termasuk payload `safety_rules`).
- Antrean lokal IndexedDB untuk log sesi, check-in, jawaban pre-check;
  `client_uuid` unik per catatan; retry saat daring; indikator
  "menunggu sinkronisasi".
- Jangan pernah menandai data tersimpan sebelum konfirmasi server.

**Acceptance:**
- [ ] Mode offline: buka app → lihat sesi terakhir → latihan → log
      tercatat → setelah daring terkirim **tanpa duplikat**.
- [ ] Dua sync berturut-turut tidak menggandakan baris (server idempotent).
- [ ] Keputusan check-in tetap berfungsi offline (payload rules
      ter-cache).
- [ ] Dicakup T06 & T07.

**Depends:** F01, F07, F08, B07, B08
