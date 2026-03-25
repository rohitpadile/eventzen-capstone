# EventZen — Executive Summary

## 1. Project Overview

**EventZen** is a full-stack enterprise event management platform designed to streamline the end-to-end lifecycle of venue logistics, event orchestration, ticket booking, and customer support. The system serves two distinct user personas — **Administrators** who manage operational infrastructure and **Customers** who discover, book, and attend events — through dedicated, role-secured portals.

The platform demonstrates professional-grade software engineering practices, including a hybrid monolithic/microservice architecture, shared stateless authentication, containerized deployment, and comprehensive data integrity safeguards.

| Attribute              | Detail                                                        |
| :--------------------- | :------------------------------------------------------------ |
| **Project Name**       | EventZen                                                      |
| **Repository**         | [eventzen-capstone](https://github.com/rohitpadile/eventzen-capstone) |
| **Architecture Style** | Hybrid Monolithic Core + Microservice                         |
| **Deployment**         | Fully containerized via Docker Compose (single-command setup) |

---

## 2. Business Value Proposition

EventZen addresses a common operational gap in event management: the disconnect between logistics planning, real-time capacity management, and customer engagement. By consolidating these concerns into a single platform, EventZen delivers the following business outcomes:

- **Centralized Operations** — Administrators manage venues, vendors, events, and booking approvals from a unified dashboard, eliminating the need for disconnected spreadsheets and communication channels.
- **Controlled Inventory Management** — A deliberate approval-gated booking model ensures that seat inventory is decremented only upon explicit administrative authorization, preventing overbooking in high-demand scenarios.
- **Isolated Support Operations** — Customer support is offloaded to a dedicated Node.js microservice, ensuring that high-volume ticket traffic does not degrade the performance of core transactional operations.
- **Audit-Ready Data Model** — Every entity in the system inherits standardized audit fields (timestamps, version control, and soft-delete flags), enabling full traceability of all data mutations.

---

## 3. Technology Stack

The platform is built on a modern, polyglot technology stack chosen for its alignment with enterprise development standards and real-world scalability patterns.

### 3.1 Frontend — React Single-Page Application
| Technology       | Purpose                                        |
| :--------------- | :--------------------------------------------- |
| React (Vite)     | Component-driven UI with optimized hot reloading |
| React Router v6+ | Declarative, role-guarded client-side routing  |
| Axios            | HTTP client with interceptor-based JWT injection |
| TailwindCSS      | Utility-first styling framework                |
| jwt-decode       | Client-side token parsing for role resolution  |

### 3.2 Backend — Spring Boot Core Service
| Technology       | Purpose                                     |
| :--------------- | :------------------------------------------ |
| Java / Spring Boot | RESTful API server and business logic engine |
| Spring Security  | JWT-based authentication and RBAC authorization |
| Hibernate (JPA)  | Object-relational mapping with soft-delete support |
| MySQL 8.0        | Relational data persistence                  |

### 3.3 Backend — Node.js Support Microservice
| Technology       | Purpose                                     |
| :--------------- | :------------------------------------------ |
| Node.js / Express | Lightweight HTTP server for support operations |
| Sequelize        | JavaScript ORM for MySQL integration         |
| jsonwebtoken     | Shared-secret JWT verification for SSO       |

### 3.4 Infrastructure
| Technology       | Purpose                                       |
| :--------------- | :-------------------------------------------- |
| Docker & Docker Compose | Multi-container orchestration            |
| Nginx            | Production-grade static file serving for the React build |
| MySQL 8.0        | Shared relational database across all services |

---

## 4. Architectural Highlights

EventZen employs a **hybrid architecture** — a monolithic Spring Boot core handles the primary transactional workloads (authentication, venue management, event orchestration, and booking lifecycle), while a standalone Node.js microservice manages customer support operations independently.

Key architectural decisions include:

- **Shared Stateless Authentication** — Both services validate JWTs using an identical shared secret. The Spring Boot service issues tokens at login; the Node.js service independently verifies them in its middleware layer. No inter-service calls are required for authentication, adhering to the stateless identity propagation pattern.
- **Single Shared Database** — Both services connect to the same MySQL instance, simplifying data consistency while maintaining clear schema ownership boundaries.
- **Approval-Gated Booking Model** — Unlike conventional immediate-decrement booking systems, EventZen holds seat reservations in a `PENDING` state. Inventory is only decremented upon explicit admin approval, and automatically restored upon rejection or cancellation.
- **Optimistic Concurrency Control** — JPA's `@Version` annotation prevents race conditions during concurrent booking attempts, ensuring transactional integrity under load.

---

## 5. API Surface Summary

### Spring Boot Core — Port 8080

| Method | Endpoint                                | Access    | Description                          |
| :----- | :-------------------------------------- | :-------- | :----------------------------------- |
| POST   | `/api/v1/auth/login`                    | Public    | User authentication and JWT issuance |
| POST   | `/api/v1/auth/register`                 | Public    | User registration with role assignment |
| GET    | `/api/v1/venues`                        | All       | Paginated venue listing              |
| POST   | `/api/v1/events`                        | Admin     | Event creation and venue linkage     |
| POST   | `/api/v1/bookings`                      | Customer  | Booking request submission           |
| PUT    | `/admin/bookings/{id}/approve`          | Admin     | Booking approval with inventory decrement |

### Node.js Support Service — Port 3000

| Method | Endpoint                                | Access    | Description                          |
| :----- | :-------------------------------------- | :-------- | :----------------------------------- |
| POST   | `/api/v1/tickets`                       | Customer  | Support ticket creation              |
| GET    | `/api/v1/tickets/customer/{id}`         | Customer  | Customer ticket history retrieval    |
| PUT    | `/api/v1/tickets/{id}`                  | Admin     | Ticket status management             |
| POST   | `/api/v1/tickets/{id}/restore`          | Admin     | Soft-deleted ticket recovery         |

---

## 6. Security and Data Integrity

| Concern                  | Implementation                                                                    |
| :----------------------- | :-------------------------------------------------------------------------------- |
| **Authentication**       | JWT-based stateless authentication with shared-secret cross-service verification  |
| **Authorization (RBAC)** | Spring Security role guards (`ROLE_ADMIN`, `ROLE_CUSTOMER`) on all protected endpoints |
| **IDOR Protection**      | `@PreAuthorize` expressions ensure users can only access their own resources       |
| **Optimistic Locking**   | `@Version`-based concurrency control on booking transactions                       |
| **Soft Deletion**        | Hibernate `@SQLDelete` and `@SQLRestriction` annotations; Sequelize `paranoid` mode |
| **Audit Trail**          | All entities inherit `createdAt`, `updatedAt`, `version`, and `isDeleted` from a shared `BaseEntity` superclass |

---

## 7. Document Index

This submission package contains the following companion documents:

| #  | Document                                         | Description                                       |
| :- | :----------------------------------------------- | :------------------------------------------------ |
| 01 | Executive Summary *(this document)*              | Project overview, business value, and tech stack   |
| 02 | System Architecture                              | Detailed architectural design and API contracts    |
| 03 | Database Design and ERD                          | Relational schema, entity relationships, and design rationale |
| 04 | User Flows and Wireframes                        | UI/UX strategy for Customer and Admin portals      |
| 05 | Deployment Guide                                 | Docker-based production environment setup          |
