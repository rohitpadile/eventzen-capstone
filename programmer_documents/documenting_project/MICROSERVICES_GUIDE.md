# Eventzen Microservices Architecture Guide

This guide explains the hybrid architecture of the Eventzen platform, where a **Spring Boot** core co-exists with a **Node.js (Express)** microservice to demonstrate distributed computing.

---

## 🏗️ Architecture Overview

Eventzen now uses a **Distributed Architecture** to demonstrate modern microservices practices.

### 1. Spring Boot Core (Port 8080)
- **Responsibility**: User Authentication, Event Management, Venue Logistics, and Ticket Bookings.
- **Tech Stack**: Java, Spring Boot, Spring Security, Hibernate (JPA).
- **Security**: Issues JWT tokens and manages User Roles.
- **Database**: Primary owner of the `users`, `events`, and `bookings` schemas.

### 2. Node.js Support Service (Port 3000)
- **Responsibility**: Dedicated handling of Support Tickets and Customer Enquiries.
- **Tech Stack**: Node.js, Express, Sequelize (ORM).
- **Security**: Verifies JWT tokens issued by Spring Boot using the same Secret Key.
- **Database**: Shares the `support_tickets` table with the core DB.

---

## 🔐 Shared Authentication (The "Glue")

One of the most powerful features of this setup is the **Shared Secret authentication**.

- Both services share the same `JWT_SECRET` string.
- When a user logs in via Spring Boot, they receive a signed JWT token.
- When the Frontend calls the Node.js service, it sends that same token in the `Authorization` header.
- The Node.js service (`auth.js` middleware) uses the secret to decode the token locally, verifying the user's identity and roles without needing to call the Spring Boot service. This is a common **Stateless Identity** pattern used in high-scale systems.

---

## 🔄 Service-to-Service Flow (Support Tickets)

### 🧩 The Request Pipeline:
1.  **Frontend**: Calls `supportApi.createTicket()`, which uses the `nodeApiClient` (pointing to port 3000).
2.  **Auth Middleware (auth.js)**: Verifies the JWT token and attaches the user's role and ID to the request object.
3.  **Controller (supportTicketController.js)**: Receives the data and calls **Sequelize**.
4.  **Database**: Node.js directly reads/writes to the same MySQL database instance as Spring Boot.
5.  **Response**: Frontend receives the data and updates the UI.

---

## 🎓 How the Node.js Code Works

### 1. Sequelize (The ORM)
In Spring Boot, you used Hibernate/JPA. In Node.js, we use **Sequelize**.
- It maps the `SupportTicket` JavaScript class directly to the `support_tickets` database table.
- `SupportTicket.findAll()` is the equivalent of a `SELECT * FROM support_tickets` query but written in JavaScript.

### 2. Express (The Server)
- **Routes**: Define which URL paths map to which controller functions.
- **Controller**: Contains the logic. For example, when you create a ticket, Node.js automatically generates a `customId` (UUID) to match the Spring Boot data pattern.

### 3. Middleware
The `auth.js` file is the "Gatekeeper". It's a piece of code that runs **before** your business logic. If the token is invalid or the user's role isn't authorized (e.g., a customer trying to delete an admin ticket), it stops the request early with a `403 Forbidden` error.

---

## 🛠️ Why this is a "Practice of Microservices"?

-   **Polyglot**: You are using two different programming languages (Java and JavaScript) to solve different parts of the same problem.
-   **Independence**: If the Spring Boot server goes down for maintenance, users could (theoretically) still submit support tickets if the Node.js service is still running!
-   **Scalability**: You can deploy 10 copies of the Support Service on different servers to handle huge ticket traffic without slowing down the core booking system.
