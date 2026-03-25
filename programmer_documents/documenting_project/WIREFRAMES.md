# Eventzen - Structural Wireframe Descriptions

Since wireframes represent the "blueprint" of the application, use these structural descriptions to satisfy the design documentation requirements.

---

## 🏗️ 1. Global Navigation (Admin vs Customer)

### [Admin Layout]
- **Sidebar (Fixed Left)**: Vertical menu with `EventZen` logo, `Dashboard`, `Venues`, `Events`, `Vendors`, `Bookings`, `Support`.
- **Top Header**: Simple bar showing `Admin Panel` and a `Logout` button.
- **Content Area**: Dynamic white-space with 20px padding (Fluid grid).

### [Customer Layout]
- **Top Navbar**: Horizontal branding, `My Bookings`, `Support`, `Profile`, and `Logout`.
- **Main Body**: Centered container (max-width: 1280px) with subtle fade-in animations.

---

## 🖼️ 2. Core Screen Layouts

### A. Landing Page (The Entry Hub)
- **Top Left**: `EventZen` clickable logo.
- **Top Right**: `Sign In` / `Register` buttons.
- **Hero Section**: Centered bold typography with a "Explore Events" call-to-action button.
- **Auth Modal**: Center-screen overlay with a 50/50 split (Visual side + Form side).

### B. Event Detail Page (High Stakes UI)
- **Header**: Large Event Title (H1) + "Seats Remaining" badge.
- **Grid (2 Columns)**:
  - **Left**: Detailed description, date/time, and venue location.
  - **Right**: Sticky Booking Card with Quantity input and total price calculation.
- **Urgency Hook**: Pulsing "Few Seats Left" tag near the CTA.

### C. Admin Booking Queue (The Gatekeeper)
- **Heading**: "Booking Approval Queue".
- **Table Structure**: Multi-column list (User, Event, Status, Actions).
- **Control Buttons**: `Approve` (Green outline) and `Reject` (Red outline) appearing only for PENDING items.

### D. Support Help Desk (Microservice Portal)
- **Split Layout**:
  - **Left**: Vertical Form (Subject + Message).
  - **Right**: Scrollable vertical list of individual ticket cards showing history.
- **Card States**: Color-coded badges for `OPEN` (Red) and `RESOLVED` (Green).
