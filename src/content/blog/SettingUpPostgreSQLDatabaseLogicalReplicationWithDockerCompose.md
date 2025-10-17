---
title: "Setting Up PostgreSQL Database Logical Replication with Docker Compose"
description: "Setting Up PostgreSQL Database Logical Replication with Docker Compose"
pubDate: 2025-11-17
author: "Manshu Sharma"
image:
  url: "/posts/post-image-1.png"
  alt: "Cover image"
tags: ["intro", "astro"]
---

Hi everyone 😀,
Sorry😅 for not posting any blogs; I was stuck in other work. In the previous post, we saw that we enabled MySQL Replication using GTID. Today, we see how to allow Postgres Database Logical Replication in our local setup using docker-compose.

#### Environment Setup
We have two PostgreSQL instances configured using Docker Compose:
1. Publisher (postgres-1): The source server where changes are captured.
2. Subscriber (postgres-2): The destination server that receives replicated data.

#### Docker Compose Configuration
1. Publisher (docker-compose.yml)
```yaml
version: '3.8'
services:
  postgres-1:
    build: ./
    container_name: postgres-1
    environment:
      POSTGRES_USER: postgresadmin
      POSTGRES_PASSWORD: admin123
      POSTGRES_DB: postgresdb
      PGDATA: "/data"
    volumes:
      - ./postgres-1/pgdata:/data
      - ./postgres-1/config:/config
      - ./postgres-1/archive:/mnt/server/archive
    ports:
      - "5000:5432"
    networks:
      - custom_network
    command: -c 'config_file=/config/postgresql.conf'
    restart: unless-stopped
networks:
  custom_network:
      name: postgres
      driver: bridge
```

2. Subscriber (docker-compose.yml)
```yaml
version: '3.8'
services:
  postgres-2:
    build: ./
    container_name: postgres-2
    environment:
      POSTGRES_USER: postgresadmin
      POSTGRES_PASSWORD: admin123
      POSTGRES_DB: postgresdb
      PGDATA: /data
    volumes:
      - ./postgres-2/pgdata:/data
      - ./postgres-2/config:/config
      - ./postgres-2/archive:/mnt/server/archive
    ports:
      - "5001:5432"
    networks:
      - custom_network
    command: -c 'config_file=/config/postgresql.conf'
    restart: unless-stopped
networks:
  custom_network:
      name: postgres
      driver: bridge
```

#### PostgreSQL Configuration
- To enable logical replication, update the *postgresql.conf* and *pg_hba.conf* for each server.
- postgresql.conf (Publisher and Subscriber)
```yml
wal_level = logical
max_wal_senders = 3
shared_preload_libraries = 'pglogical'
```
- pg_hba.conf
```yml
Publisher:

host    pub         replicator         0.0.0.0/0          md5
host    all         all                0.0.0.0/0          md5

Subscriber:

host    all         all                0.0.0.0/0          md5
```
- Steps to Configure Logical Replication

1. Start Both PostgreSQL Containers
  - docker compose -f docker-compose1.yml up -d
  - docker compose -f docker-compose2.yml up -d

2. Create a Test Database
  - CREATE DATABASE chiragLogicalRep;
  - Create a Replication Role on Publisher
  - CREATE ROLE chirag WITH REPLICATION LOGIN PASSWORD 'admin@123';
  - GRANT ALL PRIVILEGES ON DATABASE chiragLogicalRep TO chirag;

3. Set Up Publication on Publisher
  - \c chiragLogicalRep
  - CREATE TABLE products (id SERIAL, name TEXT, price DECIMAL);
  - CREATE PUBLICATION my_publication;
  - ALTER PUBLICATION my_publication ADD TABLE products;

4. Set Up Subscription on Subscriber: Connect to chiragLogicalRep on the subscriber and execute:
  - CREATE SUBSCRIPTION my_subscription CONNECTION 'host=192.168.0.211 port=5000 user=chirag password=admin@123 dbname=chiragLogicalRep'
PUBLICATION my_publication;

*Note*: Please change the IP host=192.168.0.211

5. Verify Replication: Insert data into the products table on the publisher:
```sql
INSERT INTO products (name, price) VALUES ('Pen', 5.90), ('Notebook', 9.10), ('Pencil', 8.50);
```

6. Query the products table on both servers to ensure data synchronization:
```sql
SELECT * FROM products;
```

#### Advanced Configuration
- Replica Identity
  - To enable updates and deletes, configure the replica identity on the publisher: `ALTER TABLE products REPLICA IDENTITY FULL;`
- Replication status
  - To check replication status run following command: `SELECT * FROM pg_publication_tables WHERE pubname = 'my_publication';`
 
If both servers show the same data in the *products* table that means we enabled data synchronization from publisher to subscriber.

This blog post provides a straightforward approach to setting up PostgreSQL logical replication in a Dockerized environment. For further details, visit the [Svastikkka/DOCKER](https://github.com/Svastikkka/DOCKER/blob/main/postgresql-logical-replication/ReadME.md) repository.

