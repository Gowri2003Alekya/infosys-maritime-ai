# FreightQuote AI — Milestone 2: Enterprise Authentication, RBAC & Operational Intelligence

**Infosys Springboard Internship 7.0 · Batch 4**

*Track: Full-Stack Artificial Intelligence & Enterprise Cloud Operations*

*Technology Stack: FastAPI · SQLite · JWT · bcrypt · Role-Based Access Control (RBAC) · Google Colab · ngrok*

## 1. Executive Summary & Architecture Overview

**FreightQuote AI** is an autonomous logistics and freight-management operations platform. Milestone 2 transitions the system from an open operational prototype into an enterprise-grade, secure, multi-tenant application equipped with full identity access management (IAM), strict role-based access control (RBAC), administrative audit logging, and an augmented SQL-backed conversational AI Copilot.

### System Architecture

The application is structured into a modular, asynchronous architecture operating inside a unified runtime:

```
                                  +---------------------------------------+
                                  |         Client Browser / UI           |
                                  |   (Vanilla ES6+, HTML5, Custom CSS)   |
                                  +---------------------------------------+
                                                     |
                                   HTTPS / WSS (via ngrok Tunnel)
                                                     |
                                                     v
+---------------------------------------------------------------------------------------------------+
|                                  FastAPI Unified Server (server.py)                              |
|                                                                                                   |
|  +--------------------+  +----------------------+  +---------------------+  +------------------+  |
|  | Authentication API |  | Role-Based Access    |  | Admin Management    |  | Static Assets    |  |
|  | (/api/auth/*)      |  | Control (/api/page/*)|  | Dashboard (/admin)  |  | (HTML/CSS/JS)    |  |
|  +--------------------+  +----------------------+  +---------------------+  +------------------+  |
|            |                         |                        |                      |            |
|            v                         v                        v                      v            |
|  +--------------------+  +----------------------+  +---------------------+  +------------------+  |
|  | Password Hashing   |  | Permission Verifier  |  | Audit Trail Logger  |  | AI Copilot Engine|  |
|  | (bcrypt + JWT)     |  | need(ctx, capability)|  | (activity_log)      |  | (analyst.py)     |  |
|  +--------------------+  +----------------------+  +---------------------+  +------------------+  |
+---------------------------------------------------------------------------------------------------+
                                                     |
                                  Thread-Safe SQLite Engine (db.py)
                                                     |
                                                     v
+---------------------------------------------------------------------------------------------------+
|                                      SQLite Database Layer (freight.db)                           |
|                                                                                                   |
|  [users]           [auth_lockouts]     [auth_otps]        [activity_log]      [quotes]            |
|  [shipments]       [carriers]          [weather_events]   [customs_rules]     [route_segments]    |
+---------------------------------------------------------------------------------------------------+
                                                     ^
                                                     | Background Thread
+----------------------------------------------------+----------------------------------------------+
|                              Autonomous Agent Background Worker                                   |
|   (Route AI · Spot Quotes · Carrier Risk · Weather Monitoring · Margin Engine · Customs Audit)    |
+---------------------------------------------------------------------------------------------------+





```

### Core Technology Stack

* **Backend Framework**: FastAPI (Asynchronous Python 3.10+) running with Uvicorn ASGI server.

* **Database**: SQLite 3 with WAL (Write-Ahead Logging) and thread-safe connection pool (`db.py`).

* **Security & Cryptography**: `bcrypt` password hashing (salt rounds: 12), `PyJWT` with HS256 algorithm.

* **Frontend**: Lightweight, dependency-free vanilla JavaScript (ES6 modules), responsive CSS Grid/Flexbox design, dynamic DOM hydration.

* **Tunneling & Deployment**: `pyngrok` for secure ingress tunneling, compatible with Google Colab and local environments.

* **AI Analytics**: Deterministic SQL intent classification engine (`analyst.py`, `qa.py`) with zero-hallucination guarantees.

## 2. Setup & Execution Instructions

### Prerequisites

* Python 3.10 or higher (or a Google Colab GPU/Standard runtime).

