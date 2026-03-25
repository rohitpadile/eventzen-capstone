# Eventzen Frontend Architecture & User Flows (Updated)

## 1. Tech Stack & Initialization
- **Build Tool:** Vite + React (Extremely fast HMR and optimized production builds vs CRA).
- **Core Dependencies:** - `axios` (API integration and interceptors)
  - `react-router-dom` (v6+ for declarative routing)
  - `jwt-decode` (Parsing payload claims from the Spring Boot token)
  - `tailwindcss` (Utility-first styling for rapid, premium UI building)
  - `lucide-react` (Modern vector iconography)
  - `react-hot-toast` (Sleek notification popups for success/error feedback)
- **Directory Structure Target (Strict DRY Architecture):**
  ```text
  src/
  ├── components/  # Reusable UI only (Buttons, Modals, Navbar, ProtectedRoute)
  ├── screens/     # Route-level views (LoginScreen, AdminVenueScreen, CustomerDashboard)
  └── utils/       # Centralized logic, providers, and configurations
      ├── AuthContext.jsx # Global user state provider
      ├── apiClient.js    # Spring Boot Axios instance (Port 8080)
      ├── nodeApiClient.js # Node.js Axios instance (Port 3000)
      ├── api/            # Grouped try/catch API calls
      │   ├── authApi.js
      │   ├── venueApi.js
      │   ├── bookingApi.js
      │   └── supportApi.js # Now utilizes nodeApiClient
      └── helpers.js      # Date formatting, validators, constants
  ```

## 2. State Management & API Architecture
- **State Management:** A top-level `<AuthProvider>` (located in `utils/AuthContext.jsx`) wrapping the React tree.
- **Token Management:** Upon successful login, `token` is safely committed to `localStorage`. `jwtDecode(token)` extracts the `role` and `id` into a reactive `user` state variable.
- **DRY API Strategy:**
  - `utils/apiClient.js`: Communicates with the Spring Boot Core (Port 8080).
  - `utils/nodeApiClient.js`: Communicates with the Support Microservice (Port 3000).
  - Domain-specific files (like `utils/api/venueApi.js`) contain the actual `try/catch` network requests, ensuring zero redundant fetch logic.

## 3. Routing & Security Strategy (React Router)
- **Public Routes:** `/` (Marketing Home), `/login`, `/register`.
- **Customer Protected Routes:** `/dashboard`, `/book`, `/book/events/:eventId`, `/support`, `/profile`.
- **Admin Protected Routes:** `/admin/dashboard`, `/admin/venues`, `/admin/events`, `/admin/events/:eventId`, `/admin/vendors`, `/admin/bookings`, `/admin/support`.
- **Role-Based Guards:** A `<ProtectedRoute>` wrapper intercepts route access:
  ```jsx
  const ProtectedRoute = ({ children, requiredRole }) => {
     const { user } = useAuth();
     if (!user) return <Navigate to="/login" replace />;
     if (requiredRole && user.role !== requiredRole) return <Navigate to="/unauthorized" replace />;
     return children;
  }
  ```

## 4. Page & Component Roadmaps (Execution Flow)

### Phase A: Core Initialization & Auth
- Establish Vite template with TailwindCSS.
- Build `utils/apiClient.js` and `utils/AuthContext.jsx`.
- Build `LoginScreen` and `RegisterScreen` featuring dynamic unique-check field validation mirroring the backend constraints.

### Phase B: Admin Portal
**(Role: `ROLE_ADMIN` - Logistics Management)**
- **Venue Logistics:** Data grids with Pagination toggles to execute `POST`/`PUT`/`DELETE` on Venues. Support for `pricePerSeat` for variable pricing structures.
- **Vendor Logistics:** Build tables mapping external caterers/supplies. (Vendors possess `ROLE_VENDOR` system roles).
- **Event Creation & Management:** Heavy form mapping linking `Venue` dropdowns + multi-`Vendor` checkbox assignments + Date/Time selections → Publishes to core table and dispatches related `EventVendor` assignments. Event Cards link to a dedicated **Event Management Matrix** at `/admin/events/:id` supporting: live Seats Remaining counter, Edit Event form, side-by-side Active Vendor Roster (categorized by status with **Sever Connection** de-assignment tools) + Ad-Hoc Vendor Assignment panel.
- **Booking Approvals:** Master data grid to monitor incoming bookings. Features inline action hooks hitting `/approve` or `/reject` which triggers backend capacity refund logic. Rejected bookings display a toast notification indicating freed seats.
- **Support Agent Portal:** Console view to digest `SupportTickets` open issues. Clicking a row expands to reveal the full customer query description.

