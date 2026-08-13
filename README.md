# 🐳 Containerizing a Three-Tier Student–Teacher Application

A hands-on Docker project demonstrating how to containerize and run a three-tier web application consisting of a **React frontend, Node.js backend, and MySQL database**.

The project was initially run using individual Dockerfiles and manually created containers, then progressively improved using **multi-stage Docker builds** and finally automated using **Docker Compose**.

> **Note:** The base Student–Teacher application was provided as part of an instructor-led project. My primary work in this project focused on containerization, Docker configuration, image optimization, networking, persistence, Nginx configuration, Docker Compose, and troubleshooting.

---

## 📌 Table of Contents

- [Project Overview](#-project-overview)
- [Architecture](#️-architecture)
- [Technology Stack](#️-technology-stack)
- [Project Structure](#-project-structure)
- [Application Components](#-application-components)
- [Docker Implementation](#-docker-implementation)
- [Frontend Dockerization](#-frontend-dockerization)
- [Backend Dockerization](#️-backend-dockerization)
- [MySQL Database Container](#️-mysql-database-container)
- [Multi-Stage Builds](#-frontend-multi-stage-build)
- [Nginx Configuration](#-nginx-configuration)
- [Docker Networking](#-docker-networking)
- [Persistent Storage](#-persistent-storage)
- [Docker Compose](#-docker-compose)
- [Environment Variables and Build Arguments](#-environment-variables-and-build-arguments)
- [Healthcheck and Service Dependencies](#-healthcheck-and-service-dependencies)
- [Running the Project](#️-running-the-project)
- [Testing](#-testing-the-application)
- [Image Optimization](#-image-optimization)
- [Problems Encountered and Solutions](#-problems-encountered-and-solutions)
- [Docker Commands Used](#-docker-commands-used)
- [Security and Production Considerations](#-security-and-production-considerations)
- [What I Learned](#-what-i-learned)
- [Project Evolution](#-project-evolution)
- [Future Improvements](#-future-improvements)
- [Screenshots](#-screenshots)
- [Key Takeaways](#-key-takeaways)
- [Author](#-author)
- [Acknowledgements](#-acknowledgements)

---

## 📖 Project Overview

This project demonstrates the containerization of a three-tier Student–Teacher web application.

The application consists of:

- **Frontend:** React
- **Backend:** Node.js
- **Database:** MySQL

The application allows users to work with student and teacher records, including operations such as adding and deleting records.

The main goal of the project was not to develop the application itself, but to understand how a multi-tier application can be:

1. Containerized
2. Networked
3. Persisted
4. Optimized using multi-stage builds
5. Configured using Docker Compose
6. Tested end-to-end

The project was built progressively, starting from individual Dockerfiles and manual container execution and eventually moving to a declarative Docker Compose setup.

---

## 🏗️ Architecture

The application follows a three-tier architecture:

```text
                    ┌──────────────────┐
                    │      Browser     │
                    └────────┬─────────┘
                             │
                             │ HTTP :80
                             ▼
                  ┌─────────────────────┐
                  │      FRONTEND       │
                  │   React + Nginx     │
                  │    Container :80    │
                  └──────────┬──────────┘
                             │
                             │ API :3500
                             ▼
                  ┌─────────────────────┐
                  │       BACKEND       │
                  │     Node.js API     │
                  │   Container :3500   │
                  └──────────┬──────────┘
                             │
                             │ MySQL :3306
                             ▼
                  ┌─────────────────────┐
                  │      DATABASE       │
                  │       MySQL         │
                  │   Container :3306   │
                  └─────────────────────┘
```

### Request Flow

```text
User
 ↓
React Frontend
 ↓
Node.js Backend
 ↓
MySQL Database
 ↓
Backend Response
 ↓
React Frontend
 ↓
User
```

---

## 🛠️ Technology Stack

| Technology | Purpose |
|---|---|
| React | Frontend application |
| Node.js | Backend runtime |
| MySQL | Relational database |
| Docker | Application containerization |
| Dockerfile | Image creation |
| Multi-stage Docker Builds | Image optimization |
| Nginx | Production frontend web server |
| Docker Compose | Multi-container orchestration |
| Docker Volumes | Persistent database storage |
| Docker Networking | Container-to-container communication |

---

## 📂 Project Structure

```text
Student-Teacher-Portal-Three-Tier-Application/
│
├── frontend/
│   ├── Dockerfile
│   ├── nginx.conf
│   ├── package.json
│   ├── package-lock.json
│   └── ...
│
├── backend/
│   ├── Dockerfile
│   ├── package.json
│   ├── package-lock.json
│   └── ...
│
├── database/
│   ├── Dockerfile
│   └── ...
│
├── docker-compose.yaml
│
├── .gitignore
│
└── README.md
```

> The exact application source structure may vary depending on the provided Student–Teacher application.

---

## 🧩 Application Components

### 1. Frontend

The frontend is a React application.

**Responsibilities:**
- User interface
- Student management
- Teacher management
- Sending API requests to the backend
- Displaying backend/database data

The production frontend is served using Nginx after being built into static files.

### 2. Backend

The backend is a Node.js application.

**Responsibilities:**
- Provides REST API endpoints
- Handles application logic
- Communicates with MySQL
- Creates/reads/deletes student and teacher records
- Returns data to the frontend

The backend listens on port `3500`.

### 3. Database

MySQL is used as the relational database.

- Database: `school`
- Port: `3306`

The database uses a Docker named volume to persist data beyond the lifecycle of an individual container.

---

## 🐳 Docker Implementation

The project was initially implemented using individual Dockerfiles and manually created containers.

The three services were built and connected in dependency order:

```text
Database
   ↓
Backend
   ↓
Frontend
```

This order is useful because:
- Backend depends on the database.
- Frontend depends on the backend.

---

## 🎨 Frontend Dockerization

### Initial Frontend Dockerfile

The initial frontend image used Node.js to build and serve the React application.

Conceptually:

```dockerfile
FROM node:21-alpine3.17

WORKDIR /app

COPY package.json ./

RUN npm install

COPY . .

ARG REACT_APP_API_BASE_URL
ENV REACT_APP_API_BASE_URL=$REACT_APP_API_BASE_URL

RUN npm run build

EXPOSE 3000

CMD ["npx", "serve", "-s", "build", "-l", "tcp://0.0.0.0:3000"]
```

The application was accessed using `http://localhost` through the Docker port mapping `80:3000`:

```text
Host Port 80
     ↓
Container Port 3000
```

### 🚀 Frontend Multi-Stage Build

The frontend was later converted to a multi-stage Docker build.

```dockerfile
# ---- Stage 1: Build ----
FROM node:21-alpine3.17 AS build

WORKDIR /app

COPY package.json ./

RUN npm install

COPY . .

ARG REACT_APP_API_BASE_URL
ENV REACT_APP_API_BASE_URL=$REACT_APP_API_BASE_URL

RUN npm run build


# ---- Stage 2: Serve ----
FROM nginx:alpine

COPY --from=build /app/build /usr/share/nginx/html

COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

#### Why Multi-Stage?

The Node.js environment is required to build the React application, but it is not required to serve the resulting static files.

**Stage 1:**
```text
Node.js
 ↓
npm install
 ↓
npm run build
 ↓
React static files
```

**Stage 2:**
```text
Nginx
 ↓
React static files
```

This eliminates unnecessary build tools, source code, and `node_modules` from the final runtime image.

---

## 🌐 Nginx Configuration

Nginx is used to serve the production React build.

- The built React files are copied to `/usr/share/nginx/html`
- A custom `nginx.conf` is copied to `/etc/nginx/conf.d/default.conf`

The custom configuration is useful for controlling how Nginx handles requests, particularly for React applications using client-side routing. For example, routes such as `/`, `/student`, and `/teacher` can be handled through the React application's `index.html`.

**Why Nginx?** Instead of running the React development server in the final container, the production build is served as static files by Nginx. This provides a cleaner and significantly smaller runtime image.

---

## ⚙️ Backend Dockerization

The backend was initially containerized using a Node.js base image.

The Dockerfile follows the common dependency-caching pattern:

```dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package.json package-lock.json ./

RUN npm ci --only=production

COPY . .

EXPOSE 3500

CMD ["node", "server.js"]
```

The backend container exposes port `3500`. The application was tested directly using `http://localhost:3500`.

### 🚀 Backend Multi-Stage Build

The backend was also optimized using a multi-stage approach, separating **build/dependency installation** from **runtime**. The final image contains only the dependencies and files required to run the backend.

Additional optimization included:
- Alpine-based Node.js image
- Production-only dependencies
- Removing unnecessary build artifacts
- Running the application as a non-root user

This significantly reduced the final backend image size.

---

## 🗄️ MySQL Database Container

The application uses MySQL 8.

- Database: `school`
- Port: `3306`

A specific MySQL version is used instead of `latest`. Pinning the version helps avoid unexpected behavior when a new MySQL version is released.

---

## 💾 Persistent Storage

A named Docker volume is used for MySQL:

```yaml
volumes:
  - mysql-data:/var/lib/mysql
```

The volume is declared at the bottom of the Compose file:

```yaml
volumes:
  mysql-data:
```

This allows database data to survive container deletion and recreation.

```text
MySQL Container
       │
       ▼
/var/lib/mysql
       │
       ▼
mysql-data Volume

Delete Container
       ↓
Volume remains
       ↓
Create new Container
       ↓
Mount same Volume
       ↓
Previous Database Data Available
```

This was also tested by deleting and recreating the database container.

---

## 🌐 Docker Networking

The three services need to communicate with one another. With Docker Compose, a network is automatically created for the services.

The backend can communicate with the database using the Compose service name `db` instead of `localhost`:

```env
DB_HOST=db
```

This works because Docker Compose provides internal DNS-based service discovery.

```text
backend
   │
   │ DB_HOST=db
   ▼
db
   │
   ▼
MySQL
```

This is preferable to depending on manually assigned container IP addresses.

---

## 🧩 Docker Compose

Docker Compose was introduced to replace multiple manual Docker commands with a single declarative configuration.

Instead of manually performing `docker build`, `docker run`, `docker network create`, `docker volume create`, etc., the entire application can be managed through `docker-compose.yaml` and started with:

```bash
docker compose up --build
```

### 📄 Compose Configuration

The final Compose setup contains three services:

```yaml
services:

  db:
    ...

  backend:
    ...

  frontend:
    ...
```

### Database Service

The database service includes: MySQL image, environment variables, persistent volume, port mapping, healthcheck, and restart policy.

```yaml
db:
  image: mysql:8.0.40-oraclelinux9
  restart: always

  environment:
    MYSQL_ROOT_PASSWORD: <your-password>
    MYSQL_DATABASE: school

  volumes:
    - mysql-data:/var/lib/mysql

  ports:
    - "3306:3306"

  healthcheck:
    test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-p<your-password>"]
    interval: 10s
    timeout: 5s
    retries: 5
```

> ⚠️ **Note on secrets:** the value above is a placeholder. Set the real password via an untracked `.env` file (see [Security and Production Considerations](#-security-and-production-considerations)) rather than committing it to the repository. This project is for local learning, so a hardcoded value is acceptable *only* as a local convenience — not as a production practice.

### 🔗 Backend Service

The backend is built from the `backend/` directory:

```yaml
backend:
  build:
    context: backend/

  image: backend-image

  ports:
    - "3500:3500"

  environment:
    host: db
    user: root
    password: <your-password>
    database: school

  depends_on:
    db:
      condition: service_healthy
```

The important part is `host: db` — the backend connects to the database using the Compose service name. The backend also waits for the database healthcheck to succeed before starting.

### 🎨 Frontend Service

The frontend is built from `frontend/`. The backend URL is supplied as a build argument:

```yaml
args:
  REACT_APP_API_BASE_URL: http://localhost:3500
```

The frontend is served by Nginx:

```yaml
ports:
  - "80:80"
```

Therefore `http://localhost` opens the application.

> **Note:** this project's Compose file does not use the legacy top-level `version:` key — modern Docker Compose does not require it, and including it produces a deprecation warning.

---

## 🏥 Healthcheck and Service Dependencies

The database uses a Docker healthcheck:

```yaml
healthcheck:
  test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-p<your-password>"]
  interval: 10s
  timeout: 5s
  retries: 5
```

The backend depends on the database being healthy:

```yaml
depends_on:
  db:
    condition: service_healthy
```

This is better than simply checking whether the database container has started. There is an important distinction:

- `service_started` — the container has started.
- `service_healthy` — the application's healthcheck has successfully passed.

---

## 🔐 Environment Variables and Build Arguments

The React frontend requires the backend API URL. The Dockerfile uses:

```dockerfile
ARG REACT_APP_API_BASE_URL
ENV REACT_APP_API_BASE_URL=$REACT_APP_API_BASE_URL
```

The value can then be supplied during the image build:

```bash
docker build \
  --build-arg REACT_APP_API_BASE_URL=http://localhost:3500 \
  -t frontend-image \
  frontend/
```

With Docker Compose, the same value can be provided declaratively:

```yaml
build:
  context: frontend/
  args:
    REACT_APP_API_BASE_URL: http://localhost:3500
```

### Important Concept

For Create React App-style applications, `REACT_APP_*` values are incorporated into the frontend bundle **during the build**. Therefore, changing the value after the React application has already been built does not automatically change the value embedded in the JavaScript bundle.

---

## ▶️ Running the Project

### Prerequisites

Make sure the following are installed:
- Docker
- Docker Compose / Docker Compose plugin
- Git

Verify installation:

```bash
docker --version
docker compose version
```

### 🚀 Run Using Docker Compose

From the project root:

```bash
docker compose up --build
```

To run in detached mode:

```bash
docker compose up -d --build
```

Check running containers:

```bash
docker compose ps
# or
docker ps
```

### 🌍 Access the Application

Once all services are running:

| Service | URL |
|---|---|
| Frontend | http://localhost |
| Backend | http://localhost:3500 |
| MySQL | localhost:3306 |

The MySQL port is primarily intended for database connectivity and administration rather than browser access.

---

## 🧪 Testing the Application

The following end-to-end tests were performed.

### Frontend Test

Verified that `http://localhost` loads the Student–Teacher application.

### Backend Test

Verified that `http://localhost:3500` returns a backend response containing application data, for example:

```json
{
  "message": "From Backend!!!",
  "studentData": [
    {
      "id": 1,
      "name": "Neha",
      "roll_number": "46",
      "class": "ECE"
    }
  ]
}
```

### Student CRUD Flow

- Adding a student
- Viewing student data
- Deleting a student
- Verifying the changes through the backend

### Teacher Flow

- Adding teacher information
- Viewing teacher information
- Deleting teacher information
- Verifying backend/database connectivity

### 🔄 End-to-End Verification

The complete data flow was verified:

```text
Browser
   │
   ▼
React Frontend
   │
   │ HTTP API
   ▼
Node.js Backend
   │
   │ SQL
   ▼
MySQL
   │
   │ Data
   ▼
Node.js Backend
   │
   ▼
React Frontend
   │
   ▼
Browser
```

This confirmed that the three tiers were correctly connected.

---

## 📦 Image Optimization

One of the major goals of this project was reducing Docker image size.

### Before Multi-Stage Optimization

| Image | Approximate Size |
|---|---|
| Frontend | ~594 MB |
| Backend | ~891 MB |
| MySQL | ~953 MB |

### After Optimization

| Image | Approximate Size |
|---|---|
| Frontend | ~49 MB |
| Backend | ~152 MB |
| MySQL 8.0.40 | ~608 MB |

### Frontend Optimization

The frontend image dropped from approximately **594 MB → 49 MB**, achieved by:
- Building the React application in a Node.js stage
- Using `nginx:alpine` as the final stage
- Copying only the generated production files

### Backend Optimization

The backend image dropped from approximately **891 MB → 152 MB**. Optimization techniques included:
- Alpine-based Node.js image
- Production-only dependencies
- Multi-stage build
- Removing unnecessary build dependencies
- Running the application as a non-root user

---

## 🐛 Problems Encountered and Solutions

This project involved several real Docker troubleshooting scenarios.

### 1. Frontend Could Not Be Accessed

**Problem:** The frontend container was running, but the browser showed `Welcome to nginx!`.

**Cause:** The host's existing Nginx service was already listening on port `80`. Therefore, requests to `http://localhost` were reaching the host Nginx service instead of the Docker container.

**Investigation:**

```bash
sudo ss -ltnp | grep ':80'
```

showed `nginx` listening on port 80.

**Solution:** The conflicting Nginx service was stopped so Docker could bind port 80.

### 2. Frontend Server Was Listening Only on `localhost`

Initially the frontend logs showed:

```text
Accepting connections at http://localhost:3000
```

This prevented external access to the application inside the container. The server was changed to listen on `0.0.0.0:3000`, allowing connections through Docker's published port.

### 3. Frontend Could Load but Could Not Add Data

**Problem:** The React application loaded successfully, but adding student/teacher data failed.

**Cause:** The React application's API URL had not been correctly supplied during the frontend image build.

**Solution:** The backend URL was passed as a build argument:

```bash
--build-arg REACT_APP_API_BASE_URL=http://localhost:3500
```

and later configured through Docker Compose:

```yaml
args:
  REACT_APP_API_BASE_URL: http://localhost:3500
```

After rebuilding the frontend image, API operations worked correctly.

### 4. MySQL Container Healthcheck Problems

An issue was encountered when using a floating MySQL image version with existing database data.

**Cause:** The database volume contained data created using a different MySQL version. Using `mysql:latest` allowed the MySQL version to change unexpectedly.

**Solution:** The MySQL image was pinned to a specific version: `mysql:8.0.40-oraclelinux9`. Pinning image versions makes builds more predictable and avoids unexpected version changes.

### 5. Database Persistence Testing

The database container was deleted and recreated using the same named volume, to verify that the database data survives container deletion.

The result confirmed:

```text
Container deleted
      ↓
Volume remains
      ↓
New container created
      ↓
Same volume mounted
      ↓
Previous data available
```

This demonstrated the difference between container lifecycle and persistent storage lifecycle.

### 6. Docker Compose Image Pull Failure

During one Compose build, Docker failed while downloading the MySQL image with an `EOF` error from Docker's image distribution/CDN endpoint. The failure occurred while the image was almost completely downloaded.

This was identified as an image download/network interruption rather than a Docker Compose configuration problem. Retrying the image pull/build resolved the issue.

---

## 🧰 Docker Commands Used

**Build an Image**
```bash
docker build -t frontend-image frontend/
docker build -t backend-image backend/
```

**Run a Container**
```bash
docker run -d -p 80:3000 frontend-image
```

For the Nginx-based frontend:
```bash
docker run -d -p 80:80 frontend-image
```

**List Containers**
```bash
docker ps
docker ps -a
```

**View Logs**
```bash
docker logs frontend-container
docker logs backend-container
```

**Remove a Container**
```bash
docker rm -f frontend-container
```

**Inspect Images**
```bash
docker images
```

**Pull an Image**
```bash
docker pull mysql:8.0.40-oraclelinux9
```

**Docker Compose**
```bash
# Start and build
docker compose up --build

# Start in detached mode
docker compose up -d --build

# Stop services
docker compose down

# View Compose services
docker compose ps

# View logs
docker compose logs
```

---

## 🔒 Security and Production Considerations

This project is intended for learning and local development. Some configurations should be improved before using the application in a real production environment.

### 1. Do Not Hardcode Database Passwords

The Compose examples in this README use a placeholder (`<your-password>`). For local runs, set the real value in an untracked `.env` file and commit a `.env.example` instead of a real `.env`:

```env
# .env.example
MYSQL_ROOT_PASSWORD=changeme
```

Production alternatives include:
- Docker secrets
- AWS Secrets Manager
- AWS Systems Manager Parameter Store
- HashiCorp Vault
- Kubernetes Secrets

### 2. Avoid Using the MySQL Root User

The backend currently uses the MySQL root account. A production application should use a dedicated database user with only the required permissions.

### 3. Pin Image Versions

Avoid `mysql:latest`. Prefer a specific version, e.g. `mysql:8.0.40-oraclelinux9`. Pinning versions improves reproducibility and reduces unexpected breaking changes.

### 4. Run Containers as Non-Root

The backend runtime was configured to use a non-root user. Running applications as non-root reduces the impact of a potential container compromise.

### 5. Production HTTPS

A real deployment should use HTTPS/TLS rather than exposing the application directly over HTTP. A reverse proxy or load balancer could be introduced for production deployment.

---

## 🧠 What I Learned

This project provided hands-on experience with the following Docker concepts:

**Docker Fundamentals**
- Images, containers, Dockerfiles
- Port mapping, container logs, image layers
- Container lifecycle

**Docker Networking**
- Custom Docker networks
- Container-to-container communication
- Service-name-based DNS
- Host ports vs container ports
- `localhost` vs container service names

**Docker Storage**
- Named volumes
- Persistent database storage
- Container lifecycle vs data lifecycle

**Dockerfile Best Practices**
- `FROM`, `WORKDIR`, `COPY`, `RUN`, `ARG`, `ENV`, `EXPOSE`, `CMD`
- `.dockerignore`
- Layer caching
- Multi-stage builds

**Multi-Stage Builds**
- Separating build and runtime environments
- Reducing image size
- Using Node.js only during the frontend build
- Serving production React files with Nginx
- Minimizing runtime dependencies

**Docker Compose**
- Declarative multi-container configuration
- Service definitions, build contexts
- Environment variables, build arguments
- Volumes, healthchecks, `depends_on`
- Automatic Compose networking
- Restart policies

**Troubleshooting**
- Port conflicts
- Container networking issues
- Frontend/backend API connectivity
- Build-time environment variables
- Database persistence
- Image version compatibility
- Docker image download failures

---

## 📈 Project Evolution

The project was developed progressively.

**Phase 1 — Individual Containers**
```text
MySQL Container
      ↓
Backend Container
      ↓
Frontend Container
```
Each container was manually built and run.

**Phase 2 — Docker Networking**

A custom Docker network was introduced so that the containers could communicate with each other.

```text
┌───────────── Docker Network ─────────────┐
│                                          │
│  Frontend ←→ Backend ←→ MySQL            │
│                                          │
└──────────────────────────────────────────┘
```

**Phase 3 — Persistent Storage**

A named Docker volume (`mysql-data`) was introduced for MySQL, allowing database data to survive container recreation.

**Phase 4 — Multi-Stage Builds**

Frontend and backend images were optimized using multi-stage Dockerfiles.

```text
Node.js build stage
       ↓
React production build
       ↓
Nginx runtime stage
```

**Phase 5 — Docker Compose**

All services were moved into a single Compose configuration. Instead of manually running multiple commands, the entire application can now be started with:

```bash
docker compose up --build
```

---

## 🚀 Future Improvements

This project is intended to be extended as part of a broader Cloud/DevOps learning path.

- [ ] GitHub Actions CI/CD
- [ ] Automated Docker image builds
- [ ] Push images to Docker Hub / container registry
- [ ] Deploy application to AWS
- [ ] Kubernetes Deployment manifests
- [ ] Kubernetes Services
- [ ] ConfigMaps
- [ ] Kubernetes Secrets
- [ ] Ingress
- [ ] Helm charts
- [ ] Monitoring with Prometheus
- [ ] Grafana dashboards
- [ ] Centralized logging
- [ ] HTTPS/TLS
- [ ] Production database configuration
- [ ] External secrets management

---

## 📸 Screenshots

> Add screenshots to this section after uploading them to the repository.

**Application Homepage**
`![Application Homepage](screenshots/homepage.png)`

**Student Page**
`![Student Page](screenshots/student-page.png)`

**Teacher Page**
`![Teacher Page](screenshots/teacher-page.png)`

**Docker Containers**
`![Docker Containers](screenshots/docker-ps.png)`

**Docker Compose**
`![Docker Compose](screenshots/docker-compose.png)`

**Docker Images**
`![Docker Images](screenshots/docker-images.png)`

> Recommended: create a `screenshots/` directory in the repository and place the images there.

---

## 🎯 Key Takeaways

This project demonstrates how a traditional three-tier application can be transformed into a containerized application architecture:

```text
Traditional Application
        ↓
Dockerfiles
        ↓
Individual Containers
        ↓
Docker Networking
        ↓
Persistent Volumes
        ↓
Multi-Stage Builds
        ↓
Nginx Production Frontend
        ↓
Docker Compose
        ↓
Reproducible Multi-Container Application
```

The project also demonstrates an important Docker principle:

> Containers should be treated as disposable, while persistent application data should live outside the container lifecycle.

---

## 👨‍💻 Author

**Neha V M**

B.Tech — Electronics & Communication Engineering

Interested in: Cloud Computing · DevOps · Docker · Kubernetes · AWS · Linux · CI/CD · Computer Networks

---

## 🙏 Acknowledgements

The base Student–Teacher application and the instructional project workflow were provided through an instructor-led learning project.

This repository documents my hands-on work and learning around:
- Docker containerization
- Dockerfiles
- Multi-stage builds
- Nginx
- Docker networking
- Docker volumes
- Docker Compose
- Healthchecks
- Container troubleshooting
- Image optimization
