# 🌾 Registrul Agricol — Agricultural Register

> **Team project developed during a software development internship.**

A full-stack, multi-tenant web platform for digitizing and managing the Romanian **Agricultural Register** (*Registrul Agricol*) — the foundational public administrative document maintained by Romanian local authorities (town halls / *primării*) to record land, livestock, buildings, machinery, and citizen data across all territorial-administrative units (UATs).

---

## 📋 Table of Contents

- [About the Project](#about-the-project)
- [Key Features](#key-features)
- [Tech Stack](#tech-stack)
- [Architecture](#architecture)
- [Project Structure](#project-structure)
- [Getting Started](#getting-started)
- [Default Accounts](#default-accounts)
- [API Overview](#api-overview)
- [Team](#team)

---

## About the Project

The **Registrul Agricol** (Agricultural Register) is a legally mandated administrative document (*OG nr. 28/2008*, *HG nr. 1627/2024*) that every Romanian municipality must maintain. It records:

- **Households** (*gospodării*) and their members
- **Land** (*terenuri* and *parcele*) — arable land, pastures, orchards, forests, etc.
- **Buildings** (*clădiri*) and agricultural annexes
- **Livestock** (*animale*) — by species, category, and individual identification (crotal/ear tag)
- **Agricultural machinery** (*utilaje*)
- **Crops** (*culturi*) grown on each land plot
- **Contracts** (*contracte*) — lease (*arendă*), concession, loan for use (*comodat*)
- **Citizen requests** (*cereri*) — certificates, property extracts, etc.

This application replaces paper-based or spreadsheet-driven workflows with a **modern, secure, and role-based digital platform** accessible to both administrative staff and citizens.

---

## Key Features

### 🏛️ Multi-Tenancy (Schema-per-Tenant)
- Each administrative region (county cluster / UAT group) is isolated in its own **PostgreSQL schema**
- Tenant context is resolved per-request via the `X-Tenant-ID` HTTP header
- Super Admins can **impersonate** any UAT to manage its data without switching applications

### 👥 Role-Based Access Control (RBAC)
| Role | Description |
|---|---|
| `ROLE_SUPER_ADMIN` | Creates and manages tenants, UATs, and admins; can impersonate any UAT |
| `ROLE_ADMIN` | Manages users and all agricultural data within their tenant |
| `ROLE_USER` | Standard registry operator with CRUD access to agricultural entities |

### 🗺️ GIS / Interactive Map
- Land plots (*parcele*) store PostGIS `Polygon` geometries in **EPSG:3857** projection
- Interactive **Leaflet** map with parcel drawing, editing, and visualization
- Support for **Stereo70** coordinate conversion (Romanian national projection)
- **Google Maps** satellite layer integration for parcel validation

### 📄 Document & PDF Generation
- Automatic generation of official **certificates** (*adeverințe*) as downloadable PDFs
- PDF rendering via **OpenHTMLtoPDF** + Thymeleaf HTML templates
- Excel/CSV export for statistical reporting (DAJ, INS)

### 📧 Email Notifications
- Automatic confirmation emails sent to citizens when a request (*cerere*) is submitted
- SMTP integration via **Mailtrap** (local dev) or any SMTP provider (production)
- Responsive HTML email templates built with Thymeleaf

### 🐄 Animal Management
- Full livestock registry: individual animals + group herds
- Crotal (ear tag) number uniqueness enforced at registry level
- Animal lifecycle events (birth, death, sale, transfer)
- Compliance with **SNIIA** (National Animal Identification System — ANSVSA)

### 📑 Contract Management
- Lease, concession, and loan-for-use contracts linked to specific land parcels
- Automated scheduler that detects and flags **expiring contracts**
- Real-time **WebSocket notifications** pushed to logged-in admins

### 🔒 Security
- **JWT** authentication (stateless, Bearer token)
- **AES-256** column-level encryption for sensitive personal identifiers (CNP, CUI)
- **Hibernate Envers** for full audit history of entity changes
- GDPR-conscious design: personal data access logging

### 👨‍💼 Citizen Portal (*Portal Cetățeni*)
- Separate Angular application for public-facing access
- Citizens can submit requests (*cereri*) for official certificates
- Automatic email confirmation with unique request code upon submission

### 📊 Statistics & Reporting
- Aggregate reports on crops, livestock, and machinery per UAT
- Export-ready data for **DAJ** (County Agricultural Directorate) and **INS** (National Statistics Institute)

---

## Tech Stack

### Backend
| Technology | Purpose |
|---|---|
| **Java 21** | Core language |
| **Spring Boot 3.3** | Application framework |
| **Spring Security + JWT** | Authentication & authorization |
| **Spring Data JPA / Hibernate** | ORM and database access |
| **Hibernate Spatial** | PostGIS geometry types |
| **Hibernate Envers** | Entity audit history |
| **Flyway** | Database migrations |
| **Apache Kafka** | Async messaging and event streaming |
| **Spring WebSocket** | Real-time push notifications |
| **Thymeleaf** | HTML template engine (emails & PDFs) |
| **OpenHTMLtoPDF** | PDF generation |
| **Apache POI** | Excel generation |
| **ModelMapper** | DTO mapping |
| **Lombok** | Boilerplate reduction |
| **Proj4J** | Coordinate system conversion (Stereo70 ↔ WGS84) |

### Database
| Technology | Purpose |
|---|---|
| **PostgreSQL 15** | Primary relational database |
| **PostGIS 3.3** | Spatial/GIS extension for polygon geometries |

### Frontend (Admin Panel)
| Technology | Purpose |
|---|---|
| **Angular 18** (Standalone Components) | Admin SPA |
| **Leaflet.js** | Interactive map |
| **Google Maps API** | Satellite layer |

### Citizen Portal
| Technology | Purpose |
|---|---|
| **Angular** | Citizen-facing SPA |

### Infrastructure
| Technology | Purpose |
|---|---|
| **Docker & Docker Compose** | Containerized deployment |
| **Nginx** | Frontend reverse proxy |
| **Apache Kafka + Zookeeper** | Event streaming |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        Docker Network                        │
│                                                              │
│  ┌──────────────┐  ┌───────────────┐  ┌────────────────┐   │
│  │   Frontend   │  │    Portal     │  │    Backend     │   │
│  │  (Angular)   │  │  Cetățeni     │  │ (Spring Boot)  │   │
│  │  Port: 80    │  │  (Angular)    │  │   Port: 8080   │   │
│  └──────┬───────┘  │  Port: 4202   │  └───────┬────────┘   │
│         │          └──────┬────────┘          │            │
│         └─────────────────┴───────────────────┘            │
│                                    │                        │
│                     ┌──────────────┼──────────┐            │
│                     │              │           │            │
│              ┌──────┴──────┐  ┌───┴───┐  ┌────┴──────┐    │
│              │  PostgreSQL  │  │ Kafka │  │ Zookeeper │    │
│              │   + PostGIS  │  │ 9092  │  │   2181    │    │
│              │  Port: 5433  │  └───────┘  └───────────┘    │
│              └─────────────┘                                │
└─────────────────────────────────────────────────────────────┘
```

## Project Structure

```
Registru-Agricol/
├── src/
│   ├── backend/                  # Spring Boot API
│   │   └── src/main/java/com/multitenant/
│   │       ├── controller/       # REST controllers
│   │       ├── service/          # Business logic
│   │       ├── entity/           # JPA entities
│   │       ├── dto/              # Data Transfer Objects
│   │       ├── repository/       # Spring Data repositories
│   │       ├── config/           # Security, Kafka, ModelMapper, etc.
│   │       └── tenant/           # Multi-tenancy infrastructure
│   │
│   ├── frontend/                 # Angular admin panel
│   │   └── src/app/
│   │       ├── pages/            # Feature pages
│   │       ├── services/         # API clients
│   │       └── components/       # Reusable UI components
│   │
│   └── portal-cetateni/          # Angular citizen portal
│       └── src/app/
│           ├── pages/            # Citizen-facing pages
│           └── services/         # API clients
│
├── Docs/                         # Technical documentation
├── docker-compose.yml            # Full stack orchestration
└── uploads/                      # Uploaded documents storage
```

---

## Getting Started

### Prerequisites

Make sure you have the following installed:

- [Git](https://git-scm.com/)
- [Docker Desktop](https://www.docker.com/products/docker-desktop/) (includes Docker Compose)
- [Node.js & npm](https://nodejs.org/) *(optional — only for local frontend development)*
- [Angular CLI](https://angular.io/cli): `npm install -g @angular/cli` *(optional)*

### 1. Clone the repository

```bash
git clone https://github.com/<your-username>/Registru-Agricol.git
cd Registru-Agricol
```

### 2. Configure environment variables

Copy the example env file and fill in the required values:

```bash
cp Docs/.env.example .env
```

Required variables:

```env
GOOGLE_MAPS_API_KEY=your_google_maps_api_key
```

### 3. Start the full stack with Docker

```bash
docker compose up --build
```

This will start:

| Service | URL |
|---|---|
| **Backend API** | http://localhost:8080 |
| **Admin Frontend** | http://localhost:80 |
| **Citizen Portal** | http://localhost:4202 |
| **PostgreSQL** | localhost:5433 |
| **Kafka** | localhost:9092 |

Run in background:

```bash
docker compose up --build -d
```

Stop all containers:

```bash
docker compose down
```

### 4. (Optional) Run the frontend locally for development

```bash
cd src/frontend
npm install
ng serve
# Available at http://localhost:4200
```

```bash
cd src/portal-cetateni
npm install
ng serve --port 4201
# Available at http://localhost:4201
```

---

## Default Accounts

After startup, the following default account is available:

| Username | Password | Role |
|---|---|---|
| `superadmin` | `superadmin` | SUPER_ADMIN |

> A **SUPER_ADMIN** can create tenants, assign UATs, and create admin users via the UI or API.

---


## Team

This project was developed as a **team internship project** during a software development placement program. Each team member contributed to different parts of the system — including backend domain modules, frontend features, GIT integration, security implementation, contract management, and the citizen-facing portal.

