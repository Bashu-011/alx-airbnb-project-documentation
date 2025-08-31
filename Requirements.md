# Airbnb Clone – Backend Requirements Specification

## Scope
This document specifies **technical and functional requirements** for three core backend features:
1. **User Authentication & Authorization (JWT + RBAC)**
2. **Property Management (CRUD, media, availability)**
3. **Booking System (availability, reservations, payments integration)**

It covers **API endpoints**, **request/response schemas**, **validation rules**, **error handling**, **security**, and **performance criteria**. Assumes a RESTful API with JSON over HTTPS, backing store in PostgreSQL/MySQL, and object storage for images.

---

## 1) User Authentication & Authorization

### 1.1 Functional Goals
- Allow users to **register** as Guest or Host and **login** via email/password (and optional OAuth).
- Maintain **JWT-based sessions** and **refresh tokens**.
- Enforce **RBAC** for actions (Guest, Host, Admin).
- Support **profile management** (name, phone, photo).

### 1.2 Endpoints

#### POST `/auth/register`
Registers a new user.

**Request (JSON)**
```json
{
  "firstName": "Jane",
  "lastName": "Doe",
  "email": "jane@example.com",
  "password": "S3cure!pass",
  "role": "guest"
}
```

**Validation**
- `email`: valid, unique, max 254 chars.
- `password`: min 8 chars, at least one letter & number; hashed with Argon2/bcrypt.
- `role`: enum `guest|host` (admin created via admin interface only).

**Responses**
- `201 Created`
```json
{
  "userId": "uuid",
  "email": "jane@example.com",
  "role": "guest"
}
```
- `409 Conflict` if `email` exists.
- `400 Bad Request` for schema errors.

---

#### POST `/auth/login`
Authenticates user and issues tokens.

**Request**
```json
{
  "email": "jane@example.com",
  "password": "S3cure!pass"
}
```

**Responses**
- `200 OK`
```json
{
  "accessToken": "jwt",
  "refreshToken": "jwt",
  "expiresIn": 3600,
  "user": { "userId": "uuid", "role": "guest" }
}
```
- `401 Unauthorized` for invalid credentials.

**Security**
- Brute-force protection: rate limit (e.g., 5/min/IP + user), incremental backoff, reCAPTCHA after threshold.

---

#### POST `/auth/refresh`
Exchanges a valid refresh token for a new access token.

**Request**
```json
{ "refreshToken": "jwt" }
```

**Response**
- `200 OK`
```json
{ "accessToken": "jwt", "expiresIn": 3600 }
```

**Errors**
- `401 Unauthorized` for invalid/expired/blacklisted token.

---

#### GET `/users/me` (Auth required)
Returns profile of current user.

**Response**
```json
{
  "userId": "uuid",
  "firstName": "Jane",
  "lastName": "Doe",
  "email": "jane@example.com",
  "role": "guest",
  "phone": "+2547...",
  "photoUrl": "https://cdn/..."
}
```

---

#### PATCH `/users/me` (Auth required)
Update profile fields.

**Request (any subset)**
```json
{ "firstName": "Jane", "phone": "+2547..." }
```

**Validation**
- Names: 1–100 chars, unicode letters/spaces.
- Phone: E.164 formatted.
- Photo upload: handled by pre-signed URL flow (see Property Media).

**Errors**
- `400` invalid fields; `413` too large photo; `415` unsupported media type.

### 1.3 Authorization (RBAC)
- **Guest**: browse/search, create bookings, write reviews for completed stays, manage own profile.
- **Host**: all Guest permissions + manage own properties and view bookings for their listings.
- **Admin**: moderation, payments reconciliation, user and listing admin actions.

Enforce via JWT claims: `{ sub, role, iat, exp }` and resource ownership checks.

### 1.4 Data Model (simplified)
- `users(id uuid pk, email unique, password_hash, role enum, first_name, last_name, phone, photo_url, created_at, updated_at)`
- Index on `(email)`, `(role)`.

### 1.5 Security & Compliance
- Hash passwords with **Argon2id** (preferred) or **bcrypt(12+)**.
- JWT signed with **RS256**; access exp ~ **15 min**, refresh exp ~ **7–30 days**.
- **HTTPS-only**, **HSTS**, **SameSite=Lax** for cookies if using cookie-based tokens.
- **Audit logs** for security-sensitive actions (login, role change).

