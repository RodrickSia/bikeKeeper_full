# SE104_bikeKeeper — Agent Guide

## Stack
- **Backend**: Go 1.26, `net/http` (no framework), `pgx/v5` + `database/sql`, Goose migrations
- **Frontend**: Next.js 16.2 (App Router, `output: "standalone"`), React 19, Tailwind CSS 4, Zustand, TanStack Query 5
- **OCR**: Python FastAPI, YOLO + Vintern/EasyOCR
- **Infra**: Docker Compose (6 services), nginx reverse proxy, optional cloudflared tunnel

## Key Commands
```sh
cd backend/bikeKeeper
go build -o bikekeeper ./cmd/bikeKeeper
go test -v -count=1 ./...
go vet ./...

cd FE/frontend
npm run dev          # dev server (port 3000)
npm run build        # production build
npm run lint         # ESLint

cd backend
docker compose up    # backend + postgres + ocr

# from repo root
docker compose up    # full stack (all services)
```

## Architecture
- All API routes under `/api/v1`. nginx routes `/api/` → backend:8080, `/` → frontend:3000
- 19 Goose SQL migrations in `backend/bikeKeeper/internal/database/migrations/`
- JWT auth via `auth.Authenticate()` middleware, role check via `auth.RequireRole(roles...)`
- Roles: `student` < `staff` < `faculty` < `admin`
- DB: PostgreSQL 17, driver via `github.com/jackc/pgx/v5/stdlib`, connection via `DATABASE_URL`

## Tasks Completed

### Shift DATE Scan Fix
- Root cause: `shift/repository.go` scanned PostgreSQL `DATE` into Go `string`, pgx returned full ISO timestamp (`2026-06-25T00:00:00Z`)
- Fix: Changed to `time.Time` scan + `Format("2006-01-02")`
- Files: `backend/bikeKeeper/internal/shift/repository.go:32–45,63–82`

### OCR URL Fix
- `.env` entry `OCR_SERVICE_URL` was missing the full path `/api/v1/detect-image`
- Fix: `http://ocr-service:5000/api/v1/detect-image`
- File: `backend/bikeKeeper/.env`

### Shift JSON Tags
- `CreateParams` struct in `shift/service.go` had no `json:"..."` tags
- Fix: added explicit tags (`name`, `type`, `start_time`, `end_time`, `date`)
- File: `backend/bikeKeeper/internal/shift/service.go`

### Wallet 403 Auth Bug
- Root cause: Card routes (`GET /members-cards/{memberID}`, etc.) were behind `facultyOnly` middleware → students got 403 → wallet showed balance 0
- Fix: Moved card routes from `facultyOnly` to `authenticated` middleware; added handler-level checks:
  - `listByMember`: students view own cards, faculty/admin view any
  - `create/update/delete/toggleInside`: require faculty/admin inline
- Files: `internal/card/handler.go`, `internal/app/app.go`

### Deposit Ownership Check
- Added `CardFinder` interface in `payment/handler.go` to look up card → member ownership
- Students can only deposit to own cards (`403` on cross-account deposit)
- Faculty/admin can deposit to any card
- Files: `internal/payment/handler.go`, `internal/payment/handler.go` (CardFinder), `internal/app/app.go` (adapter wiring)

### Monthly Pass Backend + Frontend Fix
- Backend: Added `Withdraw` method (repo + service + handler) — deducts balance and inserts `monthly_pass` transaction
- Frontend: `registerMonthlyPass` previously checked a mock `_wallets` array instead of real DB card balance
- Fix: Now fetches real card balances via `cardService.getCardsByMember()`, checks total, calls `POST /cards/{cardUID}/withdraw`
- Files (backend): `internal/payment/model.go`, `repository.go`, `service.go`, `handler.go`, `routes.go`
- Files (frontend): `src/services/wallet.service.ts`, `src/lib/endpoints.ts`

### Massive Data Seed
- Created `backend/bikeKeeper/scripts/seed_massive_data.sql`
- Seeds: 30 members, 35 users, 24 cards, 60 registered vehicles, 33 vehicle types, 38 parking sessions, 52 shifts, 59 shift assignments, 15 devices, 36 notifications, 15 support tickets, 10 responses, 13 incidents, 20 visitor passes, 16 card requests, 36 transactions
- Run with: `Get-Content backend/bikeKeeper/scripts/seed_massive_data.sql | docker compose exec -T postgres psql -U bikekeeper`

## Gotchas
- pgx/v5/stdlib scanning `DATE` into Go `string` produces ISO timestamp; must scan into `time.Time` + format
- Two admin accounts: `admin@parksmart.vn` / `demo123` (works in seed), `admin@bikekeeper.local` / `admin123` (may have bcrypt mismatch)
- Card routes no longer use `facultyOnly` middleware; read ops check JWT `memberId` claim vs URL `memberID`; write ops check `role` claim
- Payment deposit & withdraw endpoints validate card ownership via `payment.CardFinder` → `card.Repository` adapter
- Monthly pass frontend still uses in-memory storage (`_monthlyPasses` array); no backend table yet

## Relevant Files
- `backend/bikeKeeper/internal/shift/repository.go` — DATE scan fix
- `backend/bikeKeeper/internal/shift/service.go` — JSON tags
- `backend/bikeKeeper/.env` — OCR URL
- `backend/bikeKeeper/internal/card/handler.go` — role-based card access
- `backend/bikeKeeper/internal/payment/handler.go` — deposit/withdraw ownership checks
- `backend/bikeKeeper/internal/payment/repository.go` — Deposit/Withdraw/ChargeParking
- `backend/bikeKeeper/internal/payment/service.go` — Service layer
- `backend/bikeKeeper/internal/app/app.go` — Route wiring
- `backend/bikeKeeper/scripts/seed_massive_data.sql` — Full test data seed
- `FE/frontend/src/services/wallet.service.ts` — Fixed registerMonthlyPass
- `FE/frontend/src/lib/endpoints.ts` — WITHDRAW endpoint
- `AGENTS.md` — This file
