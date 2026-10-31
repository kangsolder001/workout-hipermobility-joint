# Plan — Backend (Go + Chi + PostgreSQL)

Baca dulu `docs/plan/README.md` (aturan main & DoD global).
Rujukan spec: `docs/spec_workout.md` v1.3.

Stack tetap: Go, **Chi**, PostgreSQL, `goose` migrations, `pgx`,
repository/service/handler, konfigurasi via **`config.json`**.

---

## B01 — Scaffold backend & infrastruktur

**Spec:** §2, §13 (Backend, Deployment)

**Scope:** `backend/go.mod`, `backend/cmd/api/`, `backend/cmd/worker/`,
`backend/internal/config/`, `config.example.json`, `docker-compose.yml`,
`.gitignore`

**Kerjakan:**
- Init module `jointfit`, layout `cmd/api`, `cmd/worker`,
  `internal/{config,handler,service,repository}`.
- Loader `config.json` (address, database DSN, `wa_gateway.base_url`,
  `wa_gateway.token`, jendela kirim, jalur `config.example.json`), gagal
  jelas kalau file absen.
- `docker-compose.yml`: layanan `postgres` (volume, healthcheck) +
  placeholder `api`, `worker`, `wa-gateway`.
- Structured logging (slog) + graceful shutdown.
- `.gitignore`: `config.json`, `bin/`, `node_modules/`, artifacts test.

**Acceptance:**
- [ ] `go run ./cmd/api` start dan menjawab `GET /health` 200.
- [ ] `go test ./...` hijau, `gofmt -l .` kosong.
- [ ] `config.json` tidak ter-track; `config.example.json` lengkap dan
      komentar/isinya menjelaskan tiap field.
- [ ] Tidak ada satu pun `.env` di repo.

**Depends:** —

---

## B02 — Migrasi skema database

**Spec:** §11 seluruhnya (termasuk §11.1 enum, §11.2 aturan & retensi)

**Scope:** `backend/migrations/*.sql`

**Kerjakan:** Migrasi bertahap (goose) untuk seluruh entitas:
`users`, `auth`, `user_preferences`, `user_conditions`, `muscle_groups`,
`exercises` (+`name_id`), `exercise_media`, `exercise_targets`,
`workout_programs`, `program_phases`, `workout_templates`,
`workout_template_exercises`, `workout_schedules` (pakai **`day_role`**,
bukan `template_id`), `workout_sessions` (+snapshot versi),
`exercise_logs` (+`exercise_content_version`, `skip_reason`),
`body_checks` (+6 gejala, `tier`, `matched_rules`, `action_taken`),
`reminders`, `outbound_messages` (+`idempotency_key` UNIQUE),
`safety_rules` (tabel aturan tier §3.1, versioned).

**Acceptance:**
- [ ] Migrasi dapat dijalankan dari kosong (`goose up`) dan mundur
      (`goose down`) tanpa error.
- [ ] Semua enum sesuai §11.1 (CHECK constraint atau lookup table).
- [ ] Foreign key, index (`user_id`, `session_id`, `recorded_at`,
      `scheduled_for`, `idempotency_key`) dan `created_at/updated_at`
      ada.
- [ ] `workout_schedules` tidak punya `template_id`.

**Depends:** B01

---

## B03 — Seed katalog, program, dan aturan keselamatan

**Spec:** §5, §6 (6.1–6.5), §7, §3.1 (tabel tier), §11 `safety_rules`

**Scope:** `backend/seed/`

**Kerjakan:**
- `muscle_groups` sesuai §7.
- 19 latihan (4 pemanasan + 6 Workout A + 7 Workout B + 2 ringan) dengan
  `name` (Inggris) + `name_id`, instruksi langkah, kesalahan umum,
  `safety_cues`, `regression_exercise_id`, `progression_exercise_id`
  (nullable), dosis awal sesuai tabel §6.2/§6.3, `content_version = 1`,
  `clinical_review_status = 'pending_review'`.
- `exercise_media`: 1 SVG per latihan (placeholder path
  `frontend/public/exercises/*.svg` dulu) + `alt_text` wajib +
  `pending_review`.
- Program 12 minggu / 6 fase + `WorkoutTemplate` A, B, dan light per
  fase + `WorkoutTemplateExercise` (set/reps/hold/rest/side_mode).
- Seed `safety_rules`: 4 baris tier §3.1 + teks darurat tier Akut,
  dengan `version`.