### Phase C: Customer Portal
**(Role: `ROLE_CUSTOMER` - End-User Funnels)**
- **Venue Browsing:** Highly visual card layouts utilizing Tailwind images highlighting available Venues.
- **Event Detail View:** Each event card links to a dedicated `/book/events/:eventId` screen showing full event info, a live **Seats Remaining** counter with a `🔥 Seats filling fast!` urgency tag when occupancy exceeds 75%, and a direct Ticket booking form — all without exposing admin-only data (no vendor management panels, no edit controls).
- **The Booking Flow:** Core pathway clicking into a Venue → Selecting an Event → Defining physical Ticket quantities → Deriving the dynamic cost based on `venue.pricePerSeat * ticketQuantity` → Sending `POST /bookings`.
- **Dashboard:** Management space to view owned bookings and execute `PUT /cancel` to refund slots.
- **Support Center (Microservice):** Form view to dispatch and track `SupportTickets`. Requests are routed to the **Node.js service** for improved scalability and isolation.
- **Profile Maintenance:** Secured `/users/{id}` identity portal utilizing our strict backend IDOR protections.
***

## 5. API Contracts & Payloads (Axios Integration Guide)
All requests automatically include `Authorization: Bearer <token>` via Axios interceptors, except for `/auth/*`.

### Auth & User APIs (`utils/api/authApi.js`, `userApi.js`)
- `POST /api/v1/auth/login` → Body: `{ username, password }`
- `POST /api/v1/auth/register` → Body: `{ username, email, password, role }` *(Role is `ROLE_CUSTOMER` or `ROLE_ADMIN`)*
- `GET /api/v1/users/{id}` → Retrieves profile.
- `PUT /api/v1/users/{id}` → Body: `{ username, email }`

### Venue APIs (`utils/api/venueApi.js`) - Admin Only
- `GET /api/v1/venues?page={page}&size={size}` → Returns `Page<VenueDTO>`
- `POST /api/v1/venues` → Body: `{ name, address, capacity, amenities }`
- `PUT /api/v1/venues/{id}` → Body: `{ name, address, capacity, amenities }`
- `DELETE /api/v1/venues/{id}` → Soft deletes venue.

### Event APIs (`utils/api/eventApi.js`)
- `GET /api/v1/events?page={page}&size={size}` → Returns `Page<EventDTO>`
- `GET /api/v1/events/{id}` → Returns enhanced `EventDTO` including `soldTickets` & `venueCapacity` for seats-remaining computation.
- `GET /api/v1/events/venue/{venueId}?page={page}&size={size}`
- `POST /api/v1/events` *(Admin)* → Body: `{ title, description, eventDate, venueId, organizerId }`
- `PUT /api/v1/events/{id}` *(Admin)* → Body: `{ title, description, eventDate }`
- `POST /api/v1/events/{eventId}/vendors/{vendorId}` *(Admin)* → Assigns a vendor; dispatched in parallel for multi-vendor bulk assignment.
- `DELETE /api/v1/events/{eventId}/vendors/{vendorId}` *(Admin)* → Manually de-assigns a vendor from an event.
- `GET /api/v1/events/{eventId}/vendors` *(Admin)* → Returns list of `EventVendorDTO` with status field.

### Booking APIs (`utils/api/bookingApi.js`)
- `POST /api/v1/bookings` *(Customer)* → Body: `{ userId, eventId, tickets: [{ ticketType, price, quantity }] }`
- `GET /api/v1/bookings/user/{userId}` *(Customer)* → Returns list of user's bookings.
- `GET /api/v1/bookings/admin/bookings` *(Admin)* → Returns all system bookings.
- `PUT /api/v1/bookings/{id}/cancel` *(Customer)* → Updates status to `CANCELLED` and safely refunds capacity.
- `PUT /api/v1/bookings/admin/bookings/{id}/approve` *(Admin)* → Updates status to `APPROVED`.
- `PUT /api/v1/bookings/admin/bookings/{id}/reject` *(Admin)* → Updates status to `REJECTED` and safely refunds capacity.

### Support APIs (`utils/api/supportApi.js`)
- `POST /api/v1/tickets` *(Customer)* → Body: `{ customerId, subject, description }`
- `GET /api/v1/tickets/customer/{customerId}` *(Customer)*
- `PUT /api/v1/tickets/{id}?status={status}` *(Admin)* → URI Param status: `CLOSED`, `IN_PROGRESS`