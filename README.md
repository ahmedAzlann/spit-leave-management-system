# SPIT Leave Management System

A full-stack web application for managing student event/leave applications at SPIT. Students apply for leave, which flows through a multi-step approval chain — Class Teacher → HOD → Dean — before the Attendance Coordinator marks the final attendance.

---

## Table of Contents

- [What It Does](#what-it-does)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [How the Workflow Works](#how-the-workflow-works)
- [Demo Credentials](#demo-credentials)
- [API Reference](#api-reference)
- [Running Locally (Development)](#running-locally-development)
- [Running with Docker (Production)](#running-with-docker-production)
- [Environment Variables](#environment-variables)
- [Database Schema](#database-schema)

---

## What It Does

| Role | What they can do |
|------|-----------------|
| **Student** | Apply for event leave, attach proof link, view all their applications and status |
| **Class Teacher** | View pending leaves, approve or reject with a comment |
| **HOD** | Same as teacher — sees leaves that the teacher already approved |
| **Dean** | Final approval stage before a leave is marked "approved" |
| **Attendance Coordinator** | Marks attendance for fully-approved leaves, completing the workflow |

Every approval step sends an **email notification** to the student (if SMTP is configured).

---

## Tech Stack

### Frontend
| Layer | Technology |
|-------|-----------|
| Framework | React 19 (Create React App) |
| Routing | React Router v7 |
| Styling | Vanilla CSS + custom theme |
| HTTP | Native `fetch` via `src/api/api.js` |

### Backend
| Layer | Technology |
|-------|-----------|
| Runtime | Node.js 20 + TypeScript |
| Framework | Express.js |
| ORM | Prisma 5 |
| Database | PostgreSQL 16 |
| Auth | JWT (Bearer token, stored in `localStorage`) |
| Validation | Zod |
| File Uploads | Multer |
| Email | Nodemailer |

### Infrastructure
| Component | Technology |
|-----------|-----------|
| Reverse Proxy | nginx (routes `/api/*` → backend, `/*` → frontend) |
| Frontend container | nginx Alpine serving static CRA build |
| Orchestration | Docker Compose |

---

## Project Structure

```
/
├── Dockerfile                  # Frontend multi-stage build (CRA → nginx)
├── docker-compose.yml          # Full-stack orchestration (4 services)
├── .dockerignore
├── .env                        # Frontend env (REACT_APP_API_URL)
├── .gitignore
│
├── docker/
│   ├── nginx/nginx.conf        # Reverse proxy config (port 80)
│   └── frontend/nginx.conf     # SPA nginx config (try_files fallback)
│
├── public/                     # CRA static assets
│
├── src/                        # React frontend
│   ├── api/api.js              # All fetch calls to the backend
│   ├── pages/
│   │   ├── Login.js
│   │   ├── StudentDashboard.js
│   │   ├── StudentApplications.js
│   │   ├── FacultyDashboard.js
│   │   └── CoordinatorDashboard.js
│   ├── components/
│   ├── styles/
│   └── App.js                  # Routes + ProtectedRoute guard
│
└── backend/
    ├── Dockerfile              # Backend multi-stage build (TS → node)
    ├── .dockerignore
    ├── .env                    # Backend secrets (not committed)
    ├── .env.example            # Copy this to .env
    ├── prisma/
    │   └── schema.prisma       # DB models: User, LeaveApplication, Subject, LeaveClass
    └── src/
        ├── index.ts            # Express entry point + middleware
        ├── config/
        │   ├── database.ts     # Prisma client singleton
        │   ├── validation.ts   # Zod schemas
        │   └── seed.ts         # Demo data seeder
        ├── middleware/
        │   ├── auth.middleware.ts    # JWT authenticate + authorize(roles)
        │   ├── validate.middleware.ts
        │   ├── error.middleware.ts
        │   └── logger.middleware.ts
        ├── controllers/
        │   ├── auth.controller.ts
        │   ├── student.controller.ts
        │   ├── faculty.controller.ts
        │   ├── coordinator.controller.ts
        │   └── upload.controller.ts
        ├── routes/
        │   ├── auth.routes.ts
        │   ├── student.routes.ts
        │   ├── faculty.routes.ts
        │   ├── coordinator.routes.ts
        │   └── upload.routes.ts
        └── utils/
            ├── email.ts        # Nodemailer + templates
            └── formatLeave.ts  # Prisma → frontend shape mapper
```

---

## How the Workflow Works

```
Student applies
      │
      ▼
 pending_teacher  ──reject──► rejected
      │
   Teacher approves
      │
      ▼
  pending_hod  ──reject──► rejected
      │
   HOD approves
      │
      ▼
 pending_dean  ──reject──► rejected
      │
   Dean approves
      │
      ▼
   approved
      │
   Coordinator marks attendance
      │
      ▼
   completed ✅
```

The frontend auto-redirects each role to the correct dashboard based on the JWT payload. A `ProtectedRoute` component in `App.js` guards every route.

---

## Demo Credentials

Seed the database first (`npm run db:seed` inside `backend/`), then use:

| Role | UID | Password |
|------|-----|----------|
| Student | `2023800110` | `2023800110` |
| Class Teacher | `TEACHER001` | `TEACHER001` |
| HOD | `HOD001` | `HOD001` |
| Dean | `DEAN001` | `DEAN001` |
| Coordinator | `COORD001` | `COORD001` |

---

## API Reference

All endpoints are prefixed with `/api`. The backend validates JWT on every protected route.

### Auth
| Method | Endpoint | Auth | Body / Notes |
|--------|----------|------|--------------|
| `POST` | `/api/auth/login` | — | `{ uid, password }` → returns `{ token, userId, name, role, ... }` |
| `GET` | `/api/auth/me` | Bearer | Returns current user |

### Student `(role: student)`
| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/student/profile` | Name, UID, class, mobile |
| `GET` | `/api/student/leaves` | All leave applications for the logged-in student |
| `POST` | `/api/student/leaves` | Submit a new leave application |

### Faculty `(role: teacher | hod | dean)`
| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/faculty/pending` | Leaves pending for your specific role |
| `POST` | `/api/faculty/approve/:leaveId` | Approve with `{ comment }` |
| `POST` | `/api/faculty/reject/:leaveId` | Reject with `{ comment }` |
| `GET` | `/api/faculty/history` | Approved / rejected history |

### Coordinator `(role: coordinator)`
| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/coordinator/approved` | Fully approved leaves, attendance not yet marked |
| `POST` | `/api/coordinator/mark-attendance/:leaveId` | Mark attendance + complete with `{ comment }` |
| `GET` | `/api/coordinator/completed` | Completed leaves |

### Upload
| Method | Endpoint | Body | Description |
|--------|----------|------|-------------|
| `POST` | `/api/upload/proof` | `multipart/form-data` (field: `file`) | Upload proof file, returns `{ url }` |

### Health
```
GET /api/health  →  { status: "ok", timestamp: "..." }
```

---

## Running Locally (Development)

This is the fastest way to work on the code. Both servers run natively with hot-reload.

### Prerequisites
- Node.js 20+
- A PostgreSQL database (local, Docker, or cloud — e.g. Neon)

### 1. Clone and install

```bash
git clone https://github.com/Nitin15122005/spit-leave-management-system.git
cd spit-leave-management-system

# Frontend dependencies
npm install

# Backend dependencies
cd backend && npm install && cd ..
```

### 2. Configure the backend

```bash
cp backend/.env.example backend/.env
# Edit backend/.env — set DATABASE_URL and JWT_SECRET at minimum
```

```env
PORT=5001
DATABASE_URL="postgresql://user:pass@host/dbname?sslmode=require"
JWT_SECRET=your_long_random_secret_here
FRONTEND_URL=http://localhost:3000
```

### 3. Run migrations and seed demo data

```bash
cd backend
npm run db:migrate   # applies Prisma migrations
npm run db:seed      # loads demo users and a sample leave
cd ..
```

### 4. Start the backend

```bash
cd backend && npm run dev
# → Server running on http://localhost:5001
```

### 5. Start the frontend (new terminal)

```bash
npm start
# → React app on http://localhost:3000
```

The frontend reads `REACT_APP_API_URL` from the root `.env` file (default: `http://localhost:5001/api`).

---

## Running with Docker (Production)

A single command builds all images and starts the full stack behind an nginx reverse proxy.

### Architecture

```
Browser
  └─► :80 (nginx reverse proxy)
          ├─ /api/*      → api container  (Express :5001)
          ├─ /uploads/*  → api container  (static files)
          └─ /*          → frontend container (nginx static, React build)
```

Only port **80** is exposed to the host. The database, API, and frontend containers communicate on Docker's internal network.

### Prerequisites
- Docker Desktop (or Docker Engine + Compose plugin)

### Steps

```bash
# 1. Clone the repo
git clone https://github.com/Nitin15122005/spit-leave-management-system.git
cd spit-leave-management-system

# 2. (Optional but recommended) Set a real JWT secret
#    Edit docker-compose.yml → api.environment.JWT_SECRET

# 3. Build images and start all 4 services
docker compose up --build -d

# 4. Seed demo data (first run only)
docker compose exec api npx ts-node src/config/seed.ts
# OR if running the compiled image:
docker compose exec api node -e "require('./dist/config/seed')"

# 5. Open the app
open http://localhost
```

### Useful commands

```bash
# Tail logs from all services
docker compose logs -f

# Tail only the API
docker compose logs -f api

# Check container health
docker compose ps

# Stop everything (keeps database volume)
docker compose down

# Stop and wipe all data (volumes deleted)
docker compose down -v

# Rebuild a single service after code changes
docker compose up --build -d frontend
```

### Services at a glance

| Container | Role | Exposed port |
|-----------|------|-------------|
| `leave_mgmt_proxy` | nginx reverse proxy | **80** (host) |
| `leave_mgmt_frontend` | React static (nginx) | internal only |
| `leave_mgmt_api` | Express + Prisma | internal only |
| `leave_mgmt_db` | PostgreSQL 16 | internal only |

---

## Environment Variables

### Frontend (`/.env`)

| Variable | Default | Description |
|----------|---------|-------------|
| `REACT_APP_API_URL` | `http://localhost:5001/api` | Backend API base URL (local dev). In Docker, overridden to `/api` via build-arg. |
| `REACT_APP_APP_NAME` | `SPIT Leave Management System` | App display name |
| `REACT_APP_ENV` | `development` | Environment label |

### Backend (`/backend/.env`)

| Variable | Required | Description |
|----------|----------|-------------|
| `PORT` | Yes | Port the Express server listens on (default `5001`) |
| `DATABASE_URL` | Yes | PostgreSQL connection string |
| `JWT_SECRET` | Yes | Secret for signing JWTs — use a long random string in production |
| `JWT_EXPIRES_IN` | No | Token expiry (default `7d`) |
| `FRONTEND_URL` | Yes | CORS allowed origin (e.g. `http://localhost:3000` or `http://localhost`) |
| `UPLOAD_DIR` | No | Directory for uploaded files (default `uploads`) |
| `MAX_FILE_SIZE_MB` | No | Max upload size in MB (default `10`) |
| `SMTP_HOST` | No | SMTP server — email notifications are skipped if not set |
| `SMTP_PORT` | No | SMTP port (default `587`) |
| `SMTP_USER` | No | SMTP login |
| `SMTP_PASS` | No | SMTP password / app password |
| `EMAIL_FROM` | No | Sender address shown in emails |

---

## Database Schema

Four models managed by Prisma:

```
User
├── id, uid (unique), name, email, password, role
├── department, class, mobile
└── → LeaveApplication[] (as student)

LeaveApplication
├── id, studentId → User
├── cached: studentName, studentUid, studentClass, studentMobile
├── event: eventName, organizedBy, venue, eventLink, proofLink
├── dates: eventDurationFrom/To, leaveDatesFrom/To, semester
├── status: pending_teacher | pending_hod | pending_dean | approved | rejected | completed
├── comments: teacherComment, hodComment, deanComment, coordinatorComment
├── attendanceMarked, declaration
└── → LeaveSubject[]

LeaveSubject
├── id, leaveId → LeaveApplication, subjectId → Subject
├── subjectCode, subjectName, theoryCount, labCount
└── → LeaveClass[]

LeaveClass
├── id, leaveSubjectId → LeaveSubject
├── date, timing, type (theory | lab), batch
```

Migrations are run automatically in Docker (`npx prisma migrate deploy` in the API container's `CMD`). For local dev, run `npm run db:migrate` inside `backend/`.



## 🚀 CI/CD & DevOps Implementation (My Contribution)

In addition to the core application, I implemented a complete CI/CD pipeline and containerized deployment for this project.

### 🔧 Tools & Technologies
- Jenkins (CI/CD automation)
- Docker & Docker Compose (containerization)
- Nginx (reverse proxy)
- GitHub (version control & trigger source)

### ⚙️ What I Implemented
- Created a Jenkins Pipeline to automate build and deployment
- Integrated GitHub repository with Jenkins using Poll SCM
- Containerized the entire application (frontend, backend, database, proxy)
- Automated deployment using Docker Compose
- Configured Nginx reverse proxy for routing:
  - `/api/*` → backend
  - `/*` → frontend

### 🔁 CI/CD Workflow

### 🛠️ Key Challenges Solved
- Fixed Docker access inside Jenkins container using Docker socket mounting
- Resolved permission issues with Docker daemon
- Fixed dependency issues (`npm ci` → `npm install`)
- Resolved nginx configuration mount issue in CI environment by switching to Dockerfile-based COPY

### ▶️ How to Run with Jenkins
1. Start Jenkins container
2. Create Pipeline job
3. Connect GitHub repository
4. Run pipeline → automatic deployment

### 📸 Demo
<img width="1600" height="850" alt="WhatsApp Image 2026-05-03 at 6 31 18 PM" src="https://github.com/user-attachments/assets/7ecc73f0-03a5-4d97-867b-502bb2769821" />


<img width="1600" height="850" alt="image" src="https://github.com/user-attachments/assets/761061f1-b26f-40dc-ab35-d0495f2ae721" />


<img width="1600" height="850" alt="WhatsApp Image 2026-05-03 at 6 38 07 PM" src="https://github.com/user-attachments/assets/2e6e7c03-e664-4c0d-a2e6-4615a1b6cfc0" />