### 1.6 Performance Criteria
- P99 latency: **< 200 ms** for `/auth/login` when cache warm (DB hit OK).
- Availability: **99.9%**.
- Rate limits: **60 req/min/IP** for auth endpoints.
- Horizontal scalability behind a load balancer; stateless services.

---

## 2) Property Management

### 2.1 Functional Goals
- Hosts can **create/edit/delete** property listings.
- Attach **images** (cloud storage), manage **amenities**, **pricing**, **availability**.
- Guests can **search/filter** with pagination and sort.

### 2.2 Endpoints

#### POST `/properties` (Host)
Create a listing.

**Request**
```json
{
  "title": "Cozy Cottage",
  "description": "Near the river...",
  "location": { "country": "KE", "city": "Kilungu", "lat": -1.8, "lng": 37.2 },
  "pricePerNight": 45.00,
  "currency": "USD",
  "maxGuests": 4,
  "amenities": ["wifi","parking","kitchen"],
  "availability": { "checkInAfter": "14:00", "checkOutBefore": "10:00" }
}
```

**Validation**
- `title`: 3–140 chars; `description`: 20–5,000 chars.
- `pricePerNight`: > 0; `currency`: ISO 4217.
- `maxGuests`: 1–30.
- `location`: required; lat -90..90, lng -180..180.

**Responses**
- `201 Created`
```json
{ "propertyId": "uuid" }
```

---

#### GET `/properties/{id}`
Retrieve listing details.

**Response**
```json
{
  "propertyId": "uuid",
  "hostId": "uuid",
  "title": "Cozy Cottage",
  "description": "Near the river...",
  "location": { "country": "KE", "city": "Kilungu", "lat": -1.8, "lng": 37.2 },
  "pricePerNight": 45.0, "currency": "USD",
  "maxGuests": 4,
  "amenities": ["wifi","parking","kitchen"],
  "photos": ["https://cdn/..1.jpg","https://cdn/..2.jpg"],
  "rating": 4.7,
  "reviewCount": 12,
  "createdAt": "2025-08-20T08:00:00Z"
}
```

**Errors**: `404` not found, `410` if removed.

---

#### PATCH `/properties/{id}` (Host – owner only)
Update editable fields (same validation as create).

**Errors**: `403` forbidden if not owner; `409` conflict if attempting to delete when active bookings exist within a protected window.

---

#### DELETE `/properties/{id}` (Host – owner only / Admin)
Soft delete listing; must check future bookings.

**Response**: `204 No Content`

---

#### POST `/properties/{id}/photos/presign` (Host)
Get pre-signed upload URL(s) for images.

**Request**
```json
{ "files": [{ "filename": "front.jpg", "contentType": "image/jpeg" }] }
```
**Response**
```json
{
  "uploads": [{
    "uploadUrl": "https://s3...",
    "publicUrl": "https://cdn.../front.jpg"
  }]
}
```

**Validation**: max 15 photos/property; max 10 MB/photo; allow `image/jpeg`, `image/png`, `image/webp`.

---

#### GET `/properties` (Public search)
Query with filters and pagination.

**Query Params**
- `q` (text), `city`, `country`, `minPrice`, `maxPrice`, `guests`, `amenities` (comma list), `sort` (price_asc|price_desc|rating_desc|new), `page`, `pageSize` (<= 50).

**Response**
```json
{
  "items": [{ "propertyId": "uuid", "title": "Cottage", "pricePerNight": 45.0, "currency": "USD", "rating": 4.7, "thumbnail": "..." }],
  "page": 1, "pageSize": 20, "total": 134
}
```

**Performance**
- Use DB indexes on `(country, city)`, `(price_per_night)`, GIN index for text search/amenities.
- Cache hot queries in Redis (TTL 60–300s).
- P95 < 300 ms for searches; support pagination up to 10k items via `page`/`pageSize` or cursor-based paging.

### 2.3 Data Model (simplified)
- `properties(id pk, host_id fk->users, title, description, country, city, lat, lng, price_per_night, currency, max_guests, amenities jsonb, is_active, created_at, updated_at)`
- `property_photos(id pk, property_id fk, url, sort_order)`
- Indexes: `(host_id)`, `(country, city)`, `(price_per_night)`, `gin(amenities)`.

### 2.4 Non-Functional
- Rate limit: `GET /properties` **120 req/min/IP**, `POST/PATCH/DELETE` **30 req/min/user**.
- Image scanning for malware/NSFW as needed.
- Soft delete vs. hard delete policy; background reindexing of search cache.

