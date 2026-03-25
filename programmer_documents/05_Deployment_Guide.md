# EventZen — Deployment Guide

## 1. Overview

The EventZen platform is fully containerized using **Docker** and orchestrated via **Docker Compose**. The deployment configuration provisions four interdependent services — a MySQL database, a Spring Boot core API, a Node.js support microservice, and an Nginx-hosted React frontend — within an isolated bridge network. The entire environment can be initialized, built, and launched with a single command.

---

## 2. Prerequisites

Ensure the following software is installed and operational on the target deployment machine before proceeding.

| Requirement              | Minimum Version | Verification Command          |
| :----------------------- | :-------------- | :---------------------------- |
| **Docker Engine**        | 20.10+          | `docker --version`            |
| **Docker Compose**       | v2.0+           | `docker compose version`      |
| **Available Ports**      | —               | Ports `80`, `3000`, `3307`, and `8080` must be unoccupied |
| **Disk Space**           | ~2 GB           | For container images and MySQL data volume |

> **Note:** Docker Desktop (Windows/macOS) includes Docker Compose by default. Linux installations may require a separate Compose plugin installation.

---

## 3. Service Topology

The `docker-compose.yml` file defines the following four-container architecture:

| Container Name           | Service Role                            | Internal Port | Exposed Port | Build Context                |
| :----------------------- | :-------------------------------------- | :------------ | :----------- | :--------------------------- |
| `eventzen-db`            | MySQL 8.0 relational database           | 3306          | 3307         | Official `mysql:8.0` image   |
| `eventzen-spring-core`   | Spring Boot core API server             | 8080          | 8080         | `./eventzen`                 |
| `eventzen-node-support`  | Node.js/Express support microservice    | 3000          | 3000         | `./eventzen-support-service` |
| `eventzen-react-ui`      | React SPA served via Nginx              | 80            | 80           | `./eventzenFrontend`         |

### Dependency Chain

Services start in the following order, enforced by Docker Compose's `depends_on` declarations and health checks:

```
eventzen-db (MySQL)
    ├── eventzen-spring-core   (starts after DB health check passes)
    ├── eventzen-node-support  (starts after DB health check passes)
    └── eventzen-react-ui      (starts after both backend services are running)
```

The database container includes a health check (`mysqladmin ping`) that prevents dependent services from starting until MySQL is fully accepting connections.

---

## 4. Startup Procedure

### 4.1 Clone the Repository

```bash
git clone https://github.com/rohitpadile/eventzen-capstone.git
cd eventzen-capstone
```

### 4.2 Build and Launch All Services

From the project root directory (where `docker-compose.yml` is located), execute:

```bash
docker-compose up --build
```

This command performs the following operations:
1. Pulls the `mysql:8.0` base image (if not already cached).
2. Builds Docker images for the Spring Boot, Node.js, and React services from their respective Dockerfiles.
3. Creates the `eventzen-network` bridge network.
4. Provisions a persistent `mysql-data` volume for database storage.
5. Starts all four containers in dependency order.

> **Note:** The initial build may take 3–5 minutes depending on network speed and system resources. Subsequent launches (without the `--build` flag) will start significantly faster.

### 4.3 Detached Mode (Optional)

To run the services in the background:

```bash
docker-compose up --build -d
```

Monitor logs for a specific service:

```bash
docker-compose logs -f spring-backend
docker-compose logs -f node-service
```

---

## 5. Network Configuration

All containers communicate over a shared Docker bridge network (`eventzen-network`). Internal service-to-service connections use Docker's DNS resolution with container service names.

### Internal Connectivity

| From                    | To                 | Connection String                                                      |
| :---------------------- | :----------------- | :--------------------------------------------------------------------- |
| Spring Boot Core        | MySQL              | `jdbc:mysql://mysql-db:3306/eventzen_db?useSSL=false&allowPublicKeyRetrieval=true` |
| Node.js Support Service | MySQL              | `mysql-db:3306` (configured via environment variables)                  |
| React Frontend          | Spring Boot Core   | `http://localhost:8080` (from the browser, via exposed host port)       |
| React Frontend          | Node.js Support    | `http://localhost:3000` (from the browser, via exposed host port)       |

### Environment Variables

Critical configuration is injected into each container via the `environment` block in `docker-compose.yml`:

| Variable                      | Service       | Purpose                                |
| :---------------------------- | :------------ | :------------------------------------- |
| `MYSQL_ROOT_PASSWORD`         | MySQL         | Database root credential               |
| `MYSQL_DATABASE`              | MySQL         | Auto-created database name             |
| `SPRING_DATASOURCE_URL`      | Spring Boot   | JDBC connection string                 |
| `SPRING_DATASOURCE_USERNAME` | Spring Boot   | Database authentication                |
| `SPRING_DATASOURCE_PASSWORD` | Spring Boot   | Database authentication                |
| `DB_HOST`, `DB_PORT`, `DB_USER`, `DB_PASS`, `DB_NAME` | Node.js | Database connection parameters |
| `JWT_SECRET`                  | Node.js       | Shared secret for cross-service JWT verification |

---

## 6. Post-Deployment Verification

After all containers report healthy status, verify the deployment by accessing the following endpoints:

| Verification Step                  | URL / Command                          | Expected Result                      |
| :--------------------------------- | :------------------------------------- | :----------------------------------- |
| **Frontend loads**                 | `http://localhost`                      | EventZen landing page renders        |
| **Spring Boot API responds**       | `http://localhost:8080/api/v1/venues`   | JSON response (empty array or venue data) |
| **Node.js API responds**           | `http://localhost:3000/api/v1/tickets/customer/1` | JSON response or 401 Unauthorized |
| **Database is accessible**         | `docker exec -it eventzen-db mysql -uroot -proot -e "SHOW DATABASES;"` | `eventzen_db` listed |
| **Container health**               | `docker-compose ps`                    | All four containers show `Up` status |

### Default Admin Account

On first startup, the Spring Boot `DataInitializer` component automatically seeds a default administrator account if no admin user exists in the database. Use this account to access admin-only functionality immediately after deployment.

---

## 7. Data Persistence

The MySQL data directory is mounted to a named Docker volume (`mysql-data`). This ensures that database contents persist across container restarts and rebuilds. To perform a full data reset:

```bash
docker-compose down -v
```

> **Caution:** The `-v` flag removes all named volumes, permanently deleting all database data.

---

## 8. Shutdown Procedure

### Graceful Shutdown (Preserve Data)

```bash
docker-compose down
```

This stops and removes all containers and the bridge network, but preserves the database volume.

### Full Teardown (Remove Data)

```bash
docker-compose down -v --rmi all
```

This removes all containers, networks, volumes, and locally built images.

---

## 9. Troubleshooting

| Symptom                                   | Likely Cause                        | Resolution                                          |
| :---------------------------------------- | :---------------------------------- | :-------------------------------------------------- |
| Spring Boot fails to start                | MySQL not yet ready                 | The health check should prevent this. If persistent, increase `start_period` in `docker-compose.yml`. |
| Port conflict on startup                  | Host port already in use            | Stop the conflicting process or modify the port mapping in `docker-compose.yml`. |
| Node.js cannot connect to database        | Incorrect environment variables     | Verify `DB_HOST`, `DB_PORT`, `DB_USER`, `DB_PASS` in `docker-compose.yml`. |
| Frontend shows network errors             | Backend services not yet ready      | Wait for all services to initialize. Check logs with `docker-compose logs`. |
| `ECONNREFUSED` from browser              | API URL misconfiguration            | Ensure the React build uses `http://localhost:8080` and `http://localhost:3000` for API targets. |
