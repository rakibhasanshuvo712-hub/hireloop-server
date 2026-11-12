# HireLoop Server & Backend API

Modern backend for **HireLoop**, a production job portal built with Next.js 16, React 19, Better-Auth, MongoDB Native Driver, and Stripe.

---

## 🚀 Features & Architecture

### 1. MongoDB Native Connection & Pooling
- Singleton connection utility located in `lib/mongodb.ts`.
- Native `mongodb` driver with connection pooling (`maxPoolSize: 10`, `minPoolSize: 2`) targeting `hireloopdb`.
- Automatic database index creation on bootstrap:
  - **jobs**: `{ category: 1, type: 1 }`, `{ companyId: 1 }`, `{ status: 1 }`, `{ createdAt: -1 }`
  - **applications**: `{ seekerId: 1, jobId: 1 }` (unique compound index), `{ jobId: 1 }`, `{ companyId: 1 }`, `{ appliedAt: -1 }`
  - **saved_jobs**: `{ seekerId: 1, jobId: 1 }` (unique compound index), `{ seekerId: 1 }`
  - **companies**: `{ recruiterId: 1 }`, `{ status: 1 }`
  - **payments**: `{ userId: 1 }`, `{ stripeSessionId: 1 }`

---

### 2. Better-Auth Integration & Extended User Model
Configured in `lib/auth.ts` with `@better-auth/mongo-adapter`.
User metadata and session schemas extend the following fields:
- `role`: `"Seeker"` | `"Recruiter"` | `"Admin"` (default: `"Seeker"`)
- `status`: `"Active"` | `"Suspended"` (default: `"Active"`)
- `plan`: `"Free"` | `"Pro"` | `"Premium"` | `"Growth"` | `"Enterprise"` (default: `"Free"`)
- `companyId`: `ObjectId` (optional, linked if Recruiter)
- `resumeUrl`: `string` (candidate CV / resume URL)
- `headline`: `string`
- `bio`: `string`
- `skills`: `string[]`

---

### 3. Database Collections & TypeScript Models
All TypeScript models and collection handlers are implemented under `lib/types/database.ts`:
1. `jobs`: Comprehensive job specifications, category, type, salary range, responsibilities, requirements, benefits, and status.
2. `applications`: Tracks applicant details, resume, cover note, and multi-stage status (`Applied` -> `Under Review` -> `Shortlisted` -> `Offered` -> `Rejected`).
3. `companies`: Verified company profiles linked to recruiter IDs.
4. `saved_jobs`: Candidate bookmarked jobs.
5. `payments`: Payment records, plans, amounts, currency, and Stripe session identifiers.

---

### 4. API Endpoints

| Method | Endpoint | Description | Access |
|---|---|---|---|
| `GET` | `/api/jobs` | Search & filter jobs (`q`, `category`, `type`, `location`, `page`, `limit`) | Public |
| `GET` | `/api/jobs/[id]` | Fetch single job with company details | Public |
| `POST` | `/api/jobs` | Post new job (checks company approval and plan posting limits) | Recruiter / Admin |
| `PATCH` | `/api/jobs/[id]` | Update job or toggle status (`Active` / `Closed`) | Owning Recruiter / Admin |
| `DELETE` | `/api/jobs/[id]` | Delete job | Owning Recruiter / Admin |
| `POST` | `/api/jobs/[id]/save` | Toggle saved job bookmark for candidate | Authenticated Seeker |
| `GET` | `/api/applications` | List applications for current seeker or recruiter's company | Authenticated User |
| `POST` | `/api/applications` | Submit application (enforces monthly limits: Free: 3, Pro: 30, Premium: unlimited) | Authenticated Seeker |
| `PATCH` | `/api/applications/[id]/status` | Move candidate stage (`Applied`, `Under Review`, `Shortlisted`, `Offered`, `Rejected`) | Owning Recruiter / Admin |
| `GET` | `/api/companies` | List companies with search and filters | Public |
| `POST` | `/api/companies` | Register new company profile | Recruiter / Admin |
| `GET` | `/api/companies/[id]` | Fetch company profile with active jobs | Public |
| `POST` | `/api/checkout` | Create Stripe checkout session or process test payment | Authenticated User |
| `POST` | `/api/webhooks/stripe` | Handle `checkout.session.completed` webhook | Stripe / Webhook |
| `GET` | `/api/admin/metrics` | Platform metrics (users, recruiters, jobs, platform MRR, total revenue) | Admin |
| `PATCH` | `/api/admin/companies/[id]` | Approve or reject company registration | Admin |
| `PATCH` | `/api/admin/users/[id]` | Update user role or suspend account | Admin |
| `GET` | `/api/admin/users` | List platform users with search and filters | Admin |
| `GET/PATCH` | `/api/user/profile` | Read and update candidate profile and skills | Authenticated User |
| `GET` | `/api/user/saved-jobs` | List bookmarked jobs for candidate | Authenticated Seeker |
| `POST` | `/api/seed` | Seed initial companies and jobs into database | Development / Admin |

---

### 5. Plan Limits & Rules
- **Job Applications (Seekers)**:
  - Free: Max 3 applications per calendar month
  - Pro: Max 30 applications per calendar month
  - Premium: Unlimited applications
- **Job Postings (Recruiters)**:
  - Free: Max 3 active job posts (requires approved company)
  - Growth: Max 10 active job posts
  - Enterprise: Max 50 active job posts
- **Payload Validation**:
  - All request inputs validated with Zod schemas (`lib/validations/index.ts`).
- **Standardized Responses**:
  - Success: `{ "success": true, "data": ... }`
  - Failure: `{ "success": false, "error": "..." }`
