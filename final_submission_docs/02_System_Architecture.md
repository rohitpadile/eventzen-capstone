# EventZen — System Architecture

## 1. Architectural Overview

EventZen is built on a **hybrid monolithic/microservice architecture** that balances the simplicity of a monolithic core with the scalability benefits of service decomposition. The system comprises four primary runtime components, each deployed as an independent Docker container and connected through a shared bridge network.

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Docker Bridge Network                        │
│                                                                     │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐            │
│  │  React SPA   │   │ Spring Boot  │   │   Node.js    │            │
│  │  (Nginx)     │   │   Core API   │   │  Support API │            │
│  │  Port 80     │   │  Port 8080   │   │  Port 3000   │            │
│  └──────┬───────┘   └──────┬───────┘   └──────┬───────┘            │
│         │                  │                   │                    │
│         │         ┌────────┴───────────────────┘                    │
│         │         │                                                 │
│         │    ┌────┴─────────┐                                       │
│         │    │   MySQL 8.0  │                                       │
│         │    │  Port 3306   │                                       │
│         │    └──────────────┘                                       │
└─────────┴───────────────────────────────────────────────────────────┘
```

| Component              | Technology           | Responsibility                                          |
| :--------------------- | :------------------- | :------------------------------------------------------ |
| **React SPA**          | React, Vite, Nginx   | Client-side rendering, role-based navigation, API consumption |
| **Spring Boot Core**   | Java, Spring Boot    | Authentication, venue/event/booking management, RBAC     |
| **Node.js Support**    | Node.js, Express     | Support ticket lifecycle, isolated from core transactions |
| **MySQL Database**     | MySQL 8.0            | Shared relational persistence for all services            |

---

## 2. Frontend Architecture

### 2.1 Technology Decisions

The frontend is a **React Single-Page Application** scaffolded with Vite for optimized development builds and fast hot-module replacement. TailwindCSS provides the styling layer, and Axios serves as the HTTP client with interceptor-based JWT injection for all authenticated requests.

### 2.2 Directory Structure

The frontend follows a strict DRY (Don't Repeat Yourself) architecture, separating concerns into three top-level directories:

```
src/
├── components/       # Reusable UI elements (Navbar, Modals, ProtectedRoute)
├── screens/          # Route-level views (LoginScreen, AdminDashboard, etc.)
└── utils/            # Centralized logic, providers, and API clients
    ├── AuthContext.jsx      # Global authentication state provider
    ├── apiClient.js         # Axios instance → Spring Boot (Port 8080)
    ├── nodeApiClient.js     # Axios instance → Node.js Service (Port 3000)
    ├── api/                 # Domain-specific API modules
    │   ├── authApi.js
    │   ├── venueApi.js
    │   ├── eventApi.js
    │   ├── bookingApi.js
    │   └── supportApi.js
    └── helpers.js           # Date formatting, validators, constants
