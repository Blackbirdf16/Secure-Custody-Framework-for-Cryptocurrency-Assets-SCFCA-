# SCFCA — Secure Custody Framework for Cryptocurrency Assets

A secure technical framework for the custody, management, preservation, and auditing of cryptocurrency assets in criminal investigations.

Master's thesis proof of concept — 29/29 FR, 20/20 SR, 9/9 MU, 41 tests.

---

## 1. Run It

### Option A: Docker Compose (recommended)

```bash
docker compose up --build
```

| Service | URL |
|---------|-----|
| Frontend | http://localhost:5173 |
| Backend API | http://localhost:8000 |
| Swagger Docs | http://localhost:8000/docs |

Seed demo data (first time):
```bash
docker compose exec backend python scripts/seed_demo_data.py
```
or if it doesnt work as supposed to

```bash
docker compose exec backend sh -c "mkdir -p /app/scripts"
docker cp .\scripts\seed_demo_data.py scfca-backend:/app/scripts/seed_demo_data.py
docker compose exec backend python /app/scripts/seed_demo_data.py
```

### Option B: Local (no Docker)

**Terminal 1 — Backend:**
```bash
pip3 install -r backend/requirements.txt
python3 -m uvicorn backend.main:app --reload --host 127.0.0.1 --port 8000
```

**Terminal 2 — Frontend:**
```bash
npm --prefix frontend install
npm --prefix frontend run dev -- --host 127.0.0.1 --port 5173
```

> Without PostgreSQL, the backend runs in **demo mode** with in-memory data and 4 pre-seeded users. All features work except DB persistence.

---

## 2. Login

| Role | Username | Password | Permissions |
|------|----------|----------|-------------|
| Handler | `alice` | `alice123` | View assigned cases, create tickets, upload PDFs |
| Admin | `bob` | `bob123` | Create cases/assets, approve tickets, execute actions |
| Admin | `eve` | `eve123` | Second approver for dual-control |
| Auditor | `carol` | `carol123` | Read-only: audit trail, verify chain, download reports |

---

## 3. Test It

### Automated Tests (41 tests, no PostgreSQL needed)

```bash
pip3 install -r backend/requirements.txt pytest
python3 -m pytest tests/ -v
```

Expected output:
```
test_cases_and_assets.py       8 passed
test_documents_and_reports.py  7 passed
test_security_phase5.py        7 passed
test_tickets_phase3.py         8 passed
test_user_management.py        5 passed
test_workflows.py              6 passed
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
41 passed in ~70s
```

### Run by category

```bash
python3 -m pytest tests/test_workflows.py -v          # Core: CSRF, dual approval, audit chain
python3 -m pytest tests/test_user_management.py -v     # Users: CRUD, RBAC
python3 -m pytest tests/test_cases_and_assets.py -v    # Cases + assets + immutability
python3 -m pytest tests/test_tickets_phase3.py -v      # Tickets: cancel, notes, side-effects
python3 -m pytest tests/test_documents_and_reports.py -v  # Documents + PDF reports
python3 -m pytest tests/test_security_phase5.py -v     # MFA, re-auth, rate limiting
```

### Run a single test

```bash
python3 -m pytest tests/ -k "mfa" -v                  # All MFA tests
python3 -m pytest tests/ -k "immutability" -v          # Asset immutability
python3 -m pytest tests/test_workflows.py::test_audit_hash_chain_verifies -v
```

---

## 4. Manual Test Walkthrough

Start the backend (`python3 -m uvicorn backend.main:app --reload`), then follow this sequence.

### Step 1 — Login as admin

```bash
curl -X POST http://localhost:8000/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"bob","password":"bob123"}' \
  -c cookies.txt

# Extract CSRF token
CSRF=$(grep scfca_csrf_token cookies.txt | awk '{print $NF}')
```

### Step 2 — Create a case

```bash
curl -X POST http://localhost:8000/api/v1/cases/ \
  -H "Content-Type: application/json" \
  -H "X-CSRF-Token: $CSRF" \
  -d '{"title":"Seized BTC wallet","wallet_ref":"WLT-001","handler_username":"alice"}' \
  -b cookies.txt
# Note the case ID from the response (e.g., C-7A3F)
```

