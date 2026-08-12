## Project: Shadow Economy Map — API Specification
**Feature:** Admin Access
**Conventions:** see 0.1 in `00-api-conventions.md` — base path `/api/v1`, standard envelope, ApiError/validate middleware.

---

### 4.1 Endpoint Table

| Method | Path | Auth | Linked Use Case | Linked FR |
|---|---|---|---|---|
| POST | /admin/auth/login | Public | UC-11 | FR-15, FR-16 |
| POST | /admin/auth/logout | Admin | UC-11 | FR-17 |
| GET | /admin/auth/me | Admin | UC-11 | FR-15 |

Note: `login` itself is `Public` auth (you can't require a token to obtain a token) — everything else in this feature and every other Admin-scoped feature requires the Bearer token this endpoint issues.

---

### 4.2 Endpoint Detail

#### POST /admin/auth/login

**Purpose:** Authenticate an admin and issue a JWT session token (UC-11).

**Auth:** Public

**Request body:**
```json
{
  "email": "string, required, valid email",
  "password": "string, required, min 8 chars"
}
```

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "Login successful",
  "data": {
    "token": "string (JWT)",
    "admin": {
      "id": "uuid",
      "email": "admin@example.com"
    }
  }
}
```

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 400 | Zod validation failure | field-specific |
| 401 | Email not found, or password incorrect | "Invalid email or password" |

Per FR-16, the 401 message is intentionally identical for "email not found" and "wrong password" — it never reveals which field was incorrect.

**Implemented in:** `src/controllers/adminAuth.controller.ts → login` · `src/services/adminAuth.service.ts → authenticateAdmin` · `src/schemas/adminAuth.schema.ts → loginSchema`

---

#### POST /admin/auth/logout

**Purpose:** Invalidate the current admin session.

**Auth:** Admin

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "Logged out",
  "data": {}
}
```
Since auth is stateless JWT (per the backend template's pattern — no session table), "logout" is enforced client-side (discarding the token) plus, if implemented, a short-lived server-side denylist entry for the token's remaining validity window. This detail should be confirmed with whoever implements auth — flagged here rather than assumed.

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 401 | Missing/invalid/expired token | "Invalid or expired token" |

**Implemented in:** `src/controllers/adminAuth.controller.ts → logout` · `src/services/adminAuth.service.ts → invalidateSession`

---

#### GET /admin/auth/me

**Purpose:** Return the currently authenticated admin's own profile — used by the admin frontend to confirm session validity on load and display who's logged in.

**Auth:** Admin

**Success response — 200:**
```json
{
  "statusCode": 200,
  "success": true,
  "message": "OK",
  "data": {
    "id": "uuid",
    "email": "admin@example.com",
    "createdAt": "2026-01-10T00:00:00Z"
  }
}
```

**Error responses:**
| Status | Condition | Message |
|---|---|---|
| 401 | Missing/invalid/expired token (per FR-17's inactivity expiry) | "Invalid or expired token" |

**Implemented in:** `src/controllers/adminAuth.controller.ts → getMe`

---

**Next:** proceed to → [05. Data Pipeline Management API]