```

### 2.3 State Management

Authentication state is managed through a top-level `<AuthProvider>` context that wraps the entire React component tree. Upon successful login, the JWT is persisted to `localStorage`, and `jwt-decode` extracts the user's `role` and `id` into a reactive state variable accessible to all components.

### 2.4 Dual API Client Pattern

The frontend maintains two independent Axios instances to communicate with the respective backend services:

- **`apiClient.js`** — Configured for the Spring Boot Core API on port 8080. An Axios request interceptor automatically attaches the `Authorization: Bearer <token>` header to every outbound request (excluding `/auth/*` endpoints).
- **`nodeApiClient.js`** — Configured for the Node.js Support Service on port 3000, using the same interceptor pattern to propagate the JWT for shared authentication.

### 2.5 Routing and Role-Based Access Control

React Router v6 provides declarative client-side routing. A custom `<ProtectedRoute>` wrapper component enforces role-based access at the route level:

```jsx
const ProtectedRoute = ({ children, requiredRole }) => {
   const { user } = useAuth();
   if (!user) return <Navigate to="/login" replace />;
   if (requiredRole && user.role !== requiredRole) return <Navigate to="/unauthorized" replace />;
   return children;
};
```

**Route Inventory:**

| Route Category    | Paths                                                                                      |
| :---------------- | :----------------------------------------------------------------------------------------- |
| **Public**        | `/` (Landing), `/login`, `/register`                                                       |
| **Customer**      | `/dashboard`, `/book`, `/book/events/:eventId`, `/support`, `/profile`                     |
| **Admin**         | `/admin/dashboard`, `/admin/venues`, `/admin/events`, `/admin/events/:eventId`, `/admin/vendors`, `/admin/bookings`, `/admin/support` |

---

## 3. Backend Architecture — Spring Boot Core

### 3.1 Service Layer Design

The Spring Boot application serves as the **primary transactional engine**, owning authentication, venue logistics, event orchestration, vendor management, and the booking lifecycle. It follows a standard layered architecture: Controller → Service → Repository → Database.

### 3.2 Authentication and Identity

The `AuthController` manages user registration and login. On successful authentication, the service issues a signed JWT containing the user's `id` and `role` claims. The JWT secret is shared with the Node.js microservice to enable cross-service identity verification without inter-service API calls.

**Security Features:**
- **Role-Based Access Control (RBAC):** Spring Security method-level annotations (`@PreAuthorize`) restrict endpoint access by role (`ROLE_ADMIN`, `ROLE_CUSTOMER`).
- **IDOR Protection:** Ownership-aware authorization expressions (e.g., `@PreAuthorize("hasRole('ADMIN') or #id == principal.id")`) ensure users can only query or modify their own resources.
- **Admin Seeding:** A `DataInitializer` component validates the database on application startup and injects a default administrator account if none exists, enabling secure registration lockdown.

### 3.3 Venue and Vendor Management

Venues and Vendors are the foundational infrastructure entities.

- **Venue Management** — Full CRUD operations with admin-only write access. Each venue defines a `capacity` and `pricePerSeat`, which cascade into event-level pricing calculations.
- **Vendor Management** — Full CRUD with referential integrity guards. The `VendorService` prevents deletion of vendors currently assigned to active events, requiring explicit de-assignment first.
- **Soft Deletion** — Hibernate `@SQLDelete` annotations convert `DELETE` operations into `is_deleted = 1` state changes. Read queries use `@SQLRestriction` to automatically filter soft-deleted records. Admin-accessible `/restore` endpoints enable recovery of accidentally deleted records.

### 3.4 Event Orchestration

The `EventService` is the platform's core engine, linking venues, vendors, and bookings into a cohesive event lifecycle.

- **Entity Relationships** — Each event is bound to a verified `Venue` and a `User` (the organizer). The system tracks `soldTickets` and computes remaining capacity by comparing against `venue.capacity`.
- **Vendor Assignment Workflows** — Administrators manage a many-to-many relationship between events and vendors through the `EventVendor` join entity. Assignments carry a status field (`PENDING`, `ACCEPTED`, `REJECTED`) to model real-world vendor confirmation workflows.
- **Automated Vendor Scheduling** — A Spring `@Scheduled` task (`VendorAssignmentScheduler`) simulates organic vendor acceptance behavior by periodically transitioning `PENDING` assignments to `ACCEPTED` or `REJECTED`, demonstrating automated B2B workflow processing.

### 3.5 Booking and Registration Engine

The `RegistrationService` manages the full booking lifecycle with an approval-gated inventory model:

1. **Booking Submission** — Customers submit a booking request specifying an event and ticket quantities. The service validates available capacity and persists the registration in `PENDING` status.
2. **Ticket Generation** — The service parses `TicketDTO` payloads to generate discrete physical `Ticket` entities, maintaining a 1:1 relationship with each registration for granular tracking.
3. **Admin Approval** — Seat inventory is decremented from the event's capacity **only** when an administrator explicitly approves the booking. This prevents premature inventory consumption in high-demand scenarios.
4. **Rejection and Cancellation** — Both admin rejection and customer cancellation trigger automatic capacity restoration, ensuring inventory accuracy at all times.
5. **Concurrency Safety** — JPA's `@Version` annotation provides optimistic locking, preventing race conditions when multiple concurrent booking requests target the same event.

### 3.6 Pagination

The `Venue` and `Event` controllers leverage Spring's `Pageable` interface to serve sliceable `Page<T>` responses, ensuring the frontend can render large datasets efficiently without memory pressure.

---

## 4. Backend Architecture — Node.js Support Microservice

### 4.1 Purpose and Rationale

Customer support operations are deliberately isolated into a separate Node.js/Express microservice. This architectural decision provides several benefits:

- **Polyglot Demonstration** — Two different language runtimes (Java and JavaScript) collaborating within the same system.
- **Fault Isolation** — Failures or performance degradation in the support service do not impact core booking and event operations.
- **Independent Scalability** — The support service can be horizontally scaled independently to handle high-volume ticket traffic.

### 4.2 Technology Stack

| Layer          | Technology    | Equivalent in Spring Boot          |
| :------------- | :------------ | :--------------------------------- |
| HTTP Server    | Express       | Spring MVC / Tomcat                |
| ORM            | Sequelize     | Hibernate / JPA                    |
| Auth Middleware | jsonwebtoken  | Spring Security Filter Chain       |
| Database       | MySQL 8.0     | MySQL 8.0 (same shared instance)   |

### 4.3 Shared Authentication Mechanism

The Node.js service implements **stateless identity verification** using the same JWT secret configured in the Spring Boot application. The authentication middleware (`auth.js`) intercepts every incoming request, decodes the JWT, and attaches the user's `id` and `role` to the request object. No network call to the Spring Boot service is required.

**Request Pipeline:**

```
Frontend  →  nodeApiClient (Port 3000)
                   ↓
          auth.js Middleware  →  JWT Verification (Shared Secret)
                   ↓
       supportTicketController.js  →  Business Logic
                   ↓
          Sequelize ORM  →  MySQL Database
                   ↓
          JSON Response  →  Frontend
```

### 4.4 Support Ticket Lifecycle

The microservice manages the complete support ticket lifecycle:

| Status        | Description                                    |
| :------------ | :--------------------------------------------- |
| `OPEN`        | Ticket submitted by customer, awaiting review  |
| `IN_PROGRESS` | Admin has acknowledged and is investigating    |
| `RESOLVED`    | Issue has been resolved and ticket is closed   |

The service supports soft deletion with admin-only restore capability, mirroring the data integrity patterns established in the Spring Boot core.

---

## 5. Complete API Contract Reference

All authenticated endpoints require the `Authorization: Bearer <token>` header, automatically injected by the frontend Axios interceptors.

### 5.1 Authentication and User Management

| Method | Endpoint                  | Access   | Request Body                                    | Response                |
| :----- | :------------------------ | :------- | :---------------------------------------------- | :---------------------- |
| POST   | `/api/v1/auth/login`      | Public   | `{ username, password }`                        | JWT Token               |
| POST   | `/api/v1/auth/register`   | Public   | `{ username, email, password, role }`           | User confirmation       |
| GET    | `/api/v1/users/{id}`      | Owner/Admin | —                                            | User profile            |
| PUT    | `/api/v1/users/{id}`      | Owner/Admin | `{ username, email }`                        | Updated profile         |

### 5.2 Venue Management (Admin Only)

| Method | Endpoint                             | Request Body                                    | Response              |
| :----- | :----------------------------------- | :---------------------------------------------- | :-------------------- |
| GET    | `/api/v1/venues?page={p}&size={s}`   | —                                               | `Page<VenueDTO>`      |
| POST   | `/api/v1/venues`                     | `{ name, address, capacity, amenities }`        | Created venue         |
| PUT    | `/api/v1/venues/{id}`                | `{ name, address, capacity, amenities }`        | Updated venue         |
| DELETE | `/api/v1/venues/{id}`                | —                                               | Soft-deleted venue    |

### 5.3 Event Management

| Method | Endpoint                                          | Access | Request Body                                        | Response                |
| :----- | :------------------------------------------------ | :----- | :-------------------------------------------------- | :---------------------- |
| GET    | `/api/v1/events?page={p}&size={s}`                | All    | —                                                   | `Page<EventDTO>`        |
| GET    | `/api/v1/events/{id}`                             | All    | —                                                   | Enhanced `EventDTO` with capacity metrics |
| GET    | `/api/v1/events/venue/{venueId}?page={p}&size={s}` | All  | —                                                   | Events at venue         |
| POST   | `/api/v1/events`                                  | Admin  | `{ title, description, eventDate, venueId, organizerId }` | Created event    |
| PUT    | `/api/v1/events/{id}`                             | Admin  | `{ title, description, eventDate }`                 | Updated event           |

### 5.4 Vendor Assignment (Admin Only)

| Method | Endpoint                                          | Response                            |
| :----- | :------------------------------------------------ | :---------------------------------- |
| POST   | `/api/v1/events/{eventId}/vendors/{vendorId}`     | Vendor assigned to event            |
| DELETE | `/api/v1/events/{eventId}/vendors/{vendorId}`     | Vendor de-assigned from event       |
| GET    | `/api/v1/events/{eventId}/vendors`                | List of `EventVendorDTO` with status |

### 5.5 Booking and Registration

| Method | Endpoint                                          | Access   | Request Body                                            | Response          |
| :----- | :------------------------------------------------ | :------- | :------------------------------------------------------ | :---------------- |
| POST   | `/api/v1/bookings`                                | Customer | `{ userId, eventId, tickets: [{ ticketType, price, quantity }] }` | Booking confirmation |
| GET    | `/api/v1/bookings/user/{userId}`                  | Customer | —                                                       | User's bookings   |
| GET    | `/api/v1/bookings/admin/bookings`                 | Admin    | —                                                       | All bookings      |
| PUT    | `/api/v1/bookings/{id}/cancel`                    | Customer | —                                                       | Cancelled booking |
| PUT    | `/api/v1/bookings/admin/bookings/{id}/approve`    | Admin    | —                                                       | Approved booking  |
| PUT    | `/api/v1/bookings/admin/bookings/{id}/reject`     | Admin    | —                                                       | Rejected booking  |

### 5.6 Support Tickets (Node.js Microservice)

| Method | Endpoint                                | Access   | Request Body                                | Response               |
| :----- | :-------------------------------------- | :------- | :------------------------------------------ | :--------------------- |
| POST   | `/api/v1/tickets`                       | Customer | `{ customerId, subject, description }`      | Created ticket         |
| GET    | `/api/v1/tickets/customer/{customerId}` | Customer | —                                           | Customer ticket history |
| PUT    | `/api/v1/tickets/{id}?status={status}`  | Admin    | —                                           | Updated ticket status  |
| POST   | `/api/v1/tickets/{id}/restore`          | Admin    | —                                           | Restored ticket        |

---

## 6. Cross-Cutting Concerns

### 6.1 Soft Deletion Strategy

All entities across both services implement soft deletion rather than physical record removal:

- **Spring Boot** — Hibernate `@SQLDelete` overrides the default `DELETE` SQL to set `is_deleted = 1`. `@SQLRestriction("is_deleted = 0")` filters soft-deleted records from all standard queries. Native `@Query` methods on `/restore` endpoints allow administrators to reverse deletions.
- **Node.js** — Sequelize model definitions enforce the same `is_deleted` pattern, with dedicated restore API endpoints.

### 6.2 Universal Entity Auditing

All JPA entities inherit from a shared `BaseEntity` (`@MappedSuperclass`) that provides:

| Field        | Type        | Purpose                                     |
| :----------- | :---------- | :------------------------------------------- |
| `customId`   | String/UUID | Business-facing unique identifier            |
| `createdAt`  | Timestamp   | Record creation timestamp                    |
| `updatedAt`  | Timestamp   | Last modification timestamp                  |
| `version`    | Integer     | Optimistic locking revision counter          |
| `isDeleted`  | Boolean     | Soft-delete flag                             |

### 6.3 Error Handling

The backend provides structured error responses with appropriate HTTP status codes. Uniqueness constraint violations (duplicate usernames/emails) are gracefully caught and returned as user-friendly validation messages rather than raw database exceptions.
