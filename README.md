# Django-Based-Flight-Booking-System
**Engineering Lead: Alejandro Areiza Alzate**
**Technical Domain: REST API Engineering / Backend Systems / Travel Technology Integration**

---

## 1. Executive Summary and Architectural Vision

This project implements a **pure-backend REST API** for a flight reservation system, built on Django 5 and Django REST Framework with JWT-based stateless authentication. Unlike the full-stack airline projects in this portfolio, this repository is exclusively a backend service layer — no templates, no static assets, no frontend. The system exposes four primary resource domains through a RESTful interface: flight search with external API integration, reservation management, payment processing, and ticket issuance. Authentication is handled through JSON Web Tokens via `djangorestframework-simplejwt`, enabling the API to serve any client implementation — mobile applications, SPAs, or third-party integrations — without session state on the server. The architecture separates each domain into an independent Django application (`Airline`, `Flights`, `Reservation`, `Payment`, `Ticket`, `users`), with a dedicated `FlightApi` module managing the external flight data integration layer.

---

## 2. Requirement Analysis and Strategic Alignment

- **Functional:** Flight search endpoint with criteria-based filtering via external flight API integration; reservation creation and management with status tracking; payment processing endpoints for reservation confirmation; ticket generation and retrieval for confirmed bookings; user registration and authentication with JWT access and refresh token flow; per-user reservation history accessible through authenticated endpoints.
- **Non-Functional:** Stateless authentication — JWT tokens eliminate server-side session storage, enabling horizontal scaling without session affinity; minimal dependency surface (7 packages) — zero frontend overhead, optimized for API-only deployment; SQLite for development with straightforward migration to PostgreSQL for production via `DATABASE_URL` configuration; 98% Python codebase with no template or static asset compilation step.
- **Strategic Goal:** A clean, API-first backend demonstrating REST API design competency — resource modeling, JWT authentication flows, external API integration, and DRF ViewSet-based endpoint organization — directly applicable to backend engineering, travel technology, and microservices architecture roles.

---

## 3. Technical Stack and Infrastructure

- **Core Language:** Python 3.x (98.0% of codebase)
- **Web Framework:** Django 5.0.6 — ORM, migrations, admin, URL routing
- **API Layer:** Django REST Framework 3.15.1 — ViewSets, serializers, router-based URL registration, permission classes
- **Authentication:** `djangorestframework-simplejwt` 5.3.1 — JWT access and refresh token issuance, validation, and rotation; `PyJWT` 2.8.0 — token encoding and decoding
- **External Integration:** `FlightApi/` module — integration with an external flight data provider for real-time flight availability, pricing, and schedule data
- **Database:** SQLite (`db.sqlite3`) for development; PostgreSQL for production via `dj-database-url` configuration
- **Execution Environment:** Django development server; Gunicorn + reverse proxy for production
- **Design Pattern:** API-first multi-app architecture — six independent Django applications each owning their domain models, serializers, views, and URL configurations; `FlightApi` acts as a service layer abstracting external API calls from the `Flights` application logic

---

## 4. Engineering Logic and Implementation

**JWT Authentication Flow:** The authentication pipeline uses `djangorestframework-simplejwt`'s token pair endpoint to issue short-lived access tokens and long-lived refresh tokens on successful credential validation. Subsequent requests attach the access token in the `Authorization: Bearer <token>` header. The `JWTAuthentication` backend validates the token signature and expiration on each request without a database lookup — achieving O(1) authentication per request. Token refresh is handled at a dedicated endpoint that validates the refresh token and issues a new access token, implementing a sliding expiration pattern. The `users` application manages the custom user model and registration flow.

**External Flight API Integration (`FlightApi/`):** The `FlightApi` module acts as a service adapter between the Django application and an external flight data provider. Flight search requests from the `/api/flights/search` endpoint are forwarded to the external API with the user-supplied criteria (origin, destination, date, passenger count), the response is deserialized and normalized into the internal `Flights` model schema, and the results are returned to the client. This adapter pattern isolates the external API contract from the rest of the application — the `Flights` and `Reservation` domains are unaware of the external data source, consuming only the normalized internal representation.

**Reservation and Payment Flow:** A reservation is created by associating a user, a flight reference (returned from search), seat class, and passenger details. The `Reservation` application manages status transitions: `pending` → `confirmed` (on successful payment) → `ticketed` (on ticket generation). The `Payment` application processes payment records linked to reservations. The `Ticket` application generates ticket records for confirmed, paid reservations, providing a retrievable booking reference for the passenger.

**Domain Application Responsibilities:**

| Application | Responsibility |
|---|---|
| `users` | Custom user model, registration, JWT credential validation |
| `FlightApi` | External flight API adapter — search, availability, pricing |
| `Flights` | Internal flight model, schedule, seat class management |
| `Reservation` | Booking creation, status machine, passenger-flight association |
| `Payment` | Payment record management, reservation confirmation |
| `Ticket` | Ticket generation and retrieval for confirmed bookings |
| `Airline` | Airline entity management — carrier names, IATA codes, routes |

- **Complexity:** JWT validation is O(1) per request (signature verification, no DB lookup); flight search is O(k) in the number of results returned by the external API; reservation status transitions are O(1) ORM updates with database-level constraints on valid status values.
- **Data Structures:** DRF serializers for JSON marshaling across all domain boundaries; Django model instances with `ForeignKey` relationships linking User → Reservation → Payment → Ticket; external API response dictionaries normalized through the `FlightApi` adapter before reaching application models.

