# Astrology Admin Dashboard

A full-stack admin dashboard for managing categories, products, languages, and product translations for an astrology/e-commerce system, backed by a single MySQL database.

## Tech Stack

### Backend
- **Django** + **Django REST Framework**
- **MySQL** (single production database; no SQLite fallback)
- **JWT Authentication** (djangorestframework-simplejwt, rotating refresh tokens + blacklist)
- **CORS** configured for the React frontend
- Uniform API error envelope (`apps/core/exceptions.py`) and throttling (login 10/min, anonymous 120/hr, user 1000/hr)

### Frontend
- **React** + **TypeScript**
- **Vite** build tool
- **Tailwind CSS** for styling
- **React Router** for routing
- **Lucide React** icons, **Headless UI** accessible components
- **Axios** for API calls (with automatic access-token refresh on 401)
- **React Hot Toast** for notifications

## Project Structure

```
astro-admin/
├── backend/
│   ├── config/              # Django settings, URLs, WSGI/ASGI
│   ├── apps/
│   │   ├── core/            # Base models, exception handler, dashboard, test runner
│   │   ├── admins/          # Admin auth (login, refresh, me)
│   │   ├── categories/      # Category management
│   │   ├── products/        # Product & SKU management
│   │   ├── languages/       # Language listing
│   │   └── translations/    # Product translations
│   ├── scripts/             # Provisioning, merge, admin-password scripts
│   ├── database/
│   │   ├── schema.sql       # Reference SQL dump (Laravel import source)
│   │   └── backup/          # Backups + merge reports
│   ├── db.sqlite3           # Read-only merge source (kept unchanged)
│   ├── manage.py
│   ├── requirements.txt
│   ├── .env                 # Environment variables (not committed)
│   └── .env.example
└── frontend/
    ├── src/
    │   ├── components/
    │   │   ├── ui/          # Reusable UI components (Table, Modal, Badge, ...)
    │   │   └── layout/      # Sidebar, Header, PageContainer
    │   ├── pages/
    │   │   ├── Login/
    │   │   ├── Dashboard/
    │   │   ├── Categories/
    │   │   ├── Products/
    │   │   └── Languages/
    │   ├── services/        # API layer
    │   ├── hooks/           # Custom React hooks
    │   ├── types/           # TypeScript types
    │   ├── utils/           # Utility functions
    │   └── router/          # React Router config
    ├── .env
    └── .env.example
```

## Prerequisites

- Python 3.11+
- Node.js 18+
- MySQL 8.0+
- A `root`-capable MySQL user (local dev only)

## Setup

### 1. Database

The production database is named `astro` and comes from `backend/database/schema.sql` (the original Laravel MySQL dump). The migration/consolidation work already applied to the local `astro` database includes:

- `admins` auth columns added (`is_staff`, `is_superuser`, `is_active`, `last_login`) and the super admin flagged.
- Language codes fixed to the canonical 11-language list (see `apps/languages/seed_data.py`).
- Django-mapped primary keys normalized from `bigint unsigned` to signed `bigint`, and the dump's FK constraints rebuilt so Django models work against the schema.
- Every row in `backend/db.sqlite3` has been merged into MySQL as a **new** record (never overwriting MySQL rows); see `scripts/migrate_sqlite_to_mysql.py`.

To recreate from scratch (destructive, local dev only):

```bash
mysql -uroot -proot -e "CREATE DATABASE astro CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci"
mysql -uroot -proot astro < backend/database/schema.sql
python backend/scripts/provision_mysql_schema.py --apply
python backend/scripts/migrate_sqlite_to_mysql.py --apply --verify
python backend/scripts/reset_admin_password.py   # resets admin@admin.com
```

### 2. Backend

```bash
cd backend
cp .env.example .env     # then fill in DB credentials
python -m pip install -r requirements.txt
python manage.py migrate
python manage.py test --noinput   # 21 tests, uses a cloned test_astro DB
python manage.py runserver
```

Backend runs at http://127.0.0.1:8000

### 3. Frontend

```bash
cd frontend
cp .env.example .env
npm install
npm run dev
```

Frontend runs at http://localhost:5173

## Authentication

- JWT access token: 1 hour
- Refresh token: 7 days, **rotating** (each refresh mints a new pair and blacklists the old)
- Login is throttle-capped (10 attempts/minute per IP)
- Default admin: `admin@admin.com` / `admin123` (set via `scripts/reset_admin_password.py`; change it after first login)

