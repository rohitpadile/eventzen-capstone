# Eventzen - User Flow & System Journey Map

This document outlines the operational pathways for both End-Users (Customers) and System Managers (Admins).

## 1. The Customer Flow (End-User Experience)
The Customer is the primary user seeking to discover venues, book events, and manage their attendance.

### 📍 Entry Point:
**Landing Page** → **Auth Modal** (Unified Login/Register) → **Customer Dashboard**

### 🌿 Operational Branches:
*   **Branch A: The Booking Funnel (Core Path)**
    *   `View Venues` → Filter by Location/Price → `Check Availability` → `Select Event` → `Choose Ticket Quantity` → `Submit Booking Request` → **Status: PENDING**.
*   **Branch B: Booking Management**
    *   `Dashboard` → View "My Bookings" → Select Booking → `Cancel Booking`.
    *   *System Logic*: Capacity is preserved during PENDING and only fully decremented once an Admin approves.
*   **Branch C: Microservice Support Center**
    *   `Help Desk` → `Submit Ticket` → View Status (OPEN/RESOLVED). 
    *   *Service Flow*: These requests are handled by the high-availability **Node.js Microservice**.
*   **Branch D: Profile Integrity**
    *   `Manage Profile` → Update Identity Info → `Save`. 
    *   *Security*: Backend enforces strict ownership checks (IDOR protection).

---

## 2. The Admin Flow (Logistics & Management)
The Admin is the staff member responsible for orchestrating venues, vendors, and verifying bookings.

### 📍 Entry Point:
**Admin Login** → **Admin Dashboard** (Analytics Overview)

### ⚙️ Operational Branches:
*   **Branch A: Venue Logistics**
    *   `Venue Management` → Add/Edit/Remove Venues. Define **Capacity** and **Price Per Seat**.
*   **Branch B: Vendor Relations**
    *   `Vendor Management` → Onboard new Caterers/Technicians. Monitor automatic B2B status shifts (Scheduled approvals).
*   **Branch C: Hybrid Event Orchestration**
    *   `Event Management` → `Link Venue` → `Assign Multiple Vendors` → Set Date → **Publish**.
*   **Branch D: The Gatekeeper (Booking Approvals)**
    *   `Booking Queue` → Filter by PENDING → Review Capacity → **APPROVE** (Decrements inventory) OR **REJECT** (Releases hold).
*   **Branch E: Support Agent Portal**
    *   `Support Dashboard` → Open Ticket → `Start/Resolve` inquiry.

---

## 🏗️ System Architecture Refinement
- **Frontend**: React-based SPA with role-based routing guards.
- **Backend (Core)**: Spring Boot managing large-scale transactions and security.
- **Backend (Microservice)**: Node.js/Express handling support operations for isolated scalability.
- **Shared Data**: MySQL instance shared via shared-secret JWT authentication.
