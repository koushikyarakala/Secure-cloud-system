# Secure Cloud — Encrypted File Storage System

A security-first cloud file storage system built with **React + Express.js + PostgreSQL**. Every file is encrypted with **AES-256-GCM** before it touches disk, integrity is verified with **SHA-256** on every download, and every action is recorded in a tamper-evident audit log.

> Capstone MVP implementing the full project proposal: secure accounts, reliable upload/download, encryption, integrity verification, folders, sharing/access control, audit logging, versioning, and secure deletion.

---

## ✨ Features

| Area | Implementation |
|---|---|
| **Accounts** | Register / login / logout · bcrypt password hashing (cost 12) · JWT in httpOnly cookie · 7-day sessions |
| **File upload** | Multipart upload via multer · 50 MB limit · per-file AES-256-GCM key · envelope-wrapped with master key |
| **File download** | On-the-fly decryption · SHA-256 integrity verification · refused if tampered |
| **Folders** | Hierarchical · breadcrumbs · recursive secure delete |
| **Sharing** | Per-user, per-file · read/write permissions · revocable · access control on every operation |
| **Versioning** | Re-upload creates a new version · previous versions preserved · one-click restore |
| **Audit log** | Every action recorded: register, login, upload, download, share, revoke, delete, restore |
| **Secure deletion** | Overwrite with random → zeros → unlink · version history also wiped |
| **Security Center** | Transparent threat-model view of what's protected and what isn't yet |

---

## 🏗 Tech Stack

| Layer | Tech |
|---|---|
| Frontend | React 18 + Vite + Zustand + lucide-react + sonner (toasts) |
| Backend | Express.js 4 + node-postgres (`pg`) + bcryptjs + jsonwebtoken + multer |
| Database | PostgreSQL 16 (via Docker, or install locally) |
| Storage | Local filesystem (`backend/storage/files/`) — ciphertext only |

> **No TypeScript. No Next.js. No Prisma.** Plain JavaScript throughout, exactly as you requested.

---

## 📦 Project Structure

```
secure-cloud/
├── backend/
│   ├── src/
│   │   ├── server.js              ← Express app entry
│   │   ├── db.js                  ← pg Pool + query helpers
│   │   ├── auth.js                ← bcrypt + JWT
│   │   ├── crypto.js              ← AES-256-GCM + SHA-256 + envelope encryption
│   │   ├── storage.js             ← filesystem + secure delete
│   │   ├── audit.js               ← audit log helper
│   │   ├── middleware/
│   │   │   └── auth.js            ← requireAuth middleware
│   │   └── routes/
│   │       ├── auth.js            ← /api/auth/{register,login,logout,me}
│   │       ├── folders.js         ← /api/folders
│   │       ├── files.js           ← /api/files (upload, list, download, delete, versions, share)
│   │       ├── shares.js          ← /api/shares (list, revoke)
│   │       ├── audit.js           ← /api/audit
│   │       └── users.js           ← /api/users (search for share)
│   ├── storage/files/             ← encrypted file bytes (gitignored)
│   ├── schema.sql                 ← PostgreSQL schema
│   ├── package.json
│   ├── .env.example
│   └── .gitignore
│
├── frontend/
│   ├── src/
│   │   ├── App.jsx
│   │   ├── main.jsx
│   │   ├── api.js                 ← fetch wrapper
│   │   ├── store.js               ← Zustand store
│   │   ├── styles.css             ← all styles (no CSS framework)
│   │   └── components/
│   │       ├── AuthScreen.jsx
│   │       ├── Dashboard.jsx
│   │       ├── FileBrowser.jsx
│   │       ├── UploadDialog.jsx
│   │       ├── NewFolderDialog.jsx
│   │       ├── ShareDialog.jsx
│   │       ├── VersionDialog.jsx
│   │       ├── SharedView.jsx
│   │       ├── AuditLog.jsx
│   │       ├── SecurityCenter.jsx
│   │       └── ui.jsx             ← Modal + ConfirmDialog
│   ├── index.html
│   ├── vite.config.js
│   ├── package.json
│   └── .gitignore
│
├── docker-compose.yml             ← PostgreSQL 16
├── README.md                      ← this file
└── .gitignore
```

---

## 🚀 Quick Start (5 minutes)

### Prerequisites
- **Node.js 18+** — <https://nodejs.org>
- **Docker** (recommended for PostgreSQL) — <https://docker.com>
  - OR a local PostgreSQL 14+ install

### Step 1 — Start PostgreSQL

```bash
cd secure-cloud
docker compose up -d
```

This starts PostgreSQL 16 on `localhost:5432` with user `postgres`, password `postgres`, database `securecloud`.

<details>
<summary><b>Without Docker</b> (use your own PostgreSQL)</summary>

```bash
createdb securecloud
# Edit backend/.env to point at your local Postgres
```
</details>

### Step 2 — Set up the backend

```bash
cd backend
cp .env.example .env
```

Now generate the two secrets and paste them into `.env`:

```bash
# Generate a JWT secret
openssl rand -hex 32

# Generate a master encryption key
openssl rand -hex 32
```

Your `.env` should look like:

```
PORT=4000
DATABASE_URL=postgres://postgres:postgres@localhost:5432/securecloud
JWT_SECRET=<your-64-hex-chars-here>
MASTER_ENCRYPTION_KEY=<your-64-hex-chars-here>
STORAGE_DIR=./storage
```

Install dependencies and initialize the database:

```bash
npm install
npm run db:init    # runs schema.sql against your Postgres
npm run dev        # starts nodemon-style: node --watch src/server.js
```

The API will be running at <http://localhost:4000>. Health check: <http://localhost:4000/api/health>

### Step 3 — Set up the frontend

