# Eventzen - Docker Deployment Guide

This document provides instructions for spinning up the complete Eventzen microservices ecosystem using Docker.

## 📦 Prerequisites
- **Docker Desktop** installed and running.
- **Docker Compose** installed (included with Docker Desktop).

## 🚀 One-Command Execution
From the project root directory, run the following command in your terminal:

```bash
docker-compose up --build
```

## 🏗️ Service Architecture
The `docker-compose.yml` file orchestrates 4 synchronized containers:

| Container Name | Role | Port |
| :--- | :--- | :--- |
| `eventzen-db` | MySQL 8.0 Database | 3306 |
| `eventzen-spring-core` | Spring Boot API (Auth & Logistics) | 8080 |
| `eventzen-node-support` | Node.js Microservice (Ticketing) | 3000 |
| `eventzen-react-ui` | React Frontend (Nginx Server) | 80 |

## 🛠️ Internal Connectivity
- **Spring Boot** connects to the database via `jdbc:mysql://mysql-db:3306/eventzen_db`.
- **Node.js** connects to the database via `mysql-db:3306`.
- **React Frontend** communicates with the respective backends via `localhost:8080` (Spring) and `localhost:3000` (Node) when accessed via your browser.

## 🛑 Shutdown
To stop and remove the containers, use:
```bash
docker-compose down
```
