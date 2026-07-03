# SaaS Secondary School Management System — Master Prompt

> **Version:** 2.0 — Enhanced with architecture decisions, error handling, billing, and deployment  
> **Status:** Production-ready specification  
> **Target:** Django developer or AI code generator

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Tech Stack](#2-tech-stack)
3. [Multi-Tenant SaaS Architecture](#3-multi-tenant-saas-architecture)
4. [School Landing Page](#4-school-landing-page)
5. [Security Requirements](#5-security-requirements)
6. [User Roles & Permissions](#6-user-roles--permissions)
7. [Core System Features](#7-core-system-features)
8. [API Design Standards](#8-api-design-standards)
9. [Database Schema Guide](#9-database-schema-guide)
10. [Error Handling Strategy](#10-error-handling-strategy)
11. [SaaS Billing & Subscription Model](#11-saas-billing--subscription-model)
12. [Super-Admin Portal](#12-super-admin-portal)
13. [School Onboarding Flow](#13-school-onboarding-flow)
14. [File Storage](#14-file-storage)
15. [Frontend Pages](#15-frontend-pages)
16. [Django App Structure](#16-django-app-structure)
17. [Async Tasks & Background Jobs](#17-async-tasks--background-jobs)
18. [Deployment & DevOps](#18-deployment--devops)
19. [Additional Features](#19-additional-features)
20. [Expected Output](#20-expected-output)

---

## 1. Project Overview

Design and develop a **multi-tenant SaaS Secondary School Management Web Application** where multiple schools can register and independently manage their own portals.

Each school operates as an isolated tenant — no school can access another school's data. The system supports four user roles (Admin, Teacher, Student, Parent) with granular permissions per role.

The system is built for the **Nigerian secondary school context**, supporting:
- Class naming conventions: JSS1–JSS3, SS1–SS3
- Nigeria's A1–F9 grading scale
- Per-term and per-session result management
- Excel bulk upload for results

---

## 2. Tech Stack

| Layer | Technology | Notes |
|---|---|---|
| Backend framework | Django + Django REST Framework | Latest stable versions |
| Frontend | Tailwind CSS + Alpine.js | Minimal JS, utility-first CSS |
| Authentication | JWT (djangorestframework-simplejwt) | Access + refresh tokens |
| Database | MySQL | Per-tenant row-level isolation |
| Async tasks | Celery + Redis | Email, PDF generation, notifications |
| File storage | AWS S3 (or local in dev) | Per-tenant folder isolation |
| API docs | Swagger / OpenAPI (drf-spectacular) | Auto-generated from views |
| Caching | Redis | Session cache, rate limiting |
| Search | Django ORM filtering + pagination | No Elasticsearch needed at MVP |

---

## 3. Multi-Tenant SaaS Architecture

### Strategy: Shared Database, Row-Level Isolation

Use a **shared database with a `tenant_id` foreign key** on every tenant-scoped model. This is the recommended approach for MVP — it is simpler to build and easier to maintain than separate schemas.

**Do NOT use separate databases or separate schemas per school at this stage.**

### Tenant Model

```python
# schools/models.py
class Tenant(models.Model):
    name         = models.CharField(max_length=200)
    slug         = models.SlugField(unique=True)   # used as subdomain identifier
    is_active    = models.BooleanField(default=False)  # activated after approval
    plan         = models.ForeignKey('billing.Plan', on_delete=models.SET_NULL, null=True)
    created_at   = models.DateTimeField(auto_now_add=True)
```

### TenantAwareModel Base Class

All tenant-scoped models must inherit from this base class:

```python
# core/models.py
class TenantAwareModel(models.Model):
    tenant = models.ForeignKey('schools.Tenant', on_delete=models.CASCADE, db_index=True)

    class Meta:
        abstract = True
```

**Every model that belongs to a school — students, teachers, results, attendance, classes, subjects — must extend `TenantAwareModel`.**

### Tenant Middleware

```python
# core/middleware.py
class TenantMiddleware:
    """
    Resolves the current tenant from:
    1. Subdomain: schoolname.yourapp.com
    2. Header: X-Tenant-Slug (for API clients / Postman testing)
    Attaches tenant to request.tenant for use in views and querysets.
    """
```

### Queryset Filtering

Override `get_queryset()` in all ViewSets to filter by the current tenant:

```python
def get_queryset(self):
    return super().get_queryset().filter(tenant=self.request.tenant)
```

### Subdomain Routing

- Production: `schoolname.yourapp.com` — resolved by Nginx + middleware
- Development: `?tenant=schoolname` query param or `X-Tenant-Slug` header

---

## 4. School Landing Page

Each school has a public-facing landing page at `/{slug}/` or `{slug}.yourapp.com/`. No login required to view it.

### Required Sections

- School name, logo, and motto
- About / school description
- Contact details (address, phone, email)
- Admission information and requirements
- Login / Register buttons for Students, Parents, and Staff
- Announcements or news section (editable by school admin)
- Social media links (optional)

### Page Behaviour

- Publicly accessible — no authentication required
- School admin can edit landing page content from their dashboard
- Logo upload must validate: image files only, max 2 MB
- Announcements support rich text (use a simple textarea with basic formatting)

---

## 5. Security Requirements

> These are non-negotiable. Every item must be implemented.

### Input & Query Security

- Use **Django ORM exclusively** — no raw SQL queries with user input
- Sanitize all text inputs before saving (strip HTML tags where plain text is expected)
- Validate file uploads: Excel files only for result uploads, images only for logos

### HTTP Security Headers

Configure these in Django settings and Nginx:

```python
SECURE_BROWSER_XSS_FILTER       = True
SECURE_CONTENT_TYPE_NOSNIFF     = True
X_FRAME_OPTIONS                 = 'DENY'           # Clickjacking protection
SECURE_HSTS_SECONDS             = 31536000         # 1 year
SECURE_HSTS_INCLUDE_SUBDOMAINS  = True
SESSION_COOKIE_SECURE           = True
CSRF_COOKIE_SECURE              = True
# Content Security Policy (via django-csp)
CSP_DEFAULT_SRC = ("'self'",)
CSP_SCRIPT_SRC  = ("'self'",)
```

### JWT Authentication

- Use `djangorestframework-simplejwt`
- **Access token lifetime:** 15 minutes
- **Refresh token lifetime:** 7 days
- Enable token rotation (new refresh token issued on each refresh)
- Blacklist refresh tokens on logout
- Never store tokens in `localStorage` — use `httpOnly` cookies

### Role-Based Access Control (RBAC)

- Enforce permissions on **every API endpoint** using DRF permission classes
- No endpoint should be accessible without authentication except school landing pages and the public registration endpoints
- Tenant isolation must be verified on every request — a user from School A must never access School B's data even if they have the correct role

### Password Security

- Use Django's default `PBKDF2PasswordHasher` (do not downgrade)
- Minimum password length: 8 characters
- Enforce via serializer-level validation

### Rate Limiting & Brute Force Protection

- Limit login attempts: max 5 failed attempts per IP per 15 minutes (use `django-axes` or Redis-based throttling)
- Rate limit all API endpoints via DRF throttling classes
- Add CAPTCHA or cooldown on the public registration form

### Environment Variables

All secrets must be stored in environment variables — never in code or version control:

```
SECRET_KEY
DATABASE_URL
JWT_SECRET
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
EMAIL_HOST_PASSWORD
REDIS_URL
```

Use `python-decouple` or `django-environ` to load them.

---

## 6. User Roles & Permissions

### Super-Admin (SaaS Owner)

Manages the entire platform — not tied to any school.

- Approve or reject school registrations
- Activate / suspend / delete school tenants
- View platform-wide analytics (total schools, students, revenue)
- Manage subscription plans and billing
- **Cannot view any school's student or result data** (data isolation applies)

### Admin (School Owner / Manager)

Full control within their own school's tenant only.

- Create, update, and delete: Teachers, Students, Parents
- View all records: Results, Attendance, Reports
- Assign students to parents (or approve student-parent link requests)
- Manage: Classes, Subjects, Academic sessions, Terms
- Delete any teacher or record within the school
- Generate and export reports (academic and attendance)
- Edit school landing page content
- Manage school subscription and billing

### Teacher

- Registered only by the school Admin
- Login via JWT
- Update own personal biodata
- Mark student attendance for assigned classes
- Upload student results:
  - Excel bulk upload (validate structure before saving)
  - Manual single-entry form
- View assigned classes and their students
- Edit results only during the current open term (term must not be locked)

### Student

- Self-registration enabled
- System auto-generates a unique student ID on registration approval:
  - Format: `{CLASS_PREFIX}/{YEAR_SHORT}{SEQUENCE}` — e.g. `JSS1/26001`, `SS2/26045`
  - Class-based and auto-incremented per school per session
- Login securely via JWT
- View results filtered by term and academic session
- Update own biodata
- View own attendance records
- Cannot view other students' data

### Parent

- Self-registration enabled
- Link to child using Student ID (pending admin approval) or via admin assignment
- View linked child's results and attendance
- Update own personal information
- Can be linked to multiple children

---

## 7. Core System Features

### Academic Structure

```
Session (e.g. 2025/2026)
  └── Term (1st Term, 2nd Term, 3rd Term)
       └── Subject results per student
       └── Attendance records per student per day
```

- Admin creates sessions and terms
- Admin opens and locks terms (teachers can only upload results for open terms)
- Results are immutable once a term is locked (except for admin override)

### Student ID Generation

Use a Django signal or a dedicated service function triggered on student approval:

```python
def generate_student_id(student):
    """
    Format: CLASS_PREFIX/YY{SEQUENCE:03d}
    Example: JSS1/26001
    Sequence is per (school, class, year) — starts at 001 and increments.
    """
```

Class prefixes: `JSS1`, `JSS2`, `JSS3`, `SS1`, `SS2`, `SS3`

### Result Management

- Subjects, CA scores, exam scores, total scores, grades, remarks
- Grading scale follows **Nigeria's A1–F9 system**:

| Grade | Range | Remark |
|---|---|---|
| A1 | 75–100 | Excellent |
| B2 | 70–74 | Very Good |
| B3 | 65–69 | Good |
| C4 | 60–64 | Credit |
| C5 | 55–59 | Credit |
| C6 | 50–54 | Credit |
| D7 | 45–49 | Pass |
| E8 | 40–44 | Pass |
| F9 | 0–39 | Fail |

- System auto-computes grade and remark from total score
- Class position and average computed per subject per term
- Term report card generated as PDF

### Attendance Tracking

- Marked per student per class per day
- States: Present, Absent, Late, Excused
- Summary report per student per term

### Class Promotion

- At the end of a session, admin triggers class promotion
- Students move from JSS1 → JSS2 → JSS3 → SS1 → SS2 → SS3
- Student ID does **not** change on promotion — it is permanent
- Graduated students (after SS3) are archived, not deleted

---

## 8. API Design Standards

### Versioning

All API endpoints must be prefixed with a version:

```
/api/v1/students/
/api/v1/results/
/api/v1/attendance/
```

Use URL-based versioning. When breaking changes are needed, introduce `/api/v2/` without removing `/api/v1/`.

### Response Format

All responses must follow this consistent envelope:

```json
// Success
{
  "success": true,
  "data": { ... },
  "message": "Student created successfully"
}

// Paginated list
{
  "success": true,
  "data": {
    "results": [ ... ],
    "count": 120,
    "next": "/api/v1/students/?page=3",
    "previous": "/api/v1/students/?page=1"
  }
}
```

### Pagination, Filtering, Search

- Default page size: 20 records
- Support `?page=`, `?page_size=`, `?search=`, `?ordering=`
- Use `django-filter` for field-level filtering

### API Documentation

Use `drf-spectacular` to auto-generate OpenAPI schema. Expose at:
- `/api/schema/` — raw schema
- `/api/docs/` — Swagger UI
- `/api/redoc/` — ReDoc UI

---

## 9. Database Schema Guide

### Key Relationships

```
Tenant (School)
  ├── User (accounts) — all users belong to a tenant
  ├── Class — e.g. JSS1A, SS2B
  │    ├── Subject
  │    └── Student (many-to-many via Enrollment)
  ├── Session
  │    └── Term
  │         ├── Result (student + subject + term)
  │         └── Attendance (student + date + status)
  ├── Teacher
  │    └── TeacherSubjectAssignment (teacher + subject + class + term)
  └── ParentStudentLink (parent + student, approved by admin)
```

### Important Model Notes

- All models (except `Tenant`, `Plan`, `Subscription`, and super-admin `User`) must include `tenant = ForeignKey(Tenant)`
- Add `db_index=True` to `tenant`, `student`, `session`, and `term` foreign keys for query performance
- Use `soft delete` (add `is_deleted` + `deleted_at` fields) instead of hard deletes for audit trail
- All models should have `created_at` and `updated_at` timestamps

---

## 10. Error Handling Strategy

> All API endpoints must return errors using this unified format. No exceptions.

### Standard Error Response

```json
{
  "success": false,
  "error": {
    "code": "STUDENT_NOT_FOUND",
    "message": "No student found with ID JSS1/26001",
    "details": {}
  }
}
```

### Standard Error Codes

| Code | HTTP Status | When to use |
|---|---|---|
| `VALIDATION_ERROR` | 400 | Input data failed validation |
| `AUTHENTICATION_REQUIRED` | 401 | No valid JWT token provided |
| `PERMISSION_DENIED` | 403 | Valid token but wrong role/tenant |
| `NOT_FOUND` | 404 | Resource does not exist |
| `DUPLICATE_ENTRY` | 409 | Unique constraint violated (e.g. same ISBN/student ID) |
| `TERM_LOCKED` | 423 | Attempting to edit results for a locked term |
| `RATE_LIMITED` | 429 | Too many requests |
| `INTERNAL_ERROR` | 500 | Unexpected server error |

### Global Exception Handler

Register a custom exception handler in DRF settings:

```python
# settings.py
REST_FRAMEWORK = {
    'EXCEPTION_HANDLER': 'core.exceptions.custom_exception_handler',
}
```

```python
# core/exceptions.py
def custom_exception_handler(exc, context):
    """
    Converts all DRF exceptions to the standard error envelope.
    Logs 500 errors to the error monitoring service.
    Never exposes stack traces or internal details in production.
    """
```

---

## 11. SaaS Billing & Subscription Model

### Plans

Define at minimum three plans:

| Plan | Price | Students Limit | Features |
|---|---|---|---|
| Free Trial | ₦0 / 30 days | 50 students | All features, trial only |
| Basic | ₦15,000 / term | 300 students | Core features |
| Premium | ₦35,000 / term | Unlimited | All features + SMS notifications |

### Models

```python
# billing/models.py

class Plan(models.Model):
    name            = models.CharField(max_length=100)
    price           = models.DecimalField(max_digits=10, decimal_places=2)
    student_limit   = models.IntegerField(null=True)  # null = unlimited
    is_active       = models.BooleanField(default=True)

class Subscription(models.Model):
    tenant          = models.OneToOneField('schools.Tenant', on_delete=models.CASCADE)
    plan            = models.ForeignKey(Plan, on_delete=models.PROTECT)
    status          = models.CharField(choices=['active','expired','suspended','trial'])
    started_at      = models.DateTimeField()
    expires_at      = models.DateTimeField()
    auto_renew      = models.BooleanField(default=False)
```

### Enforcement

- A school with an expired subscription cannot log in (blocked at middleware level)
- A school on the Basic plan cannot add students beyond the limit (blocked at the API level with a clear error message)
- Super-admin can manually extend or override a subscription

### Payment Integration

Integrate **Paystack** (Nigeria-specific) for payment processing:
- Payment webhook updates `Subscription.status` and `expires_at`
- Generate and email receipt on successful payment

---

## 12. Super-Admin Portal

A separate dashboard accessible only to the SaaS owner (not tied to any school tenant).

### Features

- View all registered schools with status (active, trial, suspended, pending)
- Approve or reject new school registrations
- Activate, suspend, or permanently delete a school
- View platform-wide metrics:
  - Total schools, total students, total active subscriptions
  - Monthly revenue, churn rate
- Manage subscription plans (create, edit, deactivate)
- Manually extend or override a school's subscription
- View audit logs across all schools (admin actions only — no student data)

### Access Control

- Super-admin is a separate `User` with `is_superadmin=True` and `tenant=None`
- Super-admin cannot access any school's student, result, or attendance data
- Super-admin portal is served from the root domain (e.g. `yourapp.com/superadmin/`)

---

## 13. School Onboarding Flow

### Registration Steps

1. School representative visits the public registration page at `yourapp.com/register/school/`
2. Fills in: school name, address, state, contact person name, email, phone, and desired subdomain slug
3. Account created with `is_active=False`, subscription set to `trial`
4. Super-admin receives notification and reviews the application
5. Super-admin approves → school is activated, confirmation email sent with login link
6. School admin logs in and completes setup:
   - Upload school logo
   - Create academic session and first term
   - Create classes and subjects
   - Add teachers

### Initial Data Seeding

On school approval, auto-create:
- Default admin user account for the school
- A default academic session based on current year
- Basic class structure (JSS1–SS3) — admin can edit/remove

---

## 14. File Storage

### Strategy

- **Development:** Local filesystem (`MEDIA_ROOT`)
- **Production:** AWS S3 (use `django-storages`)

### Tenant Isolation in Storage

All files must be stored under a tenant-scoped path to prevent cross-tenant access:

```
s3://your-bucket/
  tenants/
    {tenant_slug}/
      logos/
        school_logo.png
      excel_uploads/
        results_term1_2026.xlsx
      report_cards/
        JSS1A_term1_2026.pdf
```

### File Upload Validation

- **Excel uploads (results):**
  - Allowed MIME types: `application/vnd.ms-excel`, `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`
  - Max file size: 5 MB
  - Validate column headers before processing any rows
  - Return row-level errors (e.g. "Row 12: Invalid student ID 'JSS1/99999'")
  - Process in a Celery background task for files with more than 50 rows

- **Logo / image uploads:**
  - Allowed types: JPEG, PNG, WebP
  - Max file size: 2 MB
  - Auto-resize to max 500×500px on upload

---

## 15. Frontend Pages

Build the following pages with Tailwind CSS + Alpine.js. All pages must be fully responsive (mobile-first).

### Public Pages (no login required)

| Page | URL | Description |
|---|---|---|
| Home / Marketing | `/` | SaaS landing page, features, pricing, CTA |
| School registration | `/register/school/` | Multi-step school signup form |
| School landing page | `/{slug}/` | Public profile for each school |
| Login | `/{slug}/login/` | Unified login for all roles |
| Student self-registration | `/{slug}/register/student/` | Student signup form |
| Parent self-registration | `/{slug}/register/parent/` | Parent signup form |

### Admin Dashboard Pages

| Page | Description |
|---|---|
| Dashboard overview | Stats: total students, teachers, pending approvals |
| Manage teachers | List, create, edit, deactivate teachers |
| Manage students | List, search, filter by class, view profile |
| Manage parents | List, manage parent-student links |
| Manage classes | Create and edit class groupings |
| Manage subjects | Assign subjects to classes and teachers |
| Academic sessions | Create sessions and terms, open/lock terms |
| Results overview | View results by class/term/session |
| Attendance reports | View attendance summaries |
| School settings | Edit landing page, upload logo, change slug |
| Billing | View plan, subscription status, payment history |

### Teacher Pages

| Page | Description |
|---|---|
| Dashboard | Assigned classes summary |
| Upload results | Excel bulk upload + manual entry form |
| Mark attendance | Per-class attendance form |
| View results | Results for assigned classes |

### Student Pages

| Page | Description |
|---|---|
| Dashboard | Quick summary of latest results and attendance |
| My results | Filter by session and term |
| My attendance | Summary table |
| My profile | Edit biodata |

### Parent Pages

| Page | Description |
|---|---|
| Dashboard | Children overview |
| Child results | Select child, view results by session/term |
| Child attendance | Summary per child |
| My profile | Edit biodata |

---

## 16. Django App Structure

Structure the project into modular, single-responsibility apps:

```
project_root/
├── config/                  ← Django settings, URLs, WSGI/ASGI
│   ├── settings/
│   │   ├── base.py
│   │   ├── development.py
│   │   └── production.py
│   └── urls.py
│
├── core/                    ← Shared base models, middleware, exceptions, utils
│   ├── models.py            ← TenantAwareModel base class
│   ├── middleware.py        ← TenantMiddleware
│   ├── exceptions.py        ← Global exception handler
│   ├── permissions.py       ← Custom DRF permission classes
│   └── pagination.py        ← Standard pagination config
│
├── accounts/                ← Custom User model, JWT auth, RBAC
├── schools/                 ← Tenant model, school profile, landing page content
├── billing/                 ← Plan, Subscription, payment webhook
├── students/                ← Student model, ID generation, self-registration
├── teachers/                ← Teacher model, subject assignments
├── parents/                 ← Parent model, parent-student links
├── academics/               ← Session, Term, Class, Subject models
├── results/                 ← Result model, grading, Excel upload, PDF export
├── attendance/              ← Attendance model, marking, summary reports
└── superadmin/              ← Super-admin views and serializers (separate from schools)
```

---

## 17. Async Tasks & Background Jobs

Use **Celery** with **Redis** as the message broker for all non-blocking operations.

### Tasks to Offload to Celery

| Task | Trigger |
|---|---|
| Send registration confirmation email | On school/user registration |
| Send result publication notification | When admin publishes term results |
| Process large Excel result uploads | Files with > 50 rows |
| Generate PDF report cards | On term lock |
| Send subscription expiry reminder | 7 days and 1 day before expiry |
| Send payment receipt email | On successful Paystack webhook |

### Setup

```python
# config/celery.py
app = Celery('schoolapp')
app.config_from_object('django.conf:settings', namespace='CELERY')
app.autodiscover_tasks()

# settings.py
CELERY_BROKER_URL    = env('REDIS_URL')
CELERY_RESULT_BACKEND = env('REDIS_URL')
```

---

## 18. Deployment & DevOps

### Server Stack

```
Internet
  └── Nginx (reverse proxy + subdomain routing + SSL termination)
       └── Gunicorn (WSGI server, 4 workers)
            └── Django application
       └── Celery worker (background tasks)
       └── Redis (broker + cache)
       └── MySQL (database)
```

### Subdomain Routing (Nginx)

```nginx
server {
    listen 443 ssl;
    server_name ~^(?P<slug>[a-z0-9-]+)\.yourapp\.com$;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header X-Tenant-Slug $slug;
        proxy_set_header Host $host;
    }
}
```

### Docker Compose (Development)

Provide a `docker-compose.yml` with services: `web`, `db` (MySQL), `redis`, `celery`.

### Environment Files

- `.env.example` committed to repo with all keys and placeholder values
- `.env` never committed — added to `.gitignore`

### CI/CD (Recommended)

- GitHub Actions workflow for: lint (flake8), tests (pytest), auto-deploy on merge to `main`

### SSL

- Use Let's Encrypt with Certbot for free SSL
- Configure wildcard certificate for `*.yourapp.com` to support all subdomains

---

## 19. Additional Features

### Must-Have at MVP

- [x] Email notification on result publication (Celery task)
- [x] PDF report card generation per student per term
- [x] Export results as CSV
- [x] Audit logs for admin actions (model-level with `django-auditlog`)
- [x] Search, filtering, and pagination on all list endpoints
- [x] Dashboard analytics (counts, summaries)
- [x] API documentation (Swagger + ReDoc)

### Post-MVP Roadmap

- [ ] SMS notifications (Termii or Africa's Talking for Nigeria)
- [ ] Parent-teacher messaging
- [ ] School fee management module
- [ ] Student timetable
- [ ] Mobile app (React Native consuming the same API)
- [ ] Real-time notifications (Django Channels + WebSockets)
- [ ] Multi-language support (English + Hausa/Igbo/Yoruba)

---

## 20. Expected Output

When handing this prompt to a developer or AI generator, the expected deliverables are:

### Code Deliverables

- [ ] Custom `User` model with role field and tenant link
- [ ] `TenantMiddleware` that resolves tenant from subdomain or header
- [ ] `TenantAwareModel` base class inherited by all tenant-scoped models
- [ ] All Django models with correct relationships, indexes, and soft delete
- [ ] DRF serializers with field validation (including grading logic)
- [ ] ViewSets with `get_queryset()` tenant filtering on every one
- [ ] Custom permission classes enforcing RBAC per role
- [ ] Global exception handler returning the standard error envelope
- [ ] JWT auth endpoints (login, refresh, logout with token blacklist)
- [ ] Student ID auto-generation via signal or service layer
- [ ] Excel upload endpoint with row-level validation
- [ ] PDF report card generation task (Celery)
- [ ] Celery task definitions for all async jobs
- [ ] Tailwind frontend templates for all pages listed in Section 15
- [ ] Nginx config with subdomain routing
- [ ] Docker Compose file for local development
- [ ] `.env.example` with all required environment variables
- [ ] Swagger/OpenAPI documentation

### Quality Standards

- Code must be modular — no business logic in views, no DB queries in templates
- Every API endpoint must have a corresponding unit test
- No hardcoded secrets, URLs, or configuration values
- All migrations must be reversible
- Production settings must have `DEBUG=False`, secure cookies, and HSTS enabled

---

> **Developer instruction:** Generate models, serializers, views, URLs, Celery tasks, and frontend pages step by step. Follow best practices. Ensure all code is modular, secure, tenant-isolated, and production-ready. Start with `core/`, then `accounts/`, then `schools/`, then remaining apps in dependency order.