**Acceptance:**
- [ ] Seed idempoten (re-run tidak duplikat).
- [ ] Semua latihan pada tabel §6.2/§6.3 ada, dosis cocok, tiap latihan
      punya minimal 1 regresi yang valid (kecuali yang memang tidak perlu,
      dengan alasan di `notes`).
- [ ] `clinical_review_status = 'pending_review'` untuk semua konten.
- [ ] Tabel `safety_rules` berisi persis 4 aturan + aturan default tier
      tertinggi.

**Depends:** B02

---

## B04 — Autentikasi PIN

**Spec:** §2 (Autentikasi), §11 `Auth`, §12 (`/auth/*`), §14 Privacy

**Scope:** `backend/internal/auth/`, `backend/internal/handler/auth.go`,
`backend/internal/service/auth.go`, `backend/migrations` bila perlu

**Kerjakan:**
- `POST /api/v1/auth/bootstrap` (hanya bila PIN belum ada), `unlock`,
  `pin` (ganti), `lock`.
- Hash argon2id; rate-limit **5 percobaan gagal → kunci 5 menit**
  (`failed_attempt_count`, `locked_until`).
- Token sesi yang berlaku **selama aplikasi terbuka** (token di-simpan
  client; server memvalidasi expiry + logout).
- Middleware auth untuk semua endpoint privat + `Idempotency-Key`
  middleware (penyimpanan key ≥ 24 jam, return hasil pertama).

**Acceptance:**
- [ ] PIN tak pernah tersimpan plain; verify argon2id.
- [ ] Percobaan ke-6 selama lockout → 423/429 dengan `code` konsisten.
- [ ] Tanpa token valid → 401 di semua endpoint privat.
- [ ] Request berulang dengan key sama → respons identik, data tidak
      diproses dua kali.
- [ ] Log tidak memuat PIN/token (dicek di unit test).

**Depends:** B01 (idealnya B02 untuk kolom auth)

---

## B05 — Profil, kondisi, dan preferensi

**Spec:** §8.1, §8.7, §11 (`User`, `UserCondition`, `UserPreference`),
§12 (`/me`, `/conditions`, `/preferences`)

**Scope:** `backend/internal/handler/{me,conditions,preferences}.go`,
service & repository terkait

**Kerjakan:**
- `GET/PATCH /me`, `GET/PUT /conditions`, `GET/PUT /preferences`.
- Opt-in nomor WhatsApp: validasi + **normalisasi ke `62…`**,
  `wa_consent_at`, jendela kirim (default 06:00–20:00), toggle pengingat.
- Zona waktu default `Asia/Jakarta`, locale `id-ID`.

**Acceptance:**
- [ ] Nomor `0812…` dan `62812…` menghasilkan nilai tersimpan sama.
- [ ] `wa_opt_in = false` → worker tidak mengirim apa pun (dicek B10).
- [ ] Validasi Zod-level di backend: rentang skor, enum, panjang teks.
- [ ] Semua respons terikat `user_id` sesi.

**Depends:** B02, B04

---

## B06 — Program, jadwal, dan resolve `day_role`

**Spec:** §4, §5, §11 (`WorkoutSchedule`, `WorkoutTemplate`),
§12 (`/programs/*`, `/workouts/today`, `/schedule`), §13 (prinsip
arsitektur)

**Scope:** `backend/internal/service/{program,schedule}.go`,
handler terkait

**Kerjakan:**
- `GET /programs/active`, `/programs/{id}` (fase, minggu berjalan,
  progres).
- `GET/PUT /schedule` (per `day_role`: `workout_a|workout_b|light|rest`).
- **Resolve** `day_role` → `WorkoutTemplate` pada fase aktif untuk
  tanggal berjalan; `GET /workouts/today` memakai hasil resolve ini.
- `GET /workouts/{id}` detail + `safety_cues` + tier rules (payload
  program membawa tabel `safety_rules` untuk cache offline).

**Acceptance:**
- [ ] Mengubah fase program mengubah isi sesi tanpa menyentuh jadwal.
- [ ] `GET /workouts/today` selalu memakai template fase berjalan
      (diuji dengan fixture fase berbeda).
- [ ] Tidak ada endpoint yang menaikkan dosis otomatis (B14 memvalidasi).
- [ ] Payload berisi `safety_rules` versioned.

**Depends:** B02, B03

---

## B07 — Sesi latihan & log

**Spec:** §8.3, §11 (`WorkoutSession`, `ExerciseLog`), §12
(`/sessions*`), §14 Reliability

**Scope:** `backend/internal/{handler,service}/session.go`

