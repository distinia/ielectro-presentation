<p align="center">
  <img src="logo.png" alt="iElectro" width="88">
</p>

<h1 align="center">iElectro</h1>

<p align="center">One identity. Different products. Shared framework.</p>


Visit [the site](https://ielectro.altervista.org) and try the **demo.** Sign in on **Account** to use **Dyscover**:

- username: `demodemo`
- password: `demodemo`

iElectro is a small multi-app platform: a public company site, a single sign-on account, a social/publishing product (Dyscover), and a staff backoffice. Each app is its own Apache vhost. They share a `session_token` cookie on a common parent domain and boot through **Nesh** (`nesh/src`), a PHP 8 framework that maps URLs to pages and `/api/{service}` classes.

| App | Role | Database |
| --- | --- | --- |
| **Www** | Public company site | none (reads Admin APIs) |
| **Account** | Identity and SSO | `ielectro_account` |
| **Dyscover** | Social / publishing | `ielectro_dyscover` |
| **Admin** | Staff backoffice | `ielectro_admin` |

- [Architecture](#architecture)
- [Identity & sessions](#identity--sessions)
- [Data model](#data-model)
- [Nesh](#nesh)
- [API REST](#http-api)
- [Repository layout](#repository-layout)
- [Product tour](#product-tour)

---

## Product tour

### Www

| | | |
| --- | --- | --- |
| <img src="www/01.png" alt="www home" width="260"> | <img src="www/02.png" alt="www services" width="260"> | <img src="www/03.png" alt="www team" width="260"> |
| <img src="www/04.png" alt="www careers" width="260"> | <img src="www/05.png" alt="www news" width="260"> | |

### Account

| | | |
| --- | --- | --- |
| <img src="account/01.png" alt="account login" width="260"> | <img src="account/02.png" alt="account create" width="260"> | <img src="account/03.png" alt="account password recovery" width="260"> |
| <img src="account/04.png" alt="account home" width="260"> | <img src="account/05.png" alt="account profile" width="260"> | <img src="account/06.png" alt="account services" width="260"> |
| <img src="account/07.png" alt="account activity" width="260"> | | |

### Dyscover

| | | |
| --- | --- | --- |
| <img src="dyscover/01.png" alt="dyscover home" width="260"> | <img src="dyscover/02.png" alt="dyscover explore" width="260"> | <img src="dyscover/03.png" alt="dyscover inbox" width="260"> |
| <img src="dyscover/04.png" alt="dyscover activity" width="260"> | <img src="dyscover/05.png" alt="dyscover users" width="260"> | <img src="dyscover/06.png" alt="dyscover creator center" width="260"> |
| <img src="dyscover/07.png" alt="dyscover article" width="260"> | | |

### Admin

| | | |
| --- | --- | --- |
| <img src="admin/01.png" alt="admin home" width="260"> | <img src="admin/02.png" alt="admin news" width="260"> | <img src="admin/03.png" alt="admin team" width="260"> |
| <img src="admin/04.png" alt="admin careers" width="260"> | <img src="admin/05.png" alt="admin accounts" width="260"> | <img src="admin/06.png" alt="admin dyscover" width="260"> |

---

## Architecture

Four vhosts, one framework, three MySQL schemas. Uploads stay under each app’s `assets/`. Optional LLM calls go through `Nesh\AiClient`.

```mermaid
flowchart TB
  subgraph Browser
    U[User]
  end

  U --> WWW[www]
  U --> ACC[account]
  U --> DYS[dyscover]
  U --> ADM[admin]

  WWW --> Nesh
  ACC --> Nesh
  DYS --> Nesh
  ADM --> Nesh

  subgraph Nesh["Nesh (nesh/src)"]
    App[App::run]
    Pages[Pages]
    Api[Api]
    Id[Identity]
    App --> Pages
    App --> Api
    Api --> Id
  end

  ACC --> DA[(ielectro_account)]
  DYS --> DD[(ielectro_dyscover)]
  ADM --> DM[(ielectro_admin)]
  WWW -.->|public APIs: news, careers, team, views| ADM
  Id --> DA
```

**Front controller.** Each app `.htaccess` sends unknown paths to `index.php` and forbids `/database` and `/storage`. Path segment `0` is either `api` or a page name.

**SSO.** `Nesh\Identity` resolves `session_token` against `ielectro_account.account_sessions`.

```mermaid
flowchart LR
  Request --> Autoload[autoload.php]
  Autoload --> Start[Request::start]
  Start --> Apps[App instances]
  Apps --> Run[App::run]
  Run --> Backup[Backup::daily]
  Backup --> Route{api/?}
  Route -->|yes| Handle[Api::handle]
  Route -->|no| Render[Pages::render]
```

---

## Identity & sessions

Identity is owned by Account. Other apps only read it.

```mermaid
sequenceDiagram
  autonumber
  participant B as Browser
  participant A as Account
  participant DB as ielectro_account
  participant D as Dyscover / Admin / Www

  B->>A: POST /api/sessions (or Google OAuth)
  A->>DB: store SHA-256(session token)
  A-->>B: Set-Cookie session_token (parent domain, HttpOnly, Secure, SameSite=Lax)

  B->>D: GET or POST /api/…
  D->>DB: Identity lookup (unrevoked, unexpired)
  DB-->>D: account_id, username
  D-->>B: JSON or HTML
```

- Cookie domain: `COOKIE_DOMAIN` (shared parent of the four apps).
- `Identity::required()` → JSON 401 on protected APIs.
- Dyscover profiles: `dyscover_users.account_id` with role `user` / `moderator` and status `active` / `suspended` / `banned`.
- Admin HTML and staff APIs: active row in `ielectro_admin.team` (`Admin\Access`).
- Www HTML is public; `views/track` and `stats` are listed as public APIs.
- CSRF (`csrf_token` cookie, `X-CSRF-Token` header) on mutating authenticated calls.
- Rate limits on login, recovery, and careers apply.

Public Account routes: `oauth/google`, `recovery`, `recovery/reset`, `availability`, plus Nesh’s built-in `POST /api/sessions` and `POST /api/user`.

---

## Data model

`www` has no schema. Admin joins Account users for names. Dyscover never stores passwords.

```mermaid
erDiagram
  ACCOUNTS ||--o{ ACCOUNT_SESSIONS : issues
  ACCOUNTS ||--o| DYSCOVER_USERS : "account_id"
  ACCOUNTS ||--o| ADMIN_TEAM : "staff"
  DYSCOVER_USERS ||--o{ POSTS : publishes
  DYSCOVER_USERS ||--o{ ARTICLES : writes
  DYSCOVER_USERS ||--o{ FOLLOWS : follows
  ADMIN_TEAM ||--o{ NEWS : edits
  ADMIN_TEAM ||--o{ CAREERS : edits

  ACCOUNTS {
    int id
    string username
    string password_hash
  }
  ACCOUNT_SESSIONS {
    string token_hash
    datetime expires_at
  }
  DYSCOVER_USERS {
    int account_id
    string role
    string status
  }
  ADMIN_TEAM {
    int account_id
    string status
  }
```

| App | Database | SQL |
| --- | --- | --- |
| Account | `ielectro_account` | `account/database/schema/` |
| Dyscover | `ielectro_dyscover` | `dyscover/database/schema/` |
| Admin | `ielectro_admin` | `admin/database/schema/` |

Bootstrap files start with `CREATE DATABASE` in `0-bootstrap.sql`. The graphic installer at the site root runs them (`Database::install` skips a schema that already has tables). `Nesh\Query` uses mysqli prepared statements. `Nesh\Schema` quotes table names so Admin can join across schemas.

**Backup.** `Nesh\Backup` writes `{app}/database/backup/YYYY-MM-DD.sql` once per day and drops dumps older than seven days. Files are gitignored; Apache returns 403 for `/database`.

---

## Nesh

Nesh is not a public app. Every vhost loads `nesh/src/autoload.php`.

```mermaid
classDiagram
  class App {
    +api Api
    +pages Pages
    +run()
  }
  class Api {
    +publicApi
    +handle()
  }
  class Pages {
    +render()
  }
  class Identity {
    +required()
  }
  class Database {
    +install()
  }
  class Backup {
    +daily()
  }
  class AiClient {
    +chat()
  }
  App --> Api
  App --> Pages
  Api --> Identity
  App --> Database
  App --> Backup
```

Boot order: constants in `autoload.php` → autoload `Nesh\*` from `nesh/src` and `nesh/installer` → `Request::start()` (errors off, security headers, CORS, CSRF cookie) → four `App` instances → `App::run()`. Schemas are applied by the graphic installer, not on every HTTP request.

**Pages.** `/{page}` on an app vhost → `pages/{page}.html`. Missing files return HTML 404. Nesh injects charset, viewport, favicon, canonical URL, `og:image`, page CSS/JS, and suffixes the title with the app name.

Also: `AiClient` / `AiConfig` (OpenAI-compatible chat), GD / FFmpeg / Dompdf helpers, shared `nesh/scripts` in `nesh/scripts/nesh.js`.

---

## API REST

```
METHOD /api/{service}/{method?}/{id?}
```

Examples: `GET /api/posts`, `POST /api/sessions`, `GET /api/user/2`.

1. `{service}` → `{app}/api/{service}.php` (plural may fall back to singular).
2. Class `{Folder}\{Service}` (e.g. `Dyscover\Posts`).
3. Segment 2 is the method (`index` if missing, numeric, or a UUID).
4. Non-null return → JSON `{ success, … }`. Errors: `{ "success": false, "message": "…" }`.

Unless listed on `$app->api->publicApi` (set in each `index.php`):

- **GET** needs a session.
- **POST / PUT / PATCH / DELETE** need session **and** CSRF.

| App | `publicApi` |
| --- | --- |
| Www | `views/track`, `stats` |
| Account | `oauth/google`, `recovery`, `recovery/reset`, `availability` |
| Dyscover | *(empty)* |
| Admin | `news`, `careers`, `careers/apply`, `team`, `apps` |

**Dyscover (selected):** `posts`, `feed`, `explore`, `articles`, `article-content`, `article-generate`, `inbox`, `users`, `tags`, `templates`, `creator-center`, `activity`, `reports`, `moderation`. Article placement rules live in the graphic editor (`place.js`), not in SQL.

---

## Repository layout

```
ielectro/
├── nesh/          framework (PHP, installer, icons, vendor)
├── account/       identity app
├── dyscover/      social / articles app
├── admin/         staff app
├── www/           public site
└── index.php      bootstrap manager (login + per-app init)
```

Every application folder:

| Path | Role |
| --- | --- |
| `index.php` | boot Nesh, set `publicApi`, `run()` |
| `pages/` | `{name}.html` templates |
| `scripts/{name}/index.js` | page module |
| `styles/{name}/index.css` | page stylesheet |
| `api/` | PHP classes in the app namespace |
| `database/schema/` | numbered `.sql` bootstrap files |
| `database/backup/` | daily SQL dumps (not served, not committed) |
| `assets/` | brand files and user uploads |
| `version.env` | app version only (developer-set; used for asset `?v=`) |
| `init.boot` | initialization marker (gitignored; created by the installer) |

Nesh: `nesh/src/`, `nesh/installer/`, `nesh/scripts/`, `nesh/styles/`, `nesh/icon/`, `nesh/vendor/` (Dompdf, MaxMind DB). Browsers load ES modules; there is no npm build.