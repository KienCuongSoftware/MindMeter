# MindMeter

[![React](https://img.shields.io/badge/React-18.2.0-blue.svg)](https://reactjs.org/)
[![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.5.0-green.svg)](https://spring.io/projects/spring-boot)
[![Java](https://img.shields.io/badge/Java-17-orange.svg)](https://www.oracle.com/java/)
[![MySQL](https://img.shields.io/badge/MySQL-8.0-blue.svg)](https://www.mysql.com/)

MindMeter is a full-stack mental health platform. This README is intentionally backend-focused to show system design decisions, trade-offs, and implementation details.

## Architecture

- **Layered architecture**: `Controller -> Service -> Repository -> MySQL`.
- **API style**: RESTful endpoints with Spring MVC and DTO-based response models.
- **Security**: JWT authentication integrated into Spring Security filter chain.
- **Role model**: RBAC with `ADMIN`, `EXPERT`, `STUDENT`, and `ANONYMOUS`.
- **Data access**: Spring Data JPA/Hibernate with explicit SQL schema in `database/MindMeter.sql`.

```mermaid
flowchart LR
    C[Controller Layer] --> S[Service Layer]
    S --> R[Repository Layer]
    R --> DB[(MySQL)]
    S --> Cache[(In-memory Cache)]
    Client[Frontend / API Client] --> C
```

## Key Technical Decisions

- **JWT-centered API auth**: authentication state is carried in signed tokens and validated by filter-based security middleware.
- **Token refresh endpoints per role context**: system regenerates JWT with latest user/plan data after profile or plan updates.
- **Service-level caching for read-heavy modules**: `@Cacheable` on blog lists/categories/tags to reduce repeated DB reads.
- **Pagination-first design**: `Pageable`/`Page` is used for large collections (blog/forum/admin listing) to control payload size.
- **Anonymous-to-registered upgrade flow**: supports low-friction onboarding, then upgrades to persistent account with secure password hashing.
- **Strict role-based authorization**: endpoint access split by role at security config level to limit privilege scope.

## Trade-offs

- **In-memory cache instead of Redis**
  - **Pros**: simpler setup and lower operational overhead for local/single-node deployment.
  - **Cons**: cache is not shared across instances and has no built-in distributed invalidation.
- **JWT re-issue endpoints without refresh-token rotation store**
  - **Pros**: implementation is straightforward and easy to operate.
  - **Cons**: less control over long-lived session revocation compared with rotating refresh tokens.
- **Layered monolith instead of microservices**
  - **Pros**: faster delivery, easier debugging, and lower deployment complexity.
  - **Cons**: scaling and team ownership boundaries are less flexible at larger system size.

## Authentication Flow

1. Client calls `POST /api/auth/login` with email/password.
2. Backend authenticates via `AuthenticationManager`.
3. Backend issues JWT containing role and user claims.
4. Client sends JWT in `Authorization: Bearer <token>` for protected APIs.
5. JWT is validated by `JwtAuthenticationFilter` before controller access.
6. For role-specific refresh endpoints, backend reissues a new JWT from current user data.

## Security Considerations

- **Password hashing**: `PasswordEncoder` is configured with `BCryptPasswordEncoder`.
- **JWT expiration**: token lifetime is configurable via `jwt.expiration` (example default: 24 hours).
- **Authorization**: endpoint-level RBAC is enforced in Spring Security config.
- **CORS**: allowed origins are explicitly configured from frontend/domain settings (not wildcard).
- **CSRF**: disabled for API-first usage with token-based authentication.

## Database Design

### Core entities

- `users`: identity, role, status, subscription plan, OAuth provider, and audit timestamps.
- `appointments`: links student and expert users, stores schedule, status, consultation type, and notes.
- `depression_test_results`: per-user test outcomes with severity and diagnosis metadata.
- `depression_test_answers`: normalized answers linked to each test result.
- `expert_schedules`: availability windows for experts.

### Key relationships

- One `User` (student) -> many `Appointment`.
- One `User` (expert) -> many `Appointment`.
- One `User` -> many `DepressionTestResult`.
- One `DepressionTestResult` -> many `DepressionTestAnswer`.

```mermaid
erDiagram
    USERS ||--o{ APPOINTMENTS : "student_id"
    USERS ||--o{ APPOINTMENTS : "expert_id"
    USERS ||--o{ DEPRESSION_TEST_RESULTS : "user_id"
    DEPRESSION_TEST_RESULTS ||--o{ DEPRESSION_TEST_ANSWERS : "test_result_id"
```

## Performance Optimization

- **Connection pooling**: HikariCP is configured for stable DB throughput under concurrency.
- **Indexing strategy**: schema includes targeted indexes on scheduling and lookup columns (e.g. `appointment_date`, `student_id`, `expert_id`, `status`).
- **Bounded payloads**: paginated endpoints reduce transfer size and memory pressure in list APIs.

## Caching Strategy

- **Cache layer**: Spring Cache with in-memory `ConcurrentMapCacheManager`.
- **Cached data**:
  - Blog post listing pages.
  - Blog categories and tags.
- **Cache keys**:
  - Blog list: based on `pageNumber_pageSize`.
  - Blog detail/public view: includes post id and caller context (user or anonymous).
  - Categories and tags: static key (`all`).
- **Invalidation**:
  - Blog write operations (create/update/delete) evict `blogPosts` cache entries via `@CacheEvict(allEntries = true)`.
  - Categories/tags are currently read-cached and can be extended with explicit eviction when taxonomy writes are introduced.

## Concurrency Handling

- **Current handling**:
  - Appointment creation validates slot availability before persisting.
  - Appointment state transitions (`PENDING -> CONFIRMED/CANCELLED`) are checked in service-layer transactional methods.
  - Foreign-key constraints enforce referential integrity (`student_id`, `expert_id`).
- **Potential improvements**:
  - Add optimistic locking (`@Version`) on appointment/schedule records.
  - Add stronger database-level uniqueness policy for conflicting time slots.
  - Introduce idempotency keys for booking requests.

## API Example

### Login

`POST /api/auth/login`

Request:

```json
{
  "email": "student1@mindmeter.com",
  "password": "your_password"
}
```

Response (simplified):

```json
{
  "token": "eyJhbGciOiJIUzI1NiIs...",
  "email": "student1@mindmeter.com",
  "role": "STUDENT",
  "user": {
    "id": 1,
    "plan": "FREE"
  }
}
```

### Create Appointment

`POST /api/appointments`

Request:

```json
{
  "expertId": 5,
  "appointmentDate": "2026-04-20T10:00:00",
  "durationMinutes": 60,
  "consultationType": "ONLINE",
  "notes": "Need consultation about stress management"
}
```

Response (simplified):

```json
{
  "id": 128,
  "status": "PENDING",
  "appointmentDate": "2026-04-20T10:00:00",
  "expertId": 5,
  "studentId": 21
}
```

## Future Improvements

- Replace in-memory cache with Redis for shared cache across multiple backend instances.
- Introduce refresh-token rotation with server-side token family tracking.
- Add message queue for asynchronous notifications and email delivery.
- Improve search capabilities with full-text indexing and ranking.
- Add distributed lock or optimistic lock strategy for high-contention booking windows.

## Tech Stack

- **Backend**: Spring Boot, Spring Security, Spring Data JPA, Java 17, MySQL, JWT, Maven.
- **Frontend**: React, Tailwind CSS, React Router, Axios.
- **Tooling**: Postman, Git.

## Project Structure

```text
MindMeter/
├── backend/
│   ├── src/main/java/com/shop/backend/
│   │   ├── controller/
│   │   ├── service/
│   │   ├── repository/
│   │   ├── model/
│   │   ├── security/
│   │   └── config/
│   └── src/test/java/
├── frontend/
├── database/
│   └── MindMeter.sql
└── README.md
```

## How To Run

### Prerequisites

- Java 17+
- Node.js 18+
- MySQL 8+
- Maven 3.8+

### Backend

```bash
cd backend
cp src/main/resources/application.properties.example src/main/resources/application.properties
mvn clean install
mvn spring-boot:run
```

Backend runs at `http://localhost:8080`.

### Frontend

```bash
cd frontend
npm install
npm start
```

Frontend runs at `http://localhost:3000`.

### Database

```sql
CREATE DATABASE mindmeter;
```

Then import:

```bash
mysql -u root -p mindmeter < database/MindMeter.sql
```

## Testing

```bash
cd backend
mvn test
```

## License

This project is licensed under the Apache License 2.0. See `LICENSE`.