**Kerjakan:**
- `POST /sessions` (snapshot `program_id`, `phase_number`,
  `program_version`, `template_version`, `day_role`), `GET`, `PATCH`
  (pause/resume/complete/cancel).
- `POST /sessions/{id}/logs`: set/reps/hold/rest, `skipped` +
  `skip_reason` **wajib** bila skipped, snapshot
  `exercise_content_version`, `technique_self_rating`.
- Terima `client_uuid` untuk sinkronisasi offline (duplikat aman).
- Simpan `perceived_exertion` di sesi (satu-satunya sumber RPE).

**Acceptance:**
- [ ] Sesi yang sama tidak menggandakan log dengan `client_uuid` sama.
- [ ] Log menyimpan snapshot versi (bukti: update konten tidak mengubah
      log lama).
- [ ] `PATCH status` hanya transisi legal (state machine diuji).
- [ ] Sesi `paused` tidak menghasilkan pengingat (dikonfirmasi ke B10).

**Depends:** B02, B03, B04

---

## B08 — Check-in & evaluasi tier

**Spec:** §3.1, §3.2, §8.4, §11 `BodyCheck`, §12 `/body-checks`

**Scope:** `backend/internal/safety/`, `backend/internal/{handler,service}/bodycheck.go`

**Kerjakan:**
- `POST /body-checks` (pre/post/follow_up) + `GET /body-checks`.
- `internal/safety`: fungsi murni `Evaluate(answers) → {tier,
  matched_rules}` yang **membaca tabel `safety_rules`**, bukan hard-code.
- Simpan `tier`, `matched_rules`, `action_taken` saat disimpan; tidak
  dihitung ulang saat dibaca.
- Follow-up: tier Berat/Akut → jadwalkan `check_type = follow_up`
  6–24 jam kemudian + muncul di `GET /workouts/today`/dashboard.

**Acceptance:**
- [ ] Semua kombinasi 6 jawaban (64) menghasilkan tier sesuai tabel §3.1
      (diasumsikan tercakup test T01).
- [ ] Tier tertinggi menang saat beberapa aturan terpicu.
- [ ] Tier Akut memunculkan teks darurat yang sama persis (constanst).
- [ ] `action_taken` selalu terisi, apa pun pilihan user.

**Depends:** B02, B03, B04

---

## B09 — Riwayat & progress

**Spec:** §8.5, §7.1, §12 (`/history`, `/progress/summary`)

**Scope:** `backend/internal/{handler,service}/{history,progress}.go`

**Kerjakan:**
- `GET /history` (paginated), `GET /progress/summary`: konsistensi
  mingguan, perbandingan per `exercise_id` **dihitung per
  `exercise_content_version`** (pisahkan seri bila versi beda), ringkasan
  kondisi yang dilaporkan user.
- Tidak ada metrik koreksi flat foot / persentase kekuatan.

**Acceptance:**
- [ ] Perbandingan tidak menyatukan dua versi konten berbeda dalam satu
      tren tanpa penanda.
- [ ] Response tidak mengandung klaim klinis / kata "koreksi",
      "membaik secara struktural".
- [ ] Pagination konsisten dengan konvensi §12.1.

**Depends:** B07, B08

---

## B10 — Pengingat WhatsApp & worker

**Spec:** §8.6 (11 aturan), §13 (Integrasi WhatsApp), §11
(`OutboundMessage`, `Reminder`), §12 (`/reminders`, `/notifications/*`),
§17 risiko WA

**Scope:** `backend/internal/notify/`, `backend/cmd/worker/`,
`backend/internal/{handler,service}/{reminder,notification}.go`

**Kerjakan:**
- Interface `NotificationSender` + implementasi `WAGatewaySender`
  (baca `wa_gateway.base_url/token` dari `config.json`).
- Scheduler worker: baca `reminders` → tulis `OutboundMessage`
  (`queued`, `idempotency_key`) → proses: preflight
  `GET /api/v1/session/status` → throttle ≥ 2 dtk →
  `POST /api/v1/messages/send` → update status.
- State machine `queued → sending → sent | failed | deferred`; retry
  maks 3 dengan backoff; **tidak pernah kirim ulang `sent`**; jendela
  kirim 06:00–20:00 (tahan ke jendela berikutnya).
- Pesan untuk **seluruh hari terjadwal** + follow-up; template hanya
  jenis aktivitas + hari/jam + ajakan buka app (**tanpa data medis**).
- Endpoint: `GET/PUT /reminders`,
  `GET /notifications/outbox`,
  `GET /notifications/wa/status`,
  `POST /notifications/wa/qr`,
  `POST /notifications/wa/unlink` (proxy, token tidak pernah ke klien).

