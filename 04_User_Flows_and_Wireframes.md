# EventZen — User Flows and Wireframes

## 1. Introduction

This document defines the finalized UI/UX strategy for the EventZen platform. The application serves two distinct user personas through dedicated, role-secured portals:

- **Customer Portal** — End-users who discover venues, book events, manage their reservations, and submit support inquiries.
- **Admin Portal** — Staff members who manage venue infrastructure, vendor relationships, event orchestration, booking approvals, and support ticket resolution.

Each portal features purpose-built navigation, interface layouts, and interaction patterns tailored to its respective audience.

---

## 2. Customer Portal — User Flows

The Customer experience is designed around four primary operational pathways.

### 2.1 The Booking Funnel (Primary Flow)

This is the core revenue-generating pathway. Customers navigate from venue discovery through event selection to ticket purchase.

```
Landing Page  →  Login / Register  →  Customer Dashboard
       ↓
  Browse Venues  →  Select Venue  →  View Events at Venue
       ↓
  Select Event  →  Event Detail Page (Seats Remaining, Pricing)
       ↓
  Set Ticket Quantity  →  Review Dynamic Total (price_per_seat × quantity)
       ↓
  Submit Booking Request  →  Status: PENDING (Awaiting Admin Approval)
```

**Key UX Decisions:**
- Dynamic total cost is calculated in real-time based on the venue's `pricePerSeat` and the selected ticket quantity.
- A contextual urgency indicator ("Seats Filling Fast") appears automatically when event occupancy exceeds 75%, driving conversion through scarcity signaling.
- A live "Seats Remaining" counter is prominently displayed on the Event Detail page, computed from `venue.capacity − event.soldTickets`.

### 2.2 Booking Management

Customers manage their existing reservations through the dashboard.

```
Customer Dashboard  →  My Bookings (List View)  →  Select Booking
       ↓
  View Booking Status (PENDING / APPROVED / REJECTED / CANCELLED)
       ↓
  Cancel Booking  →  Seat Inventory Automatically Restored
```

**Business Rule:** Seat capacity is preserved during the `PENDING` state and only decremented upon admin approval. Cancellation of an approved booking automatically triggers capacity restoration.

### 2.3 Support Center (Microservice-Backed)

Customer support requests are routed to the dedicated Node.js microservice for isolated processing.

```
Help Desk  →  Submit New Ticket (Subject + Description)
       ↓
  View Ticket History  →  Track Status (OPEN → IN_PROGRESS → RESOLVED)
```

### 2.4 Profile Management

```
Profile Page  →  View/Update Identity Information  →  Save Changes
```

**Security:** All profile operations are protected by backend IDOR enforcement, ensuring users can only access and modify their own records.

---

## 3. Admin Portal — User Flows

The Admin experience is designed around five operational domains, accessible from a persistent sidebar navigation.

### 3.1 Venue Logistics

```
Venue Management  →  Paginated Venue Grid
       ↓
  Add New Venue (Name, Address, Capacity, Price Per Seat, Amenities)
       ↓
  Edit Existing Venue  |  Soft-Delete Venue  |  Restore Deleted Venue
```

### 3.2 Vendor Relations

```
Vendor Management  →  Vendor Registry Table
       ↓
  Onboard New Vendor (Name, Email, Service Type)
       ↓
  Monitor Assignment Status (Automated PENDING → ACCEPTED/REJECTED transitions)
```

**Note:** Vendors with active event assignments cannot be deleted. Administrators must de-assign them from all events before deletion is permitted.

### 3.3 Event Orchestration

```
Event Management  →  Paginated Event Grid
       ↓
  Create Event (Title, Description, Date, Venue Selection, Organizer)
       ↓
  Event Detail Matrix:
    ├── Live Seats Remaining Counter
    ├── Edit Event Details
    ├── Active Vendor Roster (Categorized by Assignment Status)
    ├── De-assign Vendor ("Sever Connection")
    └── Assign Additional Vendor (Ad-Hoc Assignment Panel)
```

### 3.4 Booking Approvals

```
Booking Queue  →  Filter by PENDING Status  →  Review Capacity Impact
       ↓
  APPROVE  →  Seat Inventory Decremented, Booking Confirmed
       ↓
  REJECT  →  Seat Inventory Released, Customer Notified
```

### 3.5 Support Agent Console

```
Support Dashboard  →  Open Tickets Queue
       ↓
  Select Ticket  →  View Full Customer Description
       ↓
  Update Status (IN_PROGRESS / RESOLVED)  |  Restore Soft-Deleted Ticket
```

---

## 4. Wireframe Specifications

The following wireframe descriptions define the structural layout and interaction patterns for each core interface.

### 4.1 Global Navigation Design

#### Admin Layout
| Element          | Specification                                                         |
| :--------------- | :-------------------------------------------------------------------- |
| **Sidebar**      | Fixed left panel · `EventZen` logo · Vertical menu: Dashboard, Venues, Events, Vendors, Bookings, Support |
| **Top Header**   | Horizontal bar · "Admin Panel" label · Logout action                  |
| **Content Area** | Fluid grid · 20px internal padding · Dynamic content rendering        |

#### Customer Layout
| Element        | Specification                                                      |
| :------------- | :----------------------------------------------------------------- |
| **Top Navbar** | Horizontal bar · `EventZen` branding · My Bookings, Support, Profile, Logout |
| **Main Body**  | Centered container · max-width 1280px · Subtle fade-in page transitions |

### 4.2 Landing Page

| Element            | Specification                                                           |
| :----------------- | :---------------------------------------------------------------------- |
| **Header Bar**     | `EventZen` logo (top-left) · Sign In / Register buttons (top-right)    |
| **Hero Section**   | Centered bold typography · "Explore Events" call-to-action button       |
| **Auth Modal**     | Center-screen overlay · 50/50 split layout (visual branding + form inputs) |

### 4.3 Event Detail Page

| Element              | Specification                                                       |
| :------------------- | :------------------------------------------------------------------ |
| **Page Header**      | Event title (H1) · "Seats Remaining" badge                         |
| **Left Column**      | Event description · Date/time · Venue location details              |
| **Right Column**     | Sticky booking card · Ticket quantity input · Dynamic total price calculation |
| **Urgency Indicator**| Pulsing "Few Seats Left" badge near the booking CTA (triggered at >75% occupancy) |

### 4.4 Admin Booking Queue

| Element              | Specification                                                       |
| :------------------- | :------------------------------------------------------------------ |
| **Page Heading**     | "Booking Approval Queue"                                            |
| **Table Structure**  | Multi-column: User, Event, Ticket Count, Status, Actions            |
| **Action Controls**  | "Approve" (green outline) · "Reject" (red outline) · Visible only for `PENDING` entries |

### 4.5 Support Help Desk

| Element            | Specification                                                        |
| :----------------- | :------------------------------------------------------------------- |
| **Left Panel**     | Vertical form: Subject field + Message textarea + Submit button      |
| **Right Panel**    | Scrollable ticket history · Individual ticket cards                  |
| **Status Badges**  | Color-coded: `OPEN` (red) · `IN_PROGRESS` (amber) · `RESOLVED` (green) |

---

## 5. Responsive Design Considerations

- All grid layouts adapt from multi-column to single-column stacking on viewports below 768px.
- The Admin sidebar collapses to a hamburger menu on tablet-sized screens.
- Interactive elements (buttons, form inputs, table rows) maintain minimum touch target sizes of 44×44px for mobile accessibility.
- Page transitions use subtle `fade-in` animations to provide visual continuity without impacting perceived performance.
