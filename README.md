# Employee Directory

A small full-stack app for managing employee records: add, view, edit and delete, with search and pagination.

**Live demo:**https://agatis.vercel.app/



**Stack:** React 18 · TypeScript · Vite · Axios · plain CSS · PHP 7.4+ (PDO) · MySQL

---

## Features


- Create, list, update and delete employees (name, email, job role) (note : when you edit the data like when you click on edit it shows the users data in the left side form you can edit from there )
- Search by name, email or role (debounced, handled server-side)
- Server-side pagination (8 per page)
- Client-side and server-side validation with inline field errors
- Duplicate email detection (`409 Conflict`, surfaced on the email field)
- Delete confirmation dialog, toast feedback, loading skeletons, empty and error states
- Responsive layout (table collapses to cards on mobile), keyboard focus styles, reduced-motion support

## Project structure

```
.
├── frontend/              React + TypeScript source (Vite)
│   └── src/
│       ├── api.ts         Axios instance and typed API calls
│       ├── types.ts       Shared types
│       ├── App.tsx        State, data loading, CRUD handlers
│       └── components/    EmployeeForm, EmployeeList, Pagination, ConfirmDialog, Avatar
└── deploy/                Ready-to-upload build (this is what gets hosted)
    ├── index.html, assets/    Compiled frontend
    └── api/
        ├── index.php      REST API (routing, validation, DB access)
        ├── config.php     Database credentials
        ├── schema.sql     Table definition
        └── .htaccess      Routes /api/* to index.php
```

`npm run build` writes the compiled frontend straight into `deploy/`, next to the PHP API, so one folder holds everything that needs to be hosted.

## Deployment

**Requirements:** PHP 7.4+ with PDO MySQL, MySQL or MariaDB, Apache with `mod_rewrite`.

1. Create an empty MySQL database and a user with all privileges on it.
2. Edit `deploy/api/config.php`:
```php
   'db_host' => 'localhost',
   'db_name' => 'your_database_name',
   'db_user' => 'your_database_user',
   'db_pass' => 'your_database_password',
```
3. Upload the contents of `deploy/` to your web root or any subfolder (for example `public_html/directory/`).

No migration step is needed. On the first request the API creates the `employees` table from `schema.sql` and inserts a few sample rows. The frontend uses relative URLs, so it works from any folder without configuration.

Do not commit real credentials. Keep the placeholder values in `config.php` in version control.

## API

Base path: `/api`. All requests and responses use JSON.

| Method | Endpoint | Description | Success |
|---|---|---|---|
| GET | `/employees?q=&page=&limit=` | List employees, newest first. `limit` is capped at 50. | `200` |
| GET | `/employees/{id}` | Fetch one employee | `200` |
| POST | `/employees` | Create an employee | `201` |
| PUT | `/employees/{id}` | Update an employee | `200` |
| DELETE | `/employees/{id}` | Delete an employee | `204` |

**List response**
```json
{
  "data": [{ "id": 1, "name": "Aarav Mehta", "email": "aarav.mehta@example.com", "role": "Frontend Developer", "created_at": "2026-10-13 06:18:33" }],
  "meta": { "total": 6, "page": 1, "limit": 8, "pages": 1 }
}
```

**Error responses** use a consistent shape: `{ "error": "message", "errors": { "field": "message" } }`

| Status | Meaning |
|---|---|
| 400 | Request body is not valid JSON |
| 404 | Route or employee not found |
| 405 | Method not allowed |
| 409 | Email already exists |
| 422 | Validation failed (per-field messages in `errors`) |
| 500 | Server or database error (details are logged, not exposed) |

CORS is enabled (`Access-Control-Allow-Origin` is configurable in `config.php`), and `OPTIONS` preflight requests are handled.

## Database schema

```sql
CREATE TABLE employees (
  id         INT UNSIGNED NOT NULL AUTO_INCREMENT,
  name       VARCHAR(100) NOT NULL,
  email      VARCHAR(150) NOT NULL,
  role       VARCHAR(100) NOT NULL,
  created_at TIMESTAMP    NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  UNIQUE KEY uq_employees_email (email),
  KEY idx_employees_name (name)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

The unique key on `email` enforces uniqueness at the database level, which avoids race conditions that an application-level check would have. The index on `name` supports search and lookups.

## Local development

```bash
cd frontend
npm install
cp .env.example .env     # set VITE_API_URL to your API, e.g. http://localhost:8000/api/index.php
npm run dev
```

To run the API locally, point `deploy/api/config.php` at a local MySQL database and start PHP's built-in server from the repository root:

```bash
php -S localhost:8000 -t deploy
```

Rebuild the production bundle with `npm run build`. Output goes to `deploy/`.

## Design decisions

- **Prepared statements everywhere** (PDO, real prepared statements, no emulation) to prevent SQL injection. `LIKE` wildcards in the search term are escaped.
- **Validation on both sides.** The client validates for fast feedback, and the server validates because the client cannot be trusted.
- **Search and pagination run in SQL** (`LIKE`, `LIMIT`/`OFFSET`, `COUNT(*)`), so the browser never loads the full table.
- **Search is debounced (300 ms)**, and stale responses are discarded, so a slow earlier request cannot overwrite a newer one.
- **No state library.** The app is small enough that `useState` and `useEffect` are the simpler choice.
- **Typed API layer.** Components never call Axios directly, and error parsing is centralised in `api.ts`.
- **Zero-config deploy.** Auto-creating the table and using relative URLs means setup is editing four values.

## Known limitations and next steps

- No authentication or authorisation. It was out of scope, and the API is open. In production I would add server-side auth (sessions or JWT) and protect all `/employees` routes.
- CORS is `*` for convenience. Restrict it to the frontend origin in production.
- No rate limiting or automated tests. I would add PHPUnit for the API and React Testing Library for the UI.
- `LIKE '%term%'` cannot use a normal index. For large datasets, use a `FULLTEXT` index or a dedicated search service.

## Troubleshooting

| Symptom | Fix |
|---|---|
| `Database connection failed` | Check the values in `api/config.php`. On shared hosting, names usually carry an account prefix (for example `u123456_dbname`). |
| API returns 404 | Make sure `api/.htaccess` was uploaded (it is a hidden file) and the entry file is named `index.php`. |
| Works at `/api/index.php/employees` but not `/api/employees` | `mod_rewrite` or `.htaccess` is not active. Ask the host to enable it. |