## API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/auth/login/` | Admin login |
| POST | `/api/auth/refresh/` | Rotate refresh token (returns new access + refresh + admin) |
| GET | `/api/auth/me/` | Get current admin |
| GET | `/api/dashboard/stats/` | Dashboard statistics (real counts) |
| GET/POST | `/api/categories/` | List/create categories (paginated, searchable) |
| GET/PATCH/DELETE | `/api/categories/{id}/` | Detail/update/deactivate (status=false) |
| GET/POST | `/api/products/` | List/create products (paginated, searchable) |
| GET/PATCH/DELETE | `/api/products/{id}/` | Detail/update/soft-delete (deleted_at; hidden from list/detail) |
| GET | `/api/languages/` | List languages |
| GET/POST | `/api/translations/products/{product_id}/` | List/create product translations |
| PATCH/DELETE | `/api/translations/products/{product_id}/{id}/` | Update/delete translation |

Queries accept `search`, `status`, `product_type`/`category_type`, `ordering` (whitelisted), `page`, and `page_size`. Invalid `page` values fall back to `1`; invalid ordering is ignored.

## Features

### Dashboard
- 6 stat cards (total/active categories, total/active products, languages, translations) powered by `/api/dashboard/stats/`
- Product status breakdown and recent products

### Categories
- Paginated, searchable table with filters
- Hierarchical categories (parent/child) with child counts (annotated, no N+1)
- Category types: Service, Product, Remedy, Puja, All
- Modal forms for create/edit, detail view modal, deactivate

### Products
- Paginated, searchable table with status/type filters
- Server-generated unique SKUs (e.g. `RUD-0001`)
- Modal forms for create/edit including product-category mapping
- Detail page with images, variations, and translation management per language
- Soft delete (removed from list/detail)

### Languages
- Table of the 11 canonical languages with native names, codes, and status
- Quick stats (total, active, languages-with-translations)

## Database Notes

Tables adopted from the legacy import are `managed=False` (never auto-DDL'd):

- `admins`, `categories`, `category_product`, `languages`, `products`, `product_variations`, `product_images`

Django-managed (`managed=True`):

- `product_translations`
- `django_*` and `admins_groups`/`admins_user_permissions` bridge tables

`apps/core/test_runner.py` clones the legacy tables from `astro` into the test database so `manage.py test` runs against the real schema without touching production data.

## Environment Variables

### Backend `.env` (see `.env.example`)
```env
DB_NAME=astro
DB_USER=root
DB_PASSWORD=root
DB_HOST=127.0.0.1
DB_PORT=3306
DB_CONN_MAX_AGE=60
DJANGO_SECRET_KEY=...
DEBUG=True
ALLOWED_HOSTS=localhost,127.0.0.1
CORS_ALLOWED_ORIGINS=http://localhost:5173
THROTTLE_LOGIN=10/minute
THROTTLE_ANON=120/hour
THROTTLE_USER=1000/hour
ADMIN_PASSWORD=admin123
```

### Frontend `.env` (see `.env.example`)
```env
VITE_API_BASE_URL=http://127.0.0.1:8000/api
```

## Running the Application

1. Ensure MySQL is running and `astro` exists.
2. Backend: `cd backend && python manage.py runserver`
3. Frontend: `cd frontend && npm run dev`
4. Open http://localhost:5173 and log in with `admin@admin.com` / `admin123`

## Testing

### Backend
```bash
cd backend
python manage.py test --noinput
```
21 tests covering SKU generation, category/product API behavior, soft deletes, invalid page/ordering guards, and the dashboard endpoint. Requires the MySQL test runner (legacy tables are cloned automatically).

### Frontend
```bash
cd frontend
npm run lint    # oxlint
npm run build   # tsc -b + vite build
```

## Production Notes

1. Set `DEBUG=False` and a strong `DJANGO_SECRET_KEY`.
2. Configure `ALLOWED_HOSTS` and `CORS_ALLOWED_ORIGINS` (no `testserver`).
3. Change the admin password from the default.
4. `python manage.py collectstatic`
5. Serve the Django app with a production WSGI server and the built frontend from `frontend/dist`.#   a s t r o a d m i n 2  
 