### Step 3 — Register an asset

```bash
curl -X POST http://localhost:8000/api/v1/assets/ \
  -H "Content-Type: application/json" \
  -H "X-CSRF-Token: $CSRF" \
  -d '{"caseId":"C-7A3F","symbol":"BTC","assetType":"native","quantity":12.5}' \
  -b cookies.txt
```

### Step 4 — Login as alice, create a ticket

```bash
curl -X POST http://localhost:8000/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"alice","password":"alice123"}' \
  -c alice.txt

ALICE_CSRF=$(grep scfca_csrf_token alice.txt | awk '{print $NF}')

curl -X POST http://localhost:8000/api/v1/tickets/ \
  -H "Content-Type: application/json" \
  -H "X-CSRF-Token: $ALICE_CSRF" \
  -d '{"caseId":"C-7A3F","ticketType":"transfer_request","description":"Transfer to cold storage"}' \
  -b alice.txt
# Note the ticket ID (e.g., T-482)
```

### Step 5 — Dual approval (bob, then eve)

```bash
# Bob approves (stage 1)
curl -X POST http://localhost:8000/api/v1/tickets/T-482/approve \
  -H "Content-Type: application/json" \
  -H "X-CSRF-Token: $CSRF" \
  -d '{"notes":"Documentation verified."}' \
  -b cookies.txt

# Eve approves (stage 2)
curl -X POST http://localhost:8000/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"eve","password":"eve123"}' \
  -c eve.txt

EVE_CSRF=$(grep scfca_csrf_token eve.txt | awk '{print $NF}')

curl -X POST http://localhost:8000/api/v1/tickets/T-482/approve \
  -H "Content-Type: application/json" \
  -H "X-CSRF-Token: $EVE_CSRF" \
  -d '{"notes":"Confirmed. Approved for execution."}' \
  -b eve.txt
```

### Step 6 — Assign and execute (with re-auth)

```bash
# Assign to bob
curl -X PATCH http://localhost:8000/api/v1/tickets/T-482/assign \
  -H "Content-Type: application/json" \
  -H "X-CSRF-Token: $CSRF" \
  -d '{"assignedTo":"bob"}' \
  -b cookies.txt

# Get re-auth token (SR-6)
REAUTH=$(curl -s -X POST http://localhost:8000/api/v1/auth/reauth \
  -H "Content-Type: application/json" \
  -H "X-CSRF-Token: $CSRF" \
  -d '{"password":"bob123"}' \
  -b cookies.txt | python3 -c "import sys,json; print(json.load(sys.stdin)['reauth_token'])")

# Execute
curl -X POST http://localhost:8000/api/v1/tickets/T-482/execute \
  -H "X-CSRF-Token: $CSRF" \
  -H "Idempotency-Key: exec-001" \
  -H "X-Reauth-Token: $REAUTH" \
  -b cookies.txt
```

### Step 7 — Verify audit chain

```bash
curl http://localhost:8000/api/v1/audit/verify -b cookies.txt
# {"ok": true, "count": N}
```

### Step 8 — Download reports

```bash
curl http://localhost:8000/api/v1/reports/audit -b cookies.txt -o audit_report.pdf
curl http://localhost:8000/api/v1/reports/case/C-7A3F -b cookies.txt -o case_report.pdf
```

---

## 5. Test Security Controls

### CSRF protection
```bash
# POST without CSRF token → 403
curl -X POST http://localhost:8000/api/v1/tickets/ \
  -H "Content-Type: application/json" \
  -d '{"caseId":"C-100","ticketType":"transfer_request","description":"test"}' \
  -b cookies.txt
# Expected: 403 CSRF validation failed
```

### Rate limiting
```bash
# 9 rapid failed logins → last ones get 429
for i in $(seq 1 9); do
  curl -s -o /dev/null -w "%{http_code} " \
    -X POST http://localhost:8000/api/v1/auth/login \
    -H "Content-Type: application/json" \
    -d '{"username":"alice","password":"wrong"}'
done
echo
# Expected: 401 401 401 401 401 401 401 401 429
```

