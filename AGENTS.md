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

## Shift Module (phân ca)
- `shift/` package: handler.go, service.go, repository.go, routes.go, model.go
- Tables: `shifts` (UUID PK, name, type, start_time, end_time, date, status) + `shift_assignments` (shift_id + user_id composite PK)
- Routes require `staffOnly` (staff/faculty/admin)
- Frontend: `admin/shifts/page.tsx` (admin calendar), `staff/my-shift/page.tsx` (staff view)
- **Known fix**: Never scan PostgreSQL `DATE` directly into Go `string` — pgx returns full timestamp (`2026-06-25T00:00:00Z`). Scan into `time.Time` and format with `Format("2006-01-02")` instead.

## Common Gotchas
- Go struct `encoding/json` requires explicit `json:"..."` tags for structs decoded from request bodies (case-insensitive fallback is unreliable). All `handler.go` request structs should have tags.
- Frontend API client uses `NEXT_PUBLIC_API_URL` env var, defaults to `http://localhost:8080/api/v1`
- Image files stored locally in `./images/` directory
- OCR service URL in `.env` must include the full path: `http://ocr-service:5000/api/v1/detect-image`
- Default dev secrets in `.env`: `JWT_SECRET=changeme`, two admin accounts in seed data:
  - `admin@bikekeeper.local` / `admin123` (migration 00005, hash fixed in 00018)
  - `admin@parksmart.vn` / `demo123` (migration 00019)
- Seed users (migration 19): students `SV20210001`/`SV20210042`, guard (`guard@parksmart.vn`), admin; password `demo123`
