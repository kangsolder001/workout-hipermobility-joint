# TODO — JointFit MVP

Checklist gabungan. Detail task ada di `docs/plan/`:
`README.md` (aturan main), `backend.md`, `frontend.md`, `testing.md`.

**Status:** `[ ]` belum · `[~]` berjalan · `[x]` selesai
**Wave:** urutan eksekusi aman untuk sub agent (lihat README §4).

Legenda dependensi: `B02` = butuh B02 selesai.

---

## Wave 1 — bisa paralel sekarang

- [ ] **B01** Backend scaffold, config.json loader, docker-compose —
      deps: — · `docs/plan/backend.md`
- [ ] **F01** Frontend scaffold, design system, PWA shell — deps: — ·
      `docs/plan/frontend.md`
- [ ] **T01** Fixture tabel tier 64 kombinasi + unit test §3.1 —
      deps: konsep (mulai dari spec) · `docs/plan/testing.md`

## Wave 2 — fondasi data

- [ ] **B02** Migrasi DB seluruh §11 + enum + index — deps: B01
- [ ] **B03** Seed katalog (19 latihan, SVG, program 12 mgg, safety_rules)
      — deps: B02
- [ ] **B04** Auth PIN (argon2id, lockout, token sesi, Idempotency-Key)
      — deps: B01, B02

> Catatan: **B12** dimulai sejak Wave 2 (draft: auth + profil) agar F02
> bisa jalan, dan diselesaikan di Wave 3.

## Wave 3 — fitur backend (paralel)

- [ ] **B05** Profil, kondisi, preferensi + opt-in WA — deps: B02, B04
- [ ] **B06** Program, schedule, resolve `day_role`, safety payload —
      deps: B02, B03
- [ ] **B07** Sesi & log + snapshot versi + `client_uuid` — deps: B02,
      B03, B04
- [ ] **B08** Body-check + service tier + follow-up — deps: B02, B03, B04
- [ ] **B09** Riwayat & progress — deps: B07, B08
- [ ] **B10** Worker WhatsApp + outbox + proxy QR/status — deps: B05, B06
- [ ] **B11** Ekspor & hapus data + job retensi 90 hari — deps: B02, B04
- [ ] **B12** OpenAPI lengkap + validasi request — deps: B05–B11
      (draft dari Wave 2)
- [ ] **B13** Logging redaksi + format error + authz terpusat — deps: B04
- [ ] **F02** API client + Zod + error handling — deps: B12 (draft)

## Wave 4 — fitur frontend (paralel)

- [ ] **F11** Runtime offline: SW cache + antrean IndexedDB + sync —
      deps: F01, (kontrak B07/B08)
- [ ] **F03** Onboarding + layar PIN — deps: F01, F02, B04
- [ ] **F04** Profil & kondisi — deps: F02, B05
- [ ] **F05** Dashboard, jadwal, detail program — deps: F02, B06
- [ ] **F06** Detail latihan (cue, regresi, SVG, alt_text) — deps: F02, B06
- [ ] **F07** Sesi aktif (set, timer, skip, resume) — deps: F02, F11, B07
- [ ] **F08** Alur check-in pre/post + layar tier/Akut — deps: F02, B08
- [ ] **F09** Riwayat & progress — deps: F02, B09
- [ ] **F10** Settings (PIN, WA opt-in, QR, ekspor/hapus) — deps: F02, B05,
      B10, B11

## Wave 5 — guard & testing (paralel setelah fitur)

- [ ] **B14** Guard konten `pending_review`, penegakan dosis, allow-list
      field progress — deps: B03, B06, B07
- [ ] **T02** Guard dosis & anti auto-progression — deps: B14, B06, B07
- [ ] **T03** Auth, idempotency, redaksi log — deps: B04, B13
- [ ] **T04** Integration API + DB — deps: B02–B11
- [ ] **T05** Worker pengingat WA (mock gateway) — deps: B10
- [ ] **T06** Unit test frontend (Vitest) — deps: F06–F11
- [ ] **T07** E2E Playwright (7 skenario) — deps: F03–F11, B04–B11
- [ ] **T08** Accessibility + performance (axe, Lighthouse) — deps: F05–F10
- [ ] **T09** Security & privacy check — deps: B01, B10, B11, B13

## Diluar cakupan sub agent (prasyarat rilis, §18)

- [ ] **R01** Review konten latihan oleh fisioterapis → flip semua seed
      `pending_review` → `reviewed` (Milestone 6, gate §15 butir 14).
- [ ] **R02** Isi final template bahasa pesan WhatsApp.
- [ ] **R03** Host/URL gateway WhatsApp final di `config.json`.
- [ ] **R04** HTTPS + backup Postgres (menjadi kebutuhan saat diakses dari
      luar localhost).

---

## Status singkat

| Wave | Task | Selesai |
|---|---|---|
| 1 | B01, F01, T01 | 0/3 |
| 2 | B02, B03, B04 (B12 draft menyusul) | 0/3 |
| 3 | B05–B13, F02 | 0/10 |
| 4 | F03–F11 | 0/9 |
| 5 | B14, T02–T09 | 0/9 |
| Rilis | R01–R04 | 0/4 |
| **Total** | | **0/38** |