In a new terminal:

```bash
cd frontend
npm install
npm run dev
```

The React app will be running at **<http://localhost:5173>**.

Vite proxies `/api/*` to the backend at `localhost:4000`, so the cookie-based auth just works in the browser.

### Step 4 — Try it!

1. Open <http://localhost:5173>
2. Register an account (any email + 8+ char password)
3. Upload a file — it's encrypted with AES-256-GCM before touching disk
4. Download it back — decrypted on the fly, integrity verified
5. Register a second user, share a file with them
6. Check the Audit Log — every action is recorded
7. Check the Security Center — transparent threat-model view

---

## 🛡 How encryption works (envelope encryption)

```
plaintext file
      │
      ▼
[1] random 16-byte salt ──► HMAC-SHA256(masterKey, salt) ──► per-file key (32 B)
                                                                    │
[2] wrap per-file key with masterKey (AES-256-GCM)  ◄──────────────┘
    → stored as `key_cipher` in DB
                                                                    │
[3] encrypt plaintext with per-file key (AES-256-GCM)  ◄───────────┘
    → ciphertext + IV + auth-tag stored on disk + DB

[4] SHA-256(plaintext) → stored as `sha256` in DB (integrity check on download)
```

The master key lives in `.env` and never touches the database. Each file gets its own random key, so compromising one file's key does not compromise others.

---

## 🧪 Verifying the encryption

After uploading a file, check that the bytes on disk are ciphertext (not plaintext):

```bash
# Upload "secret.txt" through the UI, then:
ls backend/storage/files/
# → 40fc52e6176cca78dd51e29cba7f187a179f9cf340831888509869657bb91db1

grep -c "your-plaintext-content" backend/storage/files/*
# → 0  (the plaintext is NOT on disk anywhere)
```

---

## 📚 API Reference

### Auth
| Method | Endpoint | Description |
|---|---|---|
| POST | `/api/auth/register` | Create account · `{ name, email, password }` |
| POST | `/api/auth/login` | Sign in · `{ email, password }` |
| POST | `/api/auth/logout` | Clear session cookie |
| GET | `/api/auth/me` | Get current user |

### Folders
| Method | Endpoint | Description |
|---|---|---|
| GET | `/api/folders?parentId=root\|<id>` | List folders in a folder |
| POST | `/api/folders` | Create folder · `{ name, parentId }` |
| DELETE | `/api/folders/:id` | Recursively delete folder + secure-delete all files |

### Files
| Method | Endpoint | Description |
|---|---|---|
| GET | `/api/files?folderId=root\|<id>&includeShared=true` | List files |
| POST | `/api/files` | Upload (multipart) · fields: `file`, `folderId`, optional `asNewVersion` + `existingFileId` |
| GET | `/api/files/:id` | File metadata |
| DELETE | `/api/files/:id` | Secure-delete a file |
| GET | `/api/files/:id/download` | Download (decrypts + verifies integrity) |
| GET | `/api/files/:id/versions` | List version history |
| POST | `/api/files/:id/versions` | Restore a version · `{ versionId }` |
| GET | `/api/files/:id/share` | List active shares (owner only) |
| POST | `/api/files/:id/share` | Grant share · `{ toEmail, permissionLevel }` |

### Shares
| Method | Endpoint | Description |
|---|---|---|
| GET | `/api/shares` | List shares given + received |
| DELETE | `/api/shares?id=<shareId>` | Revoke a share |

### Audit + Users
| Method | Endpoint | Description |
|---|---|---|
| GET | `/api/audit?limit=100&offset=0&action=upload` | Audit log |
| GET | `/api/users?q=<query>` | Search users (for share autocomplete) |

---

## 🛠 Common Issues

**`FATAL: DATABASE_URL is not set`**
→ You forgot to copy `.env.example` to `.env` in the backend folder.

**`MASTER_ENCRYPTION_KEY must be 32 bytes`**
→ Generate one with `openssl rand -hex 32` and paste it into `.env`.

**`ECONNREFUSED 127.0.0.1:5432`**
→ PostgreSQL isn't running. Start it with `docker compose up -d`.

**`relation "users" does not exist`**
→ You haven't run the schema. Run `npm run db:init` from the backend folder.

**CORS errors in the browser**
→ Make sure the backend is running on port 4000 and the frontend on 5173. The Vite proxy handles `/api/*` automatically.

---

## 🚢 Production deployment notes

- Set `NODE_ENV=production` so the JWT cookie becomes `Secure` (HTTPS only).
- Put the backend behind HTTPS (nginx, Caddy, or a managed proxy).
- Use a strong `JWT_SECRET` and `MASTER_ENCRYPTION_KEY` — store them in a secrets manager, not in a committed `.env`.
- Run PostgreSQL as a managed service (RDS, Supabase, Neon, etc.).
- Back up `backend/storage/files/` regularly — without those ciphertext files, the database records are useless.
- Consider rotating the master key yearly (requires re-wrapping every file key — a planned feature).

---

## 📝 Capstone report talking points

- **Confidentiality**: AES-256-GCM with per-file envelope encryption.
- **Integrity**: SHA-256 verified on every download with constant-time compare.
- **Access control**: Every file operation checks ownership or an active share.
- **Accountability**: Full audit log of every security-relevant action.
- **Secure deletion**: Random-overwrite + zero-overwrite + unlink.
- **Threat model**: Honest about what's protected (disk theft, tampering, unauthorized access) and what isn't yet (malicious server operator — would need client-side encryption).
- **Performance overhead**: Encryption adds ~5-15ms per MB on a typical machine. Measure with `time curl -X POST ...` for upload and `time curl ... /download` for download.

---

## 📄 License

MIT — use this for your capstone, your portfolio, or anything else.
