# Eventzen - Final Submission Summary

This document consolidates all technical endpoints, repository details, and system specifications for the final project submission.

## 🔗 Project Fundamentals
- **GitHub Repository**: [\[eventzen-capstone\]](https://github.com/rohitpadile/eventzen-capstone)
- **Deployment Status**: Fully Dockerized (Single-command deployment)
- **Hybrid Architecture**: Spring Boot (Core) + Node.js (Support Microservice)

---

## 📡 API Endpoints (Admin & Customer)

### 🔴 Spring Boot Core (Port 8080)
- `POST /api/v1/auth/login` - Entry point for principal authentication.
- `POST /api/v1/auth/register` - User onboarding with role assignment.
- `GET /api/v1/venues` - Paginated grid of venue logistics.
- `POST /api/v1/events` - (Admin Only) Orchestration of new event instances.
- `POST /api/v1/bookings` - Transactional seat reservation funnel.
- `PUT /admin/bookings/{id}/approve` - (Admin Only) Inventory-decrementing trigger.

### 🟢 Node.js Microservice (Port 3000)
- `POST /api/v1/tickets` - Support ticket creation.
- `GET /api/v1/tickets/customer/{id}` - Historical log retrieval.
- `PUT /api/v1/tickets/{id}` - (Admin Only) Status management.
- `POST /api/v1/tickets/{id}/restore` - (Admin Only) Accidental deletion recovery.

---

## 🛠️ Data Integrity & Security
- **JWT Shared Auth**: Cross-service identity propagation using shared-secret verification.
- **Optimistic Locking**: Database-level race condition prevention for ticket sales.
- **IDOR Protection**: Spring Security expressions ensuring users can only reach their own data.
- **Soft Deletion**: Hybrid `is_deleted` tracking across both SQL implementations (Hibernate & Sequelize).

## 📄 Associated Documents in Folder
1. `backend_business_logic_summary.md` - Technical deep-dive.
2. `frontend_architecture_plan.md` - React structure and flows.
3. `ER_diagrams.md` - Database schema visualization.
4. `USER_FLOWS.md` - Interaction journey maps.
5. `DOCKER_DEPLOYMENT.md` - Setup guide.
6. `WIREFRAMES.md` - Structural blueprints.
7. `SCREENSHOT_CHECKLIST.md` - Visual submission guide.