### MFA enrollment
```bash
# As admin bob (already logged in)
curl -X POST http://localhost:8000/api/v1/auth/mfa/setup \
  -H "X-CSRF-Token: $CSRF" \
  -b cookies.txt
# Returns: {"secret":"BASE32...","provisioning_uri":"otpauth://..."}
# Scan QR with authenticator app, then verify:
curl -X POST http://localhost:8000/api/v1/auth/mfa/verify \
  -H "Content-Type: application/json" \
  -H "X-CSRF-Token: $CSRF" \
  -d '{"code":"123456"}' \
  -b cookies.txt
```

### Re-auth required for execution
```bash
# Execute WITHOUT re-auth token → 403
curl -X POST http://localhost:8000/api/v1/tickets/T-482/execute \
  -H "X-CSRF-Token: $CSRF" \
  -H "Idempotency-Key: no-reauth" \
  -b cookies.txt
# Expected: 403 Re-authentication required
```

### Asset immutability
```bash
# This can only be tested programmatically (ORM listener):
python3 -m pytest tests/test_cases_and_assets.py::test_asset_immutability_enforced -v
# Confirms: modifying quantity raises ValueError
```

---

## 6. Demo Script (Thesis Defense)

Suggested live demonstration order:

1. `docker compose up --build` — start system
2. Login as **alice** → show handler dashboard, restricted view
3. Login as **bob** → show admin dashboard, create case, register assets
4. Upload PDF evidence → show SHA-256 hash
5. Login as **alice** → create ticket for assigned case
6. **bob** approves (stage 1) → show status `awaiting_second_approval`
7. **eve** approves (stage 2) → show status `approved`
8. **bob** executes → show re-auth requirement (SR-6)
9. Verify audit chain → `GET /audit/verify` → `{"ok": true}`
10. Download reports → audit PDF + case PDF
11. Setup MFA for bob → show TOTP enrollment (SR-5)
12. Attempt asset modification → show immutability (SR-15)
13. Rapid requests → show rate limiting (MU-9)
14. POST without CSRF → show 403 (SR-18)

---

## 7. Project Structure

```
project-scfca-main/
├── backend/
│   ├── api/v1/routes/     10 route files (~40 endpoints)
│   ├── models/            10 ORM models + listeners
│   ├── repositories/       6 data access classes
│   ├── services/           7 business logic modules
│   ├── middleware/          CSRF + rate limiting
│   ├── auth/              JWT, RBAC, MFA, re-auth
│   ├── alembic/           5 versioned migrations
│   └── requirements.txt
├── frontend/
│   ├── src/pages/          9 React pages
│   ├── src/services/       8 API service modules
│   └── src/components/     7 reusable components
├── tests/                  6 test files (41 tests)
├── docs/                   10 documentation files
├── docker-compose.yml
├── .gitlab-ci.yml
└── README.md
```

---

## 8. Documentation

See [docs/README.md](docs/README.md) for the full index, or jump directly:

- [Implementation Record](docs/implementation-record.md) — what was built and why
- [Traceability Matrix](docs/traceability-matrix.md) — every requirement → code → test
- [Testing Guide](docs/testing-guide.md) — detailed testing instructions

---

## 9. Troubleshooting

| Problem | Fix |
|---------|-----|
| `docker-compose: command not found` | Use `docker compose` (with space) or install Docker Desktop |
| `ModuleNotFoundError: No module named 'backend'` | Run from project root: `python3 -m pytest tests/` |
| `TypeError: unsupported operand type(s) for \|` | `pip3 install eval_type_backport` |
| `429 Too Many Requests` | Wait 60 seconds (rate limiter resets) |
| Alice gets "Case not assigned" | Use case ID `C-100` (pre-assigned) or create a case as bob first |
| No CSRF cookie after login | If MFA is enabled, complete the `/mfa/challenge` step first |
