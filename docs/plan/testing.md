# Plan — Testing & QA

Baca dulu `docs/plan/README.md` (aturan main & DoD global).
Tujuan: membuktikan klaim "aman" di `docs/spec_workout.md` lewat otomasi,
bukan pemeriksaan manual.

Prinsip: **aturan keselamatan adalah bagian dari kontrak test** — jika
spec §3.1/§5.1/§6.5 dilanggar oleh kode, test harus gagal.

---

## T01 — Unit test aturan tier check-in (KRITIS)

**Spec:** §3.1 (tabel 4 tier), §3.2, §15 butir 16

**Scope:** `backend/internal/safety/*_test.go`,
`backend/seed/safety_rules_test.go`, fixture JSON

**Kerjakan:**
- Tabel test **64 kombinasi** 6 jawaban ya/tidak → tier yang diharapkan +
  aturan yang terpicu + tindakan.
- Verifikasi "tier tertinggi menang", tier Akut menghasilkan teks
  darurat konstan, dan tier dasar = Aman hanya jika semua `tidak`.
- Test bahwa tier berasal dari data `safety_rules` (ubah fixture → hasil
  berubah, membuktikan bukan hard-code).
- Snapshot teks darurat tiap tier.

**Acceptance:**
- [ ] 64/64 kombinasi tercakup dan hijau.
- [ ] Gagal jika seseorang menambah "skor otomatis" (uji: tambahkan aturan
      illegal di fixture → validasi menolak).

**Depends:** B02, B03, B08 (bisa dimulai sejak B08 ada inti murninya)

---

## T02 — Unit test penegakan dosis & anti auto-progression

**Spec:** §5.1, §6.5, §13 (prinsip arsitektur), §15 butir 17

**Scope:** `backend/internal/service/guards_test.go`

**Kerjakan:**
- Tolak penambahan set/reps/hold di luar batas fase (tabel §5.1 +
  isometric 2×5 detik, maks 3×8 detik).
- Tolak peningkatan ganda (set + durasi bersamaan).
- Buktikan tidak ada endpoint/scheduler yang menaikkan dosis sendiri
  (uji: jalankan siklus "minggu berikutnya" → dosis template tidak
  berubah tanpa aksi user).
- Guard konten `pending_review` tidak bisa jadi program aktif.
- Guard allow-list field progress: tidak ada metrik koreksi flat foot /
  klaim klinis (uji pemilihan field respons).

**Acceptance:**
- [ ] Semua kasus penolakan menghasilkan error dengan `code` jelas.
- [ ] Test hijau dan gagal bila guard dihapus.

**Depends:** B03, B06, B07, B14

---

## T03 — Unit test auth, idempotency, dan redaksi log

**Spec:** §11 `Auth`, §12.1, §14 Privacy, §15 butir 18

**Scope:** `backend/internal/auth/*_test.go`,
`backend/internal/logging/*_test.go`

**Kerjakan:**
- Argon2id hash/verify, PIN tidak plain (assert substring hash ≠ PIN).
- Rate-limit: 5 gagal → kunci 5 menit; hitung mundur; unlock setelah
  kedaluwarsa.
- Token sesi kedaluwarsa → 401; `POST /auth/lock` langsung menolak.
- `Idempotency-Key`: dua request sama → satu efek + respons identik;
  key lewat 24 jam → diproses ulang.
- Redaksi log: kirim payload dengan `notes`, `pain_score`, `body_text`,
  nomor, token → assert tidak ada di output log (termasuk saat error).

**Acceptance:**
- [ ] Semua skenario hijau; tidak ada `t.Log` yang membocorkan nilai
      sensitif.

**Depends:** B04, B13

---

## T04 — Integration test API + database

**Spec:** §11, §12 seluruhnya

**Scope:** `backend/internal/repository/*_test.go`,
`backend/internal/handler/*_test.go`, harness test (testcontainers /
compose Postgres terpisah)

**Kerjakan:**
- Migrasi dari kosong → `down` → `up` bersih.
- Seed idempoten (jalankan 2× → jumlah baris sama).
- Endpoint utama: `/me`, `/conditions`, `/preferences`,
  `/programs/active`, `/workouts/today` (resolve fase),
  `/sessions` + `/logs` (snapshot versi, `skip_reason` wajib,
  `client_uuid` idempoten), `/body-checks`, `/history`,
  `/progress/summary`, `/reminders`, `/notifications/outbox`,
  `/me/export`, `DELETE /me`.
- State machine sesi (transisi ilegal → 409/400).
- Authorization: permintaan dengan sesi user A ke resource user B → 404/403.

**Acceptance:**
- [ ] Test jalan di CI lokal dengan satu perintah (`make test-integration`
      atau skrip), membersihkan data setelah selesai.
- [ ] Tidak ada test yang bergantung pada data hasil manual.

**Depends:** B02–B11

---

## T05 — Unit/integration test worker pengingat WhatsApp

**Spec:** §8.6 (11 aturan), §13 (Integrasi WhatsApp), §17 risiko WA,
§15 butir 19

**Scope:** `backend/internal/notify/*_test.go`,
`backend/cmd/worker/*_test.go`, mock gateway (httptest)

**Kerjakan:**
- State machine `queued → sending → sent | failed | deferred`.
- Preflight disconnect/503 → `deferred` (bukan hilang), dan reminder
  in-app tetap ada.
