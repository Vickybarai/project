# 📘 EasyCRUD Project – Docker Compose / yaml 

This document explains how to run the **EasyCRUD (Frontend + Backend + MariaDB)** project using **Docker Compose**, starting from Docker installation up to full application execution.

---

## Phase 0: Prerequisites

### Step 0.1: Install Docker

```bash
sudo apt update
sudo apt install docker.io -y
sudo systemctl start docker
sudo systemctl enable docker
docker --version
```

---

### Step 0.2: Install Docker Compose

```bash
sudo apt install docker-compose -y
docker-compose --version
```

---

## Phase 1: Project Setup

### Step 1: Clone Repository

```bash
git clone https://github.com/shubhamkalsait/EasyCRUD.git
cd EasyCRUD
git checkout cdec-b48
```

---

### Step 2: Verify Folder Structure

The project directory **must look exactly like this**:

```
EasyCRUD/
├── docker-compose.yml
├── backend/
│   ├── Dockerfile
│   └── (Spring Boot source code)
└── frontend/
    ├── Dockerfile
    └── (Angular/React source code)
```

Create the compose file at root level:

```bash
touch docker-compose.yml
```

---

## Phase 2: Understanding docker-compose.yml

### What is docker-compose.yml

A **YAML-based configuration file** that defines:

* Multiple services
* Their images/builds
* Networking
* Volumes
* Startup order

Docker Compose converts this file into:

* Containers
* Networks
* Volumes
  with **one command**

---

### YAML Rules (Mandatory)

* Indentation is **spaces only**
* Each level = **2 spaces**
* Structure = **key : value**
* Lists use `-`

Example:

```yaml
ports:
  - "8080:8080"
```

---

## Phase 3: docker-compose.yml (Correct & Working)

Paste **exactly** this file:

```yaml
version: '3.8'

services:

  database:
    image: mariadb:latest
    container_name: mariadb-container
    environment:
      MARIADB_ROOT_PASSWORD: "redhat"
      MARIADB_DATABASE: "studentapp"
    volumes:
      - my-vol123:/var/lib/mysql
    ports:
      - "3306:3306"
    networks:
      - school-net

  backend:
    build: ./backend
    container_name: backend-container
    ports:
      - "8080:8080"
    environment:
      SPRING_DATASOURCE_URL: "jdbc:mariadb://database:3306/studentapp"
      SPRING_DATASOURCE_USERNAME: "root"
      SPRING_DATASOURCE_PASSWORD: "redhat"
    depends_on:
      - database
    networks:
      - school-net

  frontend:
    build: ./frontend
    container_name: frontend-container
    ports:
      - "80:80"
    depends_on:
      - backend
    networks:
      - school-net

volumes:
  my-vol123:

networks:
  school-net:
```

---

## Phase 4: How This YAML Works (Flow Explanation)

### Service Resolution

* Services communicate using **service names**
* `database` becomes the hostname inside Docker network

Example:

```text
backend → jdbc:mariadb://database:3306/studentapp
```

No IP lookup.
No container inspect.
No hardcoding.

---

### Volume Handling

```yaml
volumes:
  - my-vol123:/var/lib/mysql
```

* Data stored outside container
* Container removal does not delete DB data

---

### Network Handling

```yaml
networks:
  - school-net
```

* All services join same bridge network
* DNS resolution works automatically
* Containers talk by service name

---

### Startup Order

```yaml
depends_on:
  - database
```

* Controls container startup sequence
* Does not wait for DB readiness

---

## Phase 5: Backend Configuration

### application.properties (Backend)

```properties
server.port=8080

spring.datasource.url=jdbc:mariadb://database:3306/studentapp
spring.datasource.username=root
spring.datasource.password=redhat

spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
```

---

## Phase 6: Execution (One Command)

Run from **EasyCRUD root directory**:

```bash
docker-compose up -d --build
```

### Command Breakdown

* `up` → create and start services
* `-d` → detached mode
* `--build` → rebuild images if code changed

---

## Phase 7: Verification

### Check Container Status

```bash
docker-compose ps
```

Expected:

* mariadb-container → running
* backend-container → running
* frontend-container → running

---

### View Logs

```bash
docker-compose logs backend
docker-compose logs database
docker-compose logs frontend
```

---

## Phase 8: Browser Test

Open:

```text
http://localhost
```

or (EC2):

```text
http://<PUBLIC_IP>
```

Expected Flow:

```
Browser → Frontend (Nginx)
Frontend → Backend (8080)
Backend → MariaDB (3306)
```

---

## Phase 9: Stop & Cleanup

### Stop Services

```bash
docker-compose down
```

### Stop + Remove Volumes (Data Loss)

```bash
docker-compose down -v
```

---

## Phase 10: Summary Flow

```
docker-compose.yml
        ↓
Creates Network
        ↓
Starts Database
        ↓
Starts Backend (connects via service name)
        ↓
Starts Frontend
        ↓
Application Accessible via Browser
```




🚨 Common Error: Backend Fails to Connect to Database

🔴 Error Message Seen in Logs

Connection refused host=database port=3306
Unable to determine Dialect without JDBC metadata
HikariPool - Exception during pool initialization


---

🎯 Why This Error Happens

Even though Docker Compose starts services in order:

depends_on:
  - database

This does NOT mean the database is ready.

It only means:

✔ Container started
❌ Database service inside container may still be initializing

MariaDB needs a few seconds to:

Create system tables

Apply configurations

Start accepting connections


During this time, backend tries to connect → connection refused → application crashes.


---

✅ Solution: Add Health Check

We will make Docker wait until MariaDB is actually ready, not just started.


---

🛠 Step 1 — Add Healthcheck to Database Service

Update docker-compose.yml:

database:
  image: mariadb:latest
  container_name: mariadb-container
  environment:
    MARIADB_ROOT_PASSWORD: "redhat"
    MARIADB_DATABASE: "studentapp"
  volumes:
    - my-vol123:/var/lib/mysql
  ports:
    - "3306:3306"
  networks:
    - school-net

  healthcheck:
    test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
    interval: 5s
    timeout: 5s
    retries: 10

🔍 What this does

Docker will run:

mysqladmin ping -h localhost

inside the DB container every 5 seconds.

Only when it returns:

mysqld is alive

Docker marks the DB as healthy.


---

🛠 Step 2 — Make Backend Wait for Healthy DB

Modify backend service:

backend:
  build: ./backend
  container_name: backend-container
  ports:
    - "8080:8080"
  environment:
    SPRING_DATASOURCE_URL: "jdbc:mariadb://database:3306/studentapp"
    SPRING_DATASOURCE_USERNAME: "root"
    SPRING_DATASOURCE_PASSWORD: "redhat"
  depends_on:
    database:
      condition: service_healthy
  networks:
    - school-net


---

🔄 Step 3 — Rebuild Services

docker-compose down
docker-compose up -d --build


---

🧠 What Happens Now (Flow)

Docker starts MariaDB container
        ↓
Runs healthcheck command
        ↓
Waits until DB responds
        ↓
Marks DB as HEALTHY
        ↓
Backend container starts
        ↓
Backend connects successfully


---

🟢 How to Verify Health Status

docker ps

You will see:

mariadb-container   Up (healthy)

Or:

docker inspect mariadb-container | grep Health


---

🏆 Result

Before	After

Backend crashes on first run	Backend always starts correctly
Manual restart needed	Fully automated
Unstable startup	Production-style reliability