**Acceptance:**
- [ ] Gateway down/503 → `deferred`, bukan `failed` permanen; in-app
      reminder tetap ada.
- [ ] Retry tidak pernah menghasilkan pesan ganda (idempotency_key).
- [ ] Pesan di luar jendela kirim ditahan, bukan dikirim.
- [ ] Body pesan bebas data medis (ada test assertion pada template).
- [ ] `wa_gateway.token` tidak muncul di log maupun response API.

**Depends:** B05, B06

---

## B11 — Ekspor & hapus data

**Spec:** §8.7, §11.2 (retensi), §12 (`/me/export`, `DELETE /me`),
§14 Privacy

**Scope:** `backend/internal/{handler,service}/datamanagement.go`

**Kerjakan:**
- `GET /me/export` → JSON seluruh data milik user (profil, kondisi,
  sesi, log, check-in, outbox metadata).
- `DELETE /me` → hapus semua (termasuk `body_check`, `outbound_message`),
  konfirmasi wajib (header/param), hasilkan baris audit tanpa data medis.
- Hapus `body_text` `outbound_message` `sent` setelah 90 hari (job sederhana
  di worker).

**Acceptance:**
- [ ] Setelah `DELETE`, query semua tabel menghasilkan 0 baris untuk
      user itu.
- [ ] Export mengembalikan data lengkap dan valid JSON.
- [ ] Job retensi berjalan dan hanya menghapus data yang lewat 90 hari.

**Depends:** B02, B04

---

## B12 — Kontrak OpenAPI & validasi request

**Spec:** §12, §12.1, §13 (OpenAPI untuk kontrak API)

**Scope:** `backend/api/openapi.yaml`, middleware validasi di handler

**Kerjakan:**
- `openapi.yaml` lengkap untuk seluruh endpoint §12 (path, schema, enum,
  error `code/message/details`, contoh).
- Validasi request mengikuti schema (Zod-equivalent di Go: kin-openapi
  atau validator manual) + normalisasi error format seragam.
- Dokumen konvensi: pagination, `Idempotency-Key` (endpoint yang wajib),
  auth.

**Acceptance:**
- [ ] Setiap endpoint di §12 ada di `openapi.yaml` dan berfungsi.
- [ ] Request tidak valid → 400 dengan `code` + `details` field bermasalah.
- [ ] Spec valid (lulus validator OpenAPI).

**Depends:** B04 (untuk skema auth), dikerjakan paralel dengan B05–B11,
finalisasi setelah B11.

---

## B13 — Logging, privasi, dan format error

**Spec:** §12.1, §14 Privacy & Security, §11.2

**Scope:** `backend/internal/logging/`, middleware di handler

**Kerjakan:**
- Redaction middleware: `pain_score`, `fatigue_score`, `notes`,
  `body_text`, nomor telepon, token tidak pernah masuk log.
- Format error konsisten (`code`, `message`, `details`) untuk seluruh
  handler; panic ter-recovery jadi 500 tanpa bocor stack ke klien.
- Authorization check terpusat (semua resource terikat `user_id`).

**Acceptance:**
- [ ] Test membuktikan log tidak mengandung field sensitif pada alur
      normal + alur error.
- [ ] Tidak ada endpoint yang bisa membaca data user lain.

**Depends:** B04, minimal satu fitur (B07/B08) untuk diuji

---

## B14 — Gerbang konten & penegakan dosis

**Spec:** §15 butir 14 & 17, §5.1, §6.5, §13 (prinsip arsitektur)

**Scope:** `backend/internal/service/guards.go` + dipanggil di
program/session

**Kerjakan:**
- Guard `clinical_review_status`: program/konten `pending_review` tidak
  boleh dilaporkan sebagai program aktif (baik respons API maupun seed
  runtime).
- Guard dosis: `POST /logs` menolak set/reps/hold di luar batas template
  + aturan §5.1 (maks set sesuai fase, isometric maks 3×8 detik).
- Guard anti koreksi flat foot: respons progress/summary tidak boleh
  memuat metrik semacam itu (allow-list field).

**Acceptance:**
- [ ] Permintaan menaikkan set di luar fase → 400 dengan kode jelas.
- [ ] Program tetap `pending_review` tidak muncul sebagai aktif.
- [ ] Tidak ada endpoint/kode yang menaikkan dosis otomatis (dicek
      dengan test + grep strategi di T02).

**Depends:** B03, B06, B07