- Retry maks 3 + backoff; **tidak pernah mengirim ulang `sent`**
  (assert jumlah panggilan mock = 1).
- `idempotency_key` unik per pesan; dua worker berjalan paralel tidak
  menghasilkan pesan ganda (advisory lock / row claim).
- Jendela kirim 06:00–20:00 (kasus: dijadwalkan 03:00 → dikirim 06:00).
- Throttle ≥ 2 detik antar kirim (assert timing).
- Template pesan **tanpa data medis**: kirim seluruh jenis pesan ke
  matcher yang melarang kata nyeri/diagnosis/flat foot/catatan check-in.
- `wa_gateway.token` tidak muncul di log maupun respons API.

**Acceptance:**
- [ ] Seluruh skenario hijau tanpa koneksi internet (mock).
- [ ] Test gagal jika aturan jendela/komposisi pesan diubah sembarangan.

**Depends:** B10

---

## T06 — Unit test frontend (Vitest)

**Spec:** §8, §13 (Offline), §15 butir 20

**Scope:** `frontend/src/**/*.test.ts(x)`

**Kerjakan:**
- Konsumsi payload safety rules → render tier yang benar per kasus
  (sample mewakili 4 tier, sisanya dicakup lewat test loop terhadap
  fixture yang sama dengan T01).
- Antrean offline: enqueue → flush → tidak duplikat → status sinkron.
- Zod schema request/response (termasuk validasi 0–10 skor, enum).
- Timer sesi: istirahat berjalan, tidak menandai sesi selesai sendiri.
- Komponen: skip wajib alasan, konfirmasi "lanjut dengan gejala",
  layar Akut tanpa jalan keluar.
- Layar PIN: lockout UI, hitung mundur.

**Acceptance:**
- [ ] `npm test` hijau; coverage untuk `lib/safety`, `runtime/offline`,
      `features/session` minimal 80% cabang.

**Depends:** F01, F02, F07, F08, F11

---

## T07 — End-to-end (Playwright)

**Spec:** §9 (user flow), §15 seluruhnya

**Scope:** `frontend/e2e/*.spec.ts`

**Skenario wajib:**
1. **Happy path:** onboarding + disclaimer + set PIN → unlock → dashboard
   → pre-check (semua "tidak") → jalankan sesi penuh (set, lewati satu
   latihan dengan alasan, rest timer) → post-check + RPE → riwayat muncul.
2. **Tier Ringan:** nyeri baru = ya → tawaran lewati/batalkan + konfirmasi
   lanjut → `action_taken` tercatat.
3. **Tier Berat:** instabilitas = ya → latihan terkait berhenti, tidak
   ada tawaran progresi.
4. **Tier Akut:** nyeri dada = ya → layar darurat, sesi tidak bisa
   dilanjutkan.
5. **Resilience:** reload di tengah sesi; **mode offline** → log lokal →
   daring → terkirim tanpa duplikat (pake route interception).
6. **PIN:** buka ulang app → PIN lagi; 5 gagal → lockout.
7. **Settings:** opt-in WA + QR tampil; ekspor JSON; hapus data → onboarding
   muncul lagi.

**Acceptance:**
- [ ] `npx playwright test` hijau di Chromium (mobile viewport) + desktop.
- [ ] Tidak ada test yang butuh gateway WhatsApp sungguhan.

**Depends:** F03–F11, B04–B11

---

## T08 — Accessibility & performance

**Spec:** §14 Usability, Performance, Accessibility; §15 butir 22

**Scope:** `frontend/e2e/a11y.spec.ts`, `frontend/e2e/perf.spec.ts`,
laporan singkat di `docs/plan/reports/`

**Kerjakan:**
- axe-core pada: Dashboard, Active Session, Pre/Post Check, Settings.
  Tanpa pelanggaran serious/critical; semua gambar/SVG punya `alt_text`.
- Navigasi keyboard penuh + fokus terlihat; status tidak hanya dengan
  warna; teks nyaman di zoom 200%.
- Lighthouse: LCP ≤ 2,5 dtk, TBT ≤ 300 ms (throttling 4G) di Dashboard
  dan Active Session; ukuran aset SVG dioptimalkan.

**Acceptance:**
- [ ] Laporan tersimpan; semua pelanggaran ditindaklanjuti atau dicatat
      dengan alasan.

**Depends:** F05–F10 selesai

---

## T09 — Security & privacy check

**Spec:** §11.2, §12.1, §14 Privacy & Security, §13 (config)

**Scope:** skrip test repo-level (`scripts/check-privacy.sh` atau Go test),
`docs/plan/reports/security.md`

**Kerjakan:**
- Assert `config.json` tidak ter-track git dan `.gitignore` mencakupnya;
  tidak ada `.env` di repo.
- Scan log output & response API: tidak ada `pain_score`, `notes`,
  `body_text`, nomor telepon penuh, token gateway.
- `wa_gateway.token` hanya dibaca dari `config.json`.
- Endpoint privat tanpa token → 401 (smoke semua rute).
- Backup/retensi: job 90 hari menghapus `body_text` outbox (uji dengan
  fixture tanggal).

**Acceptance:**
- [ ] Skrip/jalur test bisa dijalankan ulang kapan saja dan gagal keras
      bila ada kebocoran.

**Depends:** B01, B10, B11, B13
