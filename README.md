# Pramāṇa: Prescription Provenance Platform

Pramāṇa makes every prescription verifiable. Doctors sign prescriptions cryptographically in their own browser, and pharmacists, patients and hospitals can later check whether a prescription is genuine, untampered and issued by a verified doctor.

The repo is a monorepo with one React frontend (four portals behind one router) and a FastAPI backend.

## Team Krito

| No. | Member |
| --- | --- |
| 1 | Rishi Kumar |
| 2 | Narayan Panigrahi |
| 3 | Shaswat Singh |

## Portals

| Portal | Path | Who uses it | What it does |
| --- | --- | --- | --- |
| Doctor | `/doctor` | Doctors | Identity and licence onboarding, clinic verification, signing-key setup, writing and signing prescriptions, history, verification status |
| Verification | `/verification` | Pharmacists and patients | Check a prescription by QR or reference, photo verification, trust-tier explainer, pharmacy registration and sign-in |
| Admin | `/admin` | Platform reviewers | Flagged queue, doctor and pharmacy review, patient flags, action ledger, audit log, admin signing key |
| Organization | `/organization` | Hospitals | Register an organization, manage the verified doctor roster, activity analytics |

Checking a prescription never requires an account. Sign-in is only needed to mark a prescription as dispensed.

## How it works

1. **Onboarding.** A doctor verifies identity (Aadhaar OTP), medical licence and clinic address.
2. **Signing key.** The doctor creates an Ed25519 key pair in the browser. The private key is encrypted with a passphrase and stays on the device. Only the public key is registered with the server.
3. **Sign.** When writing a prescription, the doctor unlocks the key and signs the payload in the browser. The server verifies the signature and seals the record.
4. **Verify.** A pharmacist scans the QR code or enters the reference. The platform returns a trust tier and the signed record.
5. **Photo check.** For a photo of a paper prescription, the platform runs OCR, forensic analysis, an AI tampering signal and text and layout consistency checks.

Session tokens (HS256 JWT) are separate from prescription and admin signing keys (Ed25519).

## Tech stack

**Frontend** (`apps/frontend`): React, TypeScript, Vite, React Router, Redux Toolkit (doctor portal), Tailwind, pnpm workspaces.

Shared packages under `packages/`:
- `@pramana/ui-components`: design system and component kit
- `@pramana/api-client`: HTTP client and route constants
- `@pramana/crypto`: browser key generation, unlock and signing
- `@pramana/types`: shared TypeScript types

**Backend** (`apps/backend`): FastAPI, async SQLAlchemy, PostgreSQL (asyncpg), Alembic migrations, argon2 password hashing, PyJWT.

## Repository layout

```
prescription-provenance/
├─ apps/
│  ├─ frontend/
│  │  ├─ src/
│  │  │  ├─ router.tsx            one router for all portals
│  │  │  ├─ shared/               routes table, common components
│  │  │  └─ portals/
│  │  │     ├─ doctor/
│  │  │     ├─ admin/
│  │  │     ├─ verification/
│  │  │     └─ organization/
│  │  └─ vite.config.ts
│  └─ backend/
│     ├─ src/
│     │  ├─ app_factory.py
│     │  ├─ main.py
│     │  ├─ core/                 db, security (jwt, rbac, crypto)
│     │  └─ modules/              doctors, organizations, pharmacies, patients, ...
│     ├─ migrations/versions/     0001 to 0004
│     └─ scripts/seed_dev_data.py
└─ packages/
   ├─ ui-components/
   ├─ api-client/
   ├─ crypto/
   └─ types/
```

## Getting started

### Prerequisites

- Node.js 20+ and pnpm
- Python 3.11+
- PostgreSQL

### Backend

```bash
cd apps/backend
python -m venv venv
# Windows: venv\Scripts\activate    macOS/Linux: source venv/bin/activate
pip install -r requirements.txt     # or: pip install -e .
alembic upgrade head
python -m src.main                  # serves on http://127.0.0.1:8000
```

Create `apps/backend/.env`:

```
DATABASE_URL=postgresql+asyncpg://USER:PASSWORD@localhost:5432/DBNAME
JWT_SECRET_KEY=change-me
JWT_ALGORITHM=HS256
JWT_ACCESS_TOKEN_EXPIRE_MINUTES=60
JWT_REFRESH_TOKEN_EXPIRE_DAYS=7
API_V1_PREFIX=/api/v1
```

The interactive API docs are at `http://127.0.0.1:8000/docs`.

### Frontend

```bash
pnpm install                        # from the repo root
cd apps/frontend
npm run dev                         # http://localhost:5177
```

The dev server proxies `/api` to the backend. To point it elsewhere, set `VITE_API_ORIGIN` (default `http://127.0.0.1:8000`) in `apps/frontend/.env.local`.

Port 5177 is fixed (`strictPort`). If it is taken, stop the process holding it instead of changing the port.

### Demo data

```bash
cd apps/backend
python -m scripts.seed_dev_data
```

This creates a sample organization, doctor, pharmacy, pharmacist and patient. Seed credentials (development only):

| Role | Email | Password |
| --- | --- | --- |
| Doctor | `dr.seed@example.com` | `devpassword123` |
| Pharmacist | `pharmacist.seed@example.com` | `devpassword123` |

## Authentication status

| Portal | Sign-in |
| --- | --- |
| Verification (pharmacist) | Email and password via `POST /api/v1/auth/pharmacist/login`, session kept in `sessionStorage` |
| Doctor | No sign-in screen yet. Session token is held in memory |
| Admin | No sign-in screen yet |
| Organization | No sign-in screen yet |

Doctor, admin and organization portals need a login flow before they can load real data. Until then, a token can be minted with `create_access_token` in `src/core/security/jwt.py` for local testing only.

## Known issues

- **Seed script fails on enum types.** Models use native Postgres enums, but the migrations create `String` columns. Align them (for example `native_enum=False` on the model enums) or add a migration.
- **Photo verification returns 500** on upload. Check the backend traceback and the OCR and AI dependencies and keys.
- **Missing endpoints.** `/pharmacies/me/suspension` and login routes for doctor, admin and organization do not exist yet.
- **Generated client.** `packages/api-client/src/generated/openapi-client.ts` is generated. Fix path mismatches in the backend spec and regenerate it instead of editing it by hand.

## Development notes

- All URLs live in `apps/frontend/src/shared/constants/routes.ts`. Keep it in step with `router.tsx` when adding a route.
- Each portal has its own API client (`portals/<name>/api/client.ts`) that handles 401 responses. Send users to that portal's own sign-in, not to Home.
- The doctor portal's Redux store is scoped to `/doctor` and cannot be read by other portals.
- Never commit `.env`, `.env.local`, tokens, or helper scripts that mint tokens.

## License

Add your license here.
