# Eventzen - Entity Relationship Diagrams (ERD)

This document visualizes the core database relational structure of the Eventzen application using Mermaid.js.

```mermaid
erDiagram
    %% Core Relationships with 1:M notation
    USER ||--o{ EVENT : "1:M organizes"
    USER ||--o{ REGISTRATION : "1:M books"
    USER ||--o{ SUPPORT_TICKET : "1:M submits"
    
    VENUE ||--o{ EVENT : "1:M hosts"
    
    EVENT ||--o{ REGISTRATION : "1:M contains"
    EVENT ||--o{ EVENT_VENDOR : "1:M assigns"
    
    VENDOR ||--o{ EVENT_VENDOR : "1:M servicing"

    %% Table Definitions
    USER {
        Long id PK
        String username
        String email
        String role "ADMIN, CUSTOMER"
        Boolean is_deleted
    }

    VENUE {
        Long id PK
        String name
        String address
        Integer capacity
        Decimal price_per_seat
    }

    EVENT {
        Long id PK
        String title
        DateTime event_date
        Integer sold_tickets
        Long venue_id FK
        Long organizer_id FK
    }

    VENDOR {
        Long id PK
        String name "Business Name"
        String contact_email
        String service_type "Catering, Lights, etc."
    }

    REGISTRATION {
        Long id PK
        DateTime registration_date
        String status "APPROVED, PENDING, REJECTED"
        Long user_id FK
        Long event_id FK
    }

    SUPPORT_TICKET {
        Long id PK
        String subject
        String description
        String status "OPEN, IN_PROGRESS, RESOLVED"
        Long customer_id FK
        Boolean is_deleted
    }
    
    EVENT_VENDOR {
        Long id PK
        String status "ACCEPTED, PENDING, REJECTED"
        Long event_id FK
        Long vendor_id FK
    }
```

### Architectural Breakdown
- **User Roles**: The system differentiates between `ADMIN` (organizers) and `CUSTOMER` (bookers).
- **Logistics Framework**: The `EVENT_VENDOR` table acts as a join entity between `EVENT` and `VENDOR`, allowing a many-to-many relationship with a status tracker for each assignment.
- **Venue Hosting**: Each `EVENT` is bound to exactly one `VENUE`, while a `VENUE` can host many events.
- **Booking Flow**: `REGISTRATION` links `USER` to `EVENT` for ticket management.