* An ngrok authentication token (obtainable free from [ngrok.com](https://ngrok.com)).

* *(Optional)* A Gmail account with an App Password if testing SMTP email OTP delivery.

### Configuration via Google Colab Secrets

To maintain zero-trust security and prevent credentials from leaking into Git repositories or notebook cells, configure the following keys in Google Colab's **Secrets (🔑)** panel:

| Secret Key | Description | Required? | Default / Fallback | 
| ----- | ----- | ----- | ----- | 
| `NGROK_AUTHTOKEN` | Your private ngrok ingress tunnel token | **Yes** | Prompts in Colab if omitted | 
| `JWT_SECRET` | Secret key for signing JWT bearer tokens | Recommended | Auto-generates cryptographically secure key | 
| `GMAIL_USER` | Gmail address for sending 6-digit password reset OTPs | Optional | Falls back to in-memory console OTP logging | 
| `GMAIL_APP_PASSWORD` | 16-character Google App Password (not account password) | Optional | Console OTP fallback active if omitted | 

### Step-by-Step Google Colab Execution Guide

1. Open `FreightQuote_Milestone2.ipynb` in Google Colab.

2. Open the **Secrets** tab on the left sidebar (key icon) and add `NGROK_AUTHTOKEN`.

3. Choose **Runtime → Run all** (or press `Ctrl+F9`).

4. Cell execution flow:

   * **Cell 1**: Installs required dependencies (`fastapi`, `uvicorn`, `pyjwt`, `bcrypt`, `pyngrok`, `python-multipart`).

   * **Cell 2**: Initializes database schema (`db.py`), pre-populates carriers, routes, customs rules, and seeds the root `admin` user.

   * **Cell 3**: Defines core security modules (`auth.py`, `rbac.py`, `hardening.py`).

   * **Cell 4**: Configures operational domain models (`domain_freight.py`, `analyst.py`, `qa.py`, `views_freight.py`).

   * **Cell 5**: Constructs static assets (`login.html`, `admin.html`, `profile.html`, `app.js`, `style.css`).

   * **Cell 6**: Launches the FastAPI server on port 8000 and initiates the background autonomous agent cycle.

   * **Cell 7**: Establishes the ngrok tunnel and outputs the live public URL.

5. Click the generated public ngrok URL to open the platform.

## 3. Authentication & Security Specifications (Milestone 1 Integration)

The authentication architecture incorporates all defensive requirements from Milestone 1, hardened with enterprise lockout controls and multi-channel account recovery.

### Credential Storage & Token Management

* **Password Hashing**: Stored passwords are cryptographically salted and hashed using `bcrypt` (12 work factor rounds). Raw passwords and hashes are never returned across API payloads.

* **Session Tokens**: Upon authentication, the server generates a JSON Web Token (JWT) signed with `HS256`, encoding user ID, email, role, and expiration timestamp (24-hour lifetime).

* **Transport Security**: The JWT is delivered in both an `Authorization: Bearer <token>` header and an `httpOnly`, `SameSite=Lax` cookie (`freightquote_session`), defending against Cross-Site Scripting (XSS) and token exfiltration.

### Input Validation Rules

All signup and profile modification endpoints enforce strict regular expression checks:

* **Email Validation**: Must strictly follow RFC structure requiring at least 2 alphanumeric characters before the `@`, at least 2 characters in the domain name, and at least 2 characters in the top-level domain (`^[a-zA-Z0-9_.+-]{2,}@[a-zA-Z0-9-]{2,}\.[a-zA-Z0-9-.]{2,}$`).

* **Username Validation**: Minimum 3 characters, alphanumeric and underscore only (`^[a-zA-Z0-9_]{3,30}$`).

* **Password Strength**: Minimum 8 characters, containing at least one uppercase letter, one lowercase letter, one digit, and one special symbol (`[!@#$%^&*(),.?":{}|<>]`).

### Enterprise Lockout Policy & Security Hardening

To prevent brute-force credential stuffing and password exhaustion attacks, `auth.py` and `hardening.py` enforce a dual-tier lockout mechanism tracked in the `auth_lockouts` table:

```
[Failed Login Attempt]
         |
         v
+-------------------------------------------------------+
| 3 Consecutive Failed Attempts within 15-minute window |
+-------------------------------------------------------+
         |
         v
[Tier 1: Temporary Lockout] --> Account disabled for 300 seconds (5 minutes).
         |
         +--> Reset on successful login before 3 failures.
         |
         v
+-------------------------------------------------------+
| 5 Cumulative Temporary Lockouts                      |
+-------------------------------------------------------+
         |
         v
[Tier 2: Permanent Lockout] --> Account permanently disabled (locked = 1).
                                Requires an Administrator to unlock via Admin Dashboard.





```

* **Unified Error Messages**: To prevent username harvesting and timing attacks, the server returns the identical error response: `Invalid email or password.` regardless of whether the email exists or the password was incorrect.

* **Immediate Token Revocation**: When an administrator locks a user or changes their role, all existing JWT tokens are immediately invalidated because `user_of()` re-validates the user's `locked` status and role in SQLite on every incoming request.

### Dual-Path Account Recovery Flow

Users who lose their credentials can recover their accounts through two independent mechanisms:

1. **Security Question & Answer**:

   * Configurable during initial signup or within the Profile page.

   * The security answer is case-folded, stripped of whitespace, and hashed with `bcrypt`, guaranteeing that answers are not visible even to database administrators.

2. **One-Time Password (OTP) via Email**:

   * Generates a cryptographically random 6-digit numeric OTP with a 5-minute time-to-live (TTL).

   * Rate-limited to 1 request per 60 seconds per account, with a maximum of 5 verification attempts.

   * Delivered via TLS-encrypted SMTP (Gmail App Password) or logged to the secure server console in local offline environments.

## 4. User Profile Management

Every authenticated user has access to a dedicated **Profile Page** (`/profile`):

* **Account Overview**: Displays account metadata including Username, Registered Email, System Role, Account Created Timestamp, and Last Active Login Timestamp.

* **Username Modification**: Enables self-service updating of the display username, verified against uniqueness constraints.

* **Password Rotation**: Requires the user to enter their current password, validated against `bcrypt`, before accepting a new password adhering to the complexity rules.

* **Security Question Management**: Allows users to select from standard security questions (e.g., *"What was the name of your first logistics manager?"*, *"What was your childhood street name?"*) and set a cryptographically hashed answer.

* **Privacy Assurance**: Passwords and cryptographic hashes are strictly scrubbed from the profile serialization model.

## 5. Role-Based Access Control (RBAC) Architecture

FreightQuote AI implements a 5-role enterprise authorization model. Permissions are enforced **server-side** at the API gateway layer (`server.py`, `rbac.py`), guaranteeing that UI tampering or direct URL navigation cannot bypass security boundaries.

### Five Distinct System Roles

1. **Admin (`Admin`)**: Platform owner with unrestricted access across operational pipelines, carrier financials, system configuration, user provisioning, role assignments, and security audit logs.

2. **Operations Manager (`Operations Manager`)**: Logistics coordinator managing route planning, dispatch optimization, carrier performance evaluation, customs clearance, and weather-driven reroutes. Restricted from commercial margins and user administration.

3. **Freight Broker (`Freight Broker`)**: Commercial sales agent authorized to generate customer spot quotes, view margin calculations, inspect freight tiers, and manage customer shipments. Prohibited from modifying customs rules or system administration.

4. **Auditor (`Auditor`)**: Compliance and governance inspector with read-only visibility into operational logs, customs holds, carrier safety metrics, and quote histories. Barred from dispatching quotes or mutating users.

5. **Customer (`Customer`)**: External shipper scoped exclusively to their own account. Can request new spot quotes, track their assigned shipments, and interact with the AI Copilot for their specific cargo. Completely isolated from internal carrier metrics, margins, and operational agents.

### Complete Page Access & Capability Matrix

| Page ID | Display Page Name | Admin | Operations Manager | Freight Broker | Auditor | Customer | Required Capability | 
| ----- | ----- | ----- | ----- | ----- | ----- | ----- | ----- | 
| `copilot` | AI Copilot Chat | Allowed | Allowed | Allowed | Allowed | Allowed | `copilot` (Scoped) | 
| `route` | Agent 1 · Route AI | Allowed | Allowed | Denied | Allowed (Read) | Denied | `routes` | 
| `quotes` | Agent 2 · Spot Quotes | Allowed | Allowed | Allowed | Allowed (Read) | Allowed (Scoped) | `quotes` | 
| `carriers` | Agent 3 · Carriers Risk | Allowed | Allowed | Denied | Allowed (Read) | Denied | `carriers` | 
| `weather` | Agent 4 · Weather Risk | Allowed | Allowed | Denied | Allowed (Read) | Denied | `weather` | 
| `margin` | Agent 5 · Margin Tiers | Allowed | Denied | Allowed | Allowed (Read) | Denied | `margin` | 
| `customs` | Agent 6 · Customs Rules | Allowed | Allowed | Denied | Allowed (Read) | Denied | `customs` | 
| `profile` | User Profile Settings | Allowed | Allowed | Allowed | Allowed | Allowed | `profile` | 
| `admin` | Admin User Management | Allowed | Denied | Denied | Denied | Denied | `users` | 

### Server-Side Enforcement Mechanisms

* **Menu Rendering (`menu_for(user)`)**: Dynamically prunes navigation menus based on `ROLE_MENU[role]`.

* **API Guard (`need(ctx, capability)`)**: Evaluates `ROLE_CAPS[role]`. If the capability is absent, raises an immediate `HTTP 403 Forbidden` with payload `{"detail": "Forbidden: role '<role>' lacks capability '<capability>'"}`.

* **Direct Route Protection**: Attempting to invoke `GET /api/page/{page_id}` or `GET /api/detail/{entity_id}` directly bypasses client UI checks and triggers server-side validation against `ROLE_MENU`.

* **Tenant Data Isolation (Customer Scoping)**:

  * Users with the `Customer` role have a non-null `customer_id` assigned in `users`.

  * Database queries across quote history and shipment tracking automatically append `WHERE customer_id = :customer_id`, preventing cross-customer data leakage.