---

## 3) Booking System

### 3.1 Functional Goals
- Guests can **create** bookings for specific dates.
- Prevent **double bookings** with transactional checks.
- Support **cancellation** windows and **status lifecycle**: `pending_payment → confirmed → canceled → completed`.
- Integrate with **Payment Gateway** (Stripe/PayPal) via Payment Intents + Webhooks.

### 3.2 Endpoints

#### POST `/bookings` (Guest)
Create a new booking intent and hold inventory.

**Request**
```json
{
  "propertyId": "uuid",
  "startDate": "2025-09-10",
  "endDate": "2025-09-15",
  "guests": 2,
  "currency": "USD",
  "idempotencyKey": "b1e9f..."
}
```

**Validation**
- `startDate < endDate`, both future dates.
- Stay length 1–30 nights.
- `guests` <= property.maxGuests.
- `idempotencyKey`: required; 1–64 chars.

**Processing**
- Verify property active.
- Check availability with **`SELECT ... FOR UPDATE`** on date range (or exclusion constraint).
- Create **provisional booking** `status=pending_payment`.
- Create **payment intent** with gateway; return client secret.

**Response**
- `201 Created`
```json
{
  "bookingId": "uuid",
  "status": "pending_payment",
  "clientSecret": "pi_..._secret",
  "totalAmount": 225.00,
  "currency": "USD"
}
```

**Errors**
- `404` property not found/inactive.
- `409` dates unavailable.
- `422` invalid payload (date rules, guests).

---

#### GET `/bookings/{id}` (Owner guest / Host of property / Admin)
Fetch booking.

**Response**
```json
{
  "bookingId": "uuid",
  "propertyId": "uuid",
  "guestId": "uuid",
  "status": "confirmed",
  "startDate": "2025-09-10",
  "endDate": "2025-09-15",
  "nights": 5,
  "priceBreakdown": {
    "nightly": 40.0,
    "nights": 5,
    "fees": { "cleaning": 25.0 },
    "discounts": 0.0,
    "total": 225.0
  },
  "createdAt": "2025-08-31T04:00:00Z"
}
```

---

#### POST `/bookings/{id}/cancel` (Guest/Host per policy)
Cancel booking subject to policy/time window. Refunds initiated if eligible.

**Response**
```json
{ "bookingId": "uuid", "status": "canceled" }
```

**Errors**
- `403` not allowed (outside cancellation window).
- `409` already completed or canceled.

---

#### POST `/payments/webhooks` (Public – verify signature)
Handle payment events.

**Events**
- `payment_succeeded`: mark booking `confirmed`, schedule host payout (minus fees).
- `payment_failed|canceled`: mark booking `canceled`, release held dates, notify parties.

**Security**
- Verify gateway signature; idempotent handling by `eventId` table.

### 3.3 Data Model (simplified)
- `bookings(id pk, property_id fk, guest_id fk, start_date, end_date, status enum, total_amount, currency, created_at, updated_at)`
- Unique overlap constraint per property, e.g.:
  - Postgres: `EXCLUDE USING GIST (property_id WITH =, daterange(start_date, end_date, '[]') WITH &&)`
- `payments(id pk, booking_id fk, provider, intent_id, status, amount, currency, events jsonb)`

### 3.4 Business Rules
- **Check-in/out**: nights = `endDate - startDate`.
- **Hold duration**: provisional booking expires (e.g., 15 min) if no payment success.
- **Refunds**: use provider API; asynchronous via webhook + background jobs.
- **Notifications**: email + in-app for confirmations/cancellations.

### 3.5 Performance & Reliability
- P95 latency: **< 250 ms** for booking creation (excluding payment redirect).
- Strong transactional integrity on availability check + create.
- **Idempotency**: dedupe on `(idempotencyKey, userId)` for POST `/bookings`.
- Background queues for notifications/payout scheduling.
- Observability: tracing, structured logs, alerts on error-rate spikes.


## 4) Non-Functional Summary
- **Security**: OWASP ASVS L2, secret rotation, env var management, least privilege IAM.
- **Scalability**: stateless services, horizontal autoscaling, read replicas for DB-heavy reads.
- **Performance**: caching for hot endpoints, tuned DB indexes, async jobs for non-critical work.
- **Availability**: 99.9% target, multi-AZ infrastructure, health checks, graceful shutdowns.

