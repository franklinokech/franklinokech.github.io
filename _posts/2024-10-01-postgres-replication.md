---
title: 'PostgreSQL Logical Replication with Docker Compose'
date: 2014-10-01
permalink: /posts/2024/10/postgres-replication/
tags:
  - PostgreSQL
  - Docker
  - Logical Replication
  - DevOps
---

Logical replication in PostgreSQL **allows table-level replication** from a primary database to one or more replicas.Unlike **physical replication**, logical replication is **flexible**, supports **multiple replicas**, and works across versions.

In this guide, we’ll set up one primary PostgreSQL node and four secondary replicas using Docker

## Docker Compose Overview

```bash
services:
  postgres-primary:
    image: postgres:15
    container_name: postgres-primary
    environment:
      POSTGRES_DB: testdb
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
    ports:
      - "5432:5432"
    volumes:
      - ./primary-data:/var/lib/postgresql/data
      - ./primary-init:/docker-entrypoint-initdb.d
    command: ["postgres", "-c", "wal_level=logical", "-c", "max_replication_slots=3", "-c", "max_wal_senders=10"]
    networks:
      - postgres-net

  postgres-secondary:
    image: postgres:15
    container_name: postgres-secondary
    environment:
      POSTGRES_DB: testdb
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
    ports:
      - "5433:5432"
    volumes:
      - ./secondary-data:/var/lib/postgresql/data
      - ./secondary-init:/docker-entrypoint-initdb.d
    depends_on:
      - postgres-primary
    networks:
      - postgres-net

  postgres-secondary2:
    image: postgres:15
    container_name: postgres-secondary2
    environment:
      POSTGRES_DB: testdb
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
    ports:
      - "5434:5432"
    volumes:
      - ./secondary2-data:/var/lib/postgresql/data
    depends_on:
      - postgres-primary
    networks:
      - postgres-net

  postgres-secondary3:
    image: postgres:15
    container_name: postgres-secondary3
    environment:
      POSTGRES_DB: testdb
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
    ports:
      - "5435:5432"
    volumes:
      - ./secondary3-data:/var/lib/postgresql/data
    depends_on:
      - postgres-primary
    networks:
      - postgres-net

  postgres-secondary4:
    image: postgres:15
    container_name: postgres-secondary4
    environment:
      POSTGRES_DB: testdb
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
    ports:
      - "5436:5432"
    volumes:
      - ./secondary4-data:/var/lib/postgresql/data
    depends_on:
      - postgres-primary
    networks:
      - postgres-net

networks:
  postgres-net:
    driver: bridge

```

## How It Works
### **Primary (publisher)**:
- Sends data changes via **WAL** logs.
- Uses wal_level=logical to enable logical replication.
- Exposes **replication slots** for replicas.

## **Secondary (subscriber):**
- Connects to primary to pull updates using **subscriptions**.
- Each replica has its own persistent volume and port.

**Networking**: All containers communicate via the postgres-net bridge network.

## Setting Up Replication
### On Primary: Create Publication

```sql
CREATE PUBLICATION test_pub FOR ALL TABLES;
```

### On Secondary: Create Table Schema

```sql
CREATE TABLE test_table (
    id SERIAL PRIMARY KEY,
    name VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### On Secondary: Create Subscription

```sql
CREATE SUBSCRIPTION test_sub
CONNECTION 'host=postgres-primary port=5432 user=postgres password=password dbname=testdb'
PUBLICATION test_pub;
```

### Test Replication
- Insert on primary: INSERT INTO test_table (name) VALUES ('New Data');
- heck on secondary: SELECT * FROM test_table;

### Add More Tables
- On primary: CREATE TABLE new_table (...);
- Add to publication: ALTER PUBLICATION test_pub ADD TABLE new_table;
- Refresh subscription: ALTER SUBSCRIPTION test_sub REFRESH PUBLICATION;