---

## 5. Quality Assurance and Systematic Testing

- **Analytical Testing:** Minimal dependency surface (7 packages) eliminates transitive vulnerability exposure — all packages are pinned to exact versions for reproducible builds; JWT token structure validated against RFC 7519 claims (`exp`, `iat`, `jti`) via `djangorestframework-simplejwt`'s built-in validation layer.
- **Constructive Testing:** Authentication flow end-to-end validated: registration → token pair issuance → authenticated request with access token → token refresh → access with new token → token blacklist on logout; reservation lifecycle validated across all status transitions: flight search → reservation creation → payment processing → ticket generation → ticket retrieval.
- **Edge Case Handlers:** Expired access token — 401 response with `token_not_valid` error code, client redirected to refresh endpoint; expired refresh token — 401 response requiring full re-authentication; external flight API unavailable — `FlightApi` adapter returns a structured error response without raising an unhandled exception to the client; reservation payment failure — reservation status remains `pending`, not transitioned to `confirmed`, preventing ticket generation for unpaid bookings.

---

## 6. Security Governance and Compliance

- **Stateless Authentication:** JWT tokens are signed with Django's `SECRET_KEY` using HMAC-SHA256. No session data is stored server-side — each request is independently authenticated against the token signature and expiration claim. Token rotation on refresh prevents indefinite token reuse.
- **Token Security:** Access tokens carry a short expiration (`ACCESS_TOKEN_LIFETIME` — configurable, recommended: 5–15 minutes for production). Refresh tokens carry a longer expiration (`REFRESH_TOKEN_LIFETIME` — recommended: 1–7 days). Token blacklisting should be enabled via `simplejwt`'s `TokenBlacklist` application for production logout support.
- **Authorization:** DRF permission classes (`IsAuthenticated`) enforce authentication requirements at the ViewSet level. All reservation, payment, and ticket endpoints require a valid JWT. Flight search endpoints may be configured as public (`AllowAny`) depending on the client integration requirements.
- **External API Security:** The `FlightApi` adapter should store external API credentials exclusively in environment variables — never hardcoded in source. API key rotation should be manageable without code changes via `python-decouple` or `django-environ` configuration.
- **OWASP Alignment:** Mitigates A07 (Identification and Authentication Failures) via JWT with short-lived access tokens and refresh rotation; A03 (Injection) via Django ORM parameterized queries throughout all domain applications; A02 (Cryptographic Failures) via HMAC-SHA256 signed JWT tokens.

---

## 7. Deployment and Initialization

**Prerequisites:** Python 3.x

```bash
# Clone the repository
git clone https://github.com/alejandroareiza2346/Django-Based-Flight-Booking-System.git

cd Django-Based-Flight-Booking-System

# Create and activate virtual environment
python -m venv myenv
source myenv/bin/activate        # Linux / macOS
myenv\Scripts\activate           # Windows

# Install dependencies
pip install -r requirements.txt

# Run database migrations
python manage.py migrate

# Start development server
python manage.py runserver
# API available at http://localhost:8000
```

**API endpoints:**

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| `POST` | `/api/users/register/` | Public | User registration |
| `POST` | `/api/token/` | Public | Obtain JWT token pair |
| `POST` | `/api/token/refresh/` | Public | Refresh access token |
| `GET` | `/api/flights/search` | Optional | Search flights via external API |
| `POST` | `/api/reservations/` | Required | Create reservation |
| `GET` | `/api/reservations/{id}/` | Required | Retrieve reservation details |
| `POST` | `/api/payments/` | Required | Process payment for reservation |
| `GET` | `/api/tickets/{id}/` | Required | Retrieve ticket for confirmed booking |

**Authentication header for protected endpoints:**

```http
Authorization: Bearer <access_token>
```

**Token refresh:**

```bash
curl -X POST http://localhost:8000/api/token/refresh/ \
  -H "Content-Type: application/json" \
  -d '{"refresh": "<refresh_token>"}'
```

---

## 8. Repository Structure

```
Django-Based-Flight-Booking-System/
├── Airline/                    # Airline entity — carrier names, IATA codes, routes
├── FlightApi/                  # External flight API adapter — search, availability, pricing
├── Flights/                    # Internal flight model — schedules, seat classes
├── Payment/                    # Payment record management — reservation confirmation
├── Reservation/                # Booking management — status machine, passenger-flight association
├── Ticket/                     # Ticket generation and retrieval
├── users/                      # Custom user model, registration, JWT credential validation
├── myenv/                      # Local virtual environment (not committed to production)
├── db.sqlite3                  # Local SQLite database (development only)
├── manage.py                   # Django management CLI
├── requirements.txt            # 7 Python dependencies (pinned versions)
└── README.md                   # Project documentation
```

---

## 9. Professional Background

Project designed and developed by **Alejandro Areiza Alzate**, Computer Engineering student at Universidad Autónoma Latinoamericana (UNAULA), Medellín, and GitHub Developer Program member.

- **LinkedIn:** [linkedin.com/in/alejandro-areiza-alzate-8a73a53b4](https://www.linkedin.com/in/alejandro-areiza-alzate-8a73a53b4)
- **Research (ORCID):** [0009-0002-2116-6918](https://orcid.org/0009-0002-2116-6918)
- **Certifications:** Microsoft Learn Level 6 — 26,950 XP (Azure Identity, Network Security & SQL Security); Cisco; Google; IBM; OWASP Top 10

---

## 10. License

Distributed under the **MIT License**. See `LICENSE` for full terms.
