# Eventzen Backend - Business Logic Summary

## 1. Authentication & Identity (`AuthController`, `UserController`)
- **Endpoints:** `POST /api/v1/auth/register`, `POST /api/v1/auth/login`, `GET | PUT | DELETE /api/v1/users/{id}`
- **Business Logic:** 
  - **Uniqueness checks:** Gracefully catches duplicate username/emails on registration.
  - **JWT Authorization:** Generates access tokens and authenticates principals.
  - **IDOR Protection:** Leverages Spring Security expressions (`@PreAuthorize("hasRole('ADMIN') or #id == principal.id")`) to strictly ensure users can only ever query or soft-delete their *own* profiles.

## 2. Infrastructure (`VenueService`, `VendorService`)
- **Endpoints:** Standard CRUD at `/api/v1/venues` and `/api/v1/vendors`
- **Business Logic:**
  - **Authorization:** Only Admins can create, delete, or update Venues/Vendors.
  - **Dynamic Venue Pricing:** `Venue` entities enforce a strict `@Column` baseline parameter for `pricePerSeat`, cascading accurate total pricing constraints universally.
  - **Soft Deletes:** Deletions trigger a state change (`is_deleted = 1`) via Hibernate `@SQLDelete`. All read queries automatically filter deleted records out using heavily optimized `@SQLRestriction` clauses.
  - **Deletion Safety:** The `VendorService` prevents deleting vendors who are currently assigned to events. Admins must first de-assign them to maintain data integrity.

  - **Recovery:** Custom native queries mapped to `/restore` paths allow Admins to undo accidental deletions.

## 3. Core Engine (`EventService`)
- **Endpoints:** Standard CRUD at `/api/v1/events`
- **Business Logic:**
  - **Relational Integrity:** Links explicitly verified `Venue` and `User` (Organizer) entities.
  - **Concurrency tracking:** Initializes `soldTickets = 0` and includes a `@Version` field to monitor state changes. Computes advanced metrics like Total Seats vs Distributed Tickets by injecting `venueCapacity` directly into standard `EventDTO` responses.
  - **Vendor Assignment Workflows:** Admins orchestrate many-to-many relationship pipelines by linking `Vendor` entities to individual `Event` objects (`EventVendor`). Robust mapping logic ensures the system remains stable even if a vendor record is later modified or removed.
  - **Manual Management:** Admins possess full control to "Sever Connections" (de-assign) or "Recall Assign" directly from the Event Detail interface, enabling real-time logistics adjustments.
  - **Automated Scheduling (`VendorAssignmentScheduler`):** Spring Boot's native `@Scheduled` library simulates organic B2B behavior by continuously executing a polling cycle, randomly shifting `PENDING` vendor authorizations to `ACCEPTED` or `REJECTED`.
  - **Venue Lookups:** Implements nested queries to find active events for a specific venue calendar.

## 4. Transaction Engine (`RegistrationService`)
- **Endpoints:** `/api/v1/bookings`, plus Admin endpoints like `/admin/bookings/{id}/approve`
- **Business Logic:**
- **Transaction-Safe Seat Allocation**: Before a registration saves, the system verifies available capacity.
- **Optimistic Locking**: Leverages `@Version` to prevent race conditions during bulk ticket purchases.
- **Approval-Linked Inventory**: Unlike standard systems, seats are only **decremented** from inventory when an Admin explicitly triggers `/approve`. This provides precise control over high-demand event bookings.
- **Automatic Capacity Reversion**: If an admin hits `/reject` or a user hits `/cancel` (for an approved booking), the service dynamically restores the physical slot.

## 5. Support Microservice (Node.js & Express)
- **Architecture**: Separated from core logic to demonstrate high-availability microservices.
- **Tech Stack**: Node.js, Express, Sequelize.
- **Inter-service Auth**: Uses a shared JWT Secret for identity propagation (SSO) from Spring Boot.
- **Business Logic**: Handles ticket lifecycle (OPEN, IN_PROGRESS, RESOLVED) and customer history logs with direct database access via Sequelize.

## 6. System Hardening & Optimization (Final Gap Closures)
- **Admin Account Seeding:** A `DataInitializer` automatically validates database state on startup and dynamically injects a default `ROLE_ADMIN` if none exists—enabling safe registration lockdown.
- **Pagination & Scaling:** The `Venue` and `Event` Controllers/Services employ Spring's `Pageable` to optimally serve sliceable `Page<T>` grids, ensuring browser crash-safety on the React UI.
- **Physical Ticket Generation:** `RegistrationService` seamlessly parses distinct `TicketDTO` quantities to build discrete physical `Ticket` entities tied locally to respective `Registration` models to maintain 1:1 state parity.
- **Universal Superclass Auditing:** All entities now natively inherit `@Version`, `createdAt`, `updatedAt`, `isDeleted`, and `customId` attributes from the polymorphic `@MappedSuperclass` `BaseEntity`.

---

## 7. Containerized Deployment (Docker)
The entire ecosystem is containerized for standardized deployment across environments.
- **Orchestration**: `docker-compose.yml` manages four distinct containers:
  1. `eventzen-db`: MySQL 8.0 instance.
  2. `eventzen-spring-core`: The Java backend on port 8080.
  3. `eventzen-node-support`: The Node.js microservice on port 3000.
  4. `eventzen-react-ui`: The Nginx-powered frontend on port 80.
- **One-Command Setup**: `docker-compose up --build` initializes the database, builds all service images, and establishes the networking bridge between them automatically.
