# JointFit — Plan Pengembangan MVP

Dokumen kerja untuk menjalankan implementasi lewat sub agent.
Sumber kebenaran tetap **`docs/spec_workout.md` (v1.2)** — plan ini hanya
memecahnya jadi unit kerja yang bisa dikerjakan terpisah.

## 1. Struktur dokumen plan

| File | Isi | Untuk siapa |
|---|---|---|
| `docs/plan/README.md` | Aturan main, konvensi, dependensi (file ini) | Semua |
| `docs/plan/backend.md` | Task backend `B01`–`B14` (Go + Chi + PostgreSQL) | Sub agent backend |
| `docs/plan/frontend.md` | Task frontend `F01`–`F11` (React + TS + Vite + PWA) | Sub agent frontend |
| `docs/plan/testing.md` | Task testing `T01`–`T09` (unit, integration, E2E, a11y, security) | Sub agent QA/testing |
| `docs/TODO.md` | Checklist gabungan semua task + dependensi + status | Koordinator / manusia |

## 2. Layout repo yang dituju

```
workout-hipermobility-joint/
├── docs/
│   ├── spec_workout.md          # sumber kebenaran (v1.2)
│   ├── reference/openapi.json   # kontrak gateway WhatsApp
│   ├── plan/{README,backend,frontend,testing}.md
│   └── TODO.md
├── backend/
│   ├── cmd/api/                 # HTTP server
│   ├── cmd/worker/              # scheduler pengingat WhatsApp
│   ├── internal/config/         # loader config.json (+ config.example.json)
│   ├── internal/auth/           # argon2id, rate-limit, session token
│   ├── internal/safety/         # evaluasi tier check-in (data-driven)
│   ├── internal/notify/         # NotificationSender + WAGatewaySender
│   ├── internal/{handler,service,repository}/
│   ├── migrations/              # goose SQL migrations
│   ├── seed/                    # seed katalog, program, safety rules
│   ├── api/openapi.yaml         # kontrak API JointFit
│   └── *_test.go
├── frontend/
│   ├── src/api/                 # client + Zod schema
│   ├── src/lib/safety/          # konsumsi tabel tier dari backend
│   ├── src/runtime/offline/     # SW + IndexedDB queue + sync
│   ├── src/features/, src/pages/
│   └── tests: src/**/*.test.ts, e2e/*.spec.ts (Playwright)
├── config.example.json          # template konfigurasi (config.json di-gitignore)
└── docker-compose.yml           # postgres + api + worker + wa-gateway
```

## 3. Aturan main (WAJIB dibaca sub agent)

1. **Spesifikasi adalah kontrak.** Jangan mengubah isi
   `docs/spec_workout.md` dalam mengerjakan task. Jika ada bagian yang
   ternyata tidak bisa dipenuhi atau bertentangan, **berhenti dan laporkan**
   kepada koordinator — jangan menyimpang diam-diam.
2. **Tanpa `.env`.** Semua konfigurasi lewat `config.json`
   (`config.example.json` adalah template). `config.json` masuk
   `.gitignore` dan tidak pernah dikirim ke klien atau log.
3. **Konten seed berstatus `pending_review`.** Jangan menandai `reviewed`
   — itu prasyarat rilis Milestone 6, di luar cakupan task dev.
4. **Batas keselamatan tidak boleh dilanggar**: tanpa auto-progression,
   tanpa skor otomatis pada check-in, tanpa klaim medis/koreksi flat foot,
   pesan WhatsApp tanpa data medis.
5. **Bahasa**: UI, pesan, dan catatan user-facing berbahasa Indonesia;
    nama latihan tetap Inggris dengan alias `name_id`.
6. **Commit**: konvensional (`feat(backend): …`, `fix(frontend): …`,
    `test: …`, `docs: …`), satu task minimal satu commit, Jangan commit
   file yang bukan bagian scope task.
7. **Definisi selesai (DoD) global** — sebuah task dianggap selesai hanya
   jika:
   - seluruh *Acceptance* pada task itu terpenuhi;
   - code ter-format (`gofmt` / `prettier`), linter hijau;
   - unit test yang relevan hijau (untuk task testing: seluruh test yang
     ditugaskan ada dan hijau);
   - `docs/TODO.md` baris task tersebut dicentang `[x]`;
   - ada commit dengan prefix ID task (mis. `feat(backend): B04 auth`).

## 4. Peta dependensi

```
B01 (scaffold) ─┬─> B02 (migrasi) ─┬─> B03 (seed) ─┬─> B06 (schedule/resolve)
                │                  │               ├─> B07 (sessions)
                ├─> B04 (auth)     ├─> B05 (profil)├─> B08 (body-check + tier)
                │                  │               ├─> B09 (history/progress)
                │                  │               └─> B10 (reminder + worker)
                └─> B12 (OpenAPI) ─┴─> F02 (API client) ─> F03..F10
                                                └─> F11 (offline) ─> T06/T07
B11 (export/delete) ────────────────────────────────┐
T01..T03 bisa mulai segera setelah B02/B04/B08 ─────┤
                                                     └─> T04, T05, T08, T09
```

Urutan aman untuk sub agent:

- **Wave 1 (paralel):** `B01` (scaffold backend), `F01` (scaffold
  frontend), `T01` (fixture tabel tier — boleh dimulai dari spec).
- **Wave 2 (paralel):** `B02` (migrasi) → `B03` (seed), `B04` (auth);
  `B12` draft (auth + profil) agar `F02` bisa mulai.
- **Wave 3 (paralel):** `B05`–`B13` (fitur backend) + `F02` (API client).
- **Wave 4 (paralel):** `F11` (offline runtime), `F03`–`F10` (fitur
  frontend).
- **Wave 5 (paralel):** `B14` (guard) lalu `T02`–`T09` (testing); perbaiki
  temuan.
- **Rilis (di luar plan ini):** `R01`–`R04` — review konten fisioterapis →
  flip `reviewed` → gate §15 terbuka → rilis.

## 5. Cara menjalankan sub agent (template prompt)

```
Kamu mengerjakan task <ID> dari rencana JointFit.

1. Baca: docs/spec_workout.md bagian <daftar§>, docs/plan/<track>.md
   bagian <ID>, dan docs/plan/README.md (aturan main, wajib dipatuhi).
2. Repo: /home/kangsolder/Documents/m2/code/workout-hipermobility-joint
3. Kerjakan hanya scope file yang tercantum pada task. Jangan menyentuh
   file di luar scope dan jangan mengubah spec.
4. Penuhi seluruh Acceptance; jalankan test/format yang relevan.
5. Centang task di docs/TODO.md, commit dengan prefix ID task.
6. Laporkan: yang selesai, yang tertunda + alasan, dan pertanyaan bila
   ada bagian spec yang tidak jelas.
```

## 6. Perintah dasar

```bash
# backend
cd backend && go test ./... && gofmt -l .

# frontend
cd frontend && npm run lint && npm test

# stack lokal
docker compose up -d postgres
cp config.example.json config.json   # lalu isi nilai gateway

# E2E
cd frontend && npx playwright test
```
