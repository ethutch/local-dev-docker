# Local Development & Observability Infrastructure Stack

This project builds up a comprehensive development environment.  The goal is to provide an enterprise-grade, fully containerized local development, observability, and cloud-emulator environment managed via Docker. This environment provides database instances, messaging queues, monitoring tools, CI/CD pipelines, and cloud service emulation under a single local-dev Docker network.
---

## Licensing

This has been tested in a Docker Desktop environment.  Conceptually it should work in other open source containerizaation solutions such as Podman.  However at time of creation there were stability problems with portions of the stack, espeially QEMU.  All software except Docker Desktop is open source.  Please observe licensing requirements for using Docker Desktop in a corporate environment. It should also be possible to use the open source components of docker installed directly without Docker Desktop packaging.  This would preserve the unencumbered envirnonment but will lose the UI benefits of Docker Desktop.

## 🏗 System Architecture & Network Topology

All containers in this stack reside within a single custom bridge network named `local-dev`. This network serves as a virtual Private Network Interface (NIC) layer enabling service-to-service communication via Docker's built-in DNS engine.

```
                         [ HOST MACHINE ]
                                |
      +-------------------------+-------------------------+
      | (Port Mappings)                                   |
      v                                                   v
+-------------------------------------------------------------------------+
| Docker Network: local-dev (Bridge)                                      |
|                                                                         |
|  [ Datastores ]            [ Messaging / Eventing ]  [ Monitoring ]     |
|    * postgres-local (5432)   * kafka (9092, 9094)      * prometheus     |
|    * mongodb (27017)         * apicurio (8083)         * grafana (3000) |
|    * dynamodb-local (8000)   * apicurio-ui (8084)      * exporters...   |
|                              * kafka-ui (8085)                          |
|                                                                         |
|  [ Cloud & DevOps ]                                                     |
|    * local-stack [AWS]     (4566)                                       |
|    * gitea                 (3080, 2222)                                 |
|    * jenkins               (8090)                                       |
|    * registry [Kubernetes] (5001)                                       |             |
+-------------------------------------------------------------------------+
```

### Network Communication Rules
* **Container-to-Container:** Services communicate directly using their container/service names as hostnames over internal Docker network ports (e.g., `postgres-local:5432` or `prometheus:9090`).
* **Cross Network:** Services that have to talk cross network use the special Docker DNS name host.docker.internal.  For example this is required for a kubernetes service to connect to Postgres.
* **Host-to-Container:** Host applications (IDE, database GUIs, browser) interact with services using `localhost` or `127.0.0.1` on published host ports (e.g., `localhost:3000` for Grafana).

---
## 🎯 Quick Start

### 1. Create the directories for the locally persisted data
```bash
mkdir -p local-dev-data/{docker-registry,prometheus-data,grafana-data,kafka-data,jenkins_home,postgres-data,mongo-data,dynamodb/data,localstack-data,gitea-data}
 ```

### 2. Create the External Docker Network
Because the stack uses an external network named local-dev, create it manually before starting:

```bash
docker network create local-dev
```

### 3. Prepare Directory Structure
Ensure required configuration directories and files exist relative to your Compose file:

```
local-dev-docker
├── config
│   ├── grafana
│   │   └── provisioning
│   │       └── datasources
│   │           └── loki.yaml
│   └── prometheus
│       └── prometheus.yml
├── docker-compose-local-dev.yaml


local-dev-data
├── docker-registry
├── dynamodb
├── gitea-data
├── grafana-data
├── jenkins_home
├── kafka-data
├── localstack-data
├── mongo-data
├── postgres-data
└── prometheus-data

```

### 4. Launch the Stack
Run Docker Compose with the build flag to build the embedded Jenkins image and start all services in detached mode:

```docker compose -f <your-compose-filename>.yaml up -d --build```

---

## 🌐 Service Directory & Port Mappings

| Service Name | Container Name | Host Port(s) | Container Port(s) | Description |
|---|---|---|---|---|
| Docker Registry | registry | 5001 | 5000 | Local v2 Docker Registry (Kubernetes support) |
| Prometheus | prometheus | 9090 | 9090 | Metrics collection and alert management |
| Grafana | grafana | 3000 | 3000 | Metrics visualization and dashboards |
| Postgres Exporter | postgres-exporter | 9187 | 9187 | Prometheus exporter for PostgreSQL |
| MongoDB Exporter | mongodb-exporter | 9216 | 9216 | Prometheus exporter for MongoDB |
| Kafka (KRaft) | kafka | 9092, 9094 | 9092, 9094 | Apache Kafka broker (No Zookeeper required) |
| Kafka Exporter | kafka-exporter | 9308 | 9308 | Prometheus exporter for Kafka |
| Apicurio Registry | apicurio-registry-v3 | 8083, 9000 | 8080, 9000 | Schema Registry v3 API |
| Apicurio UI | apicurio-registry-v3-ui | 8084 | 8080 | Web UI for Apicurio Schema Registry |
| Kafka UI | kafka-ui | 8085 | 8080 | Web UI for managing Kafka topics and schemas |
| Node Exporter | node-exporter | 9100 | 9100 | Hardware and OS metrics exporter for the Docker VM |
| Jenkins | jenkins | 8090, 50000 | 8080, 50000 | CI/CD Server (pre-loaded with Docker CLI & Helm) |
| PostgreSQL | postgres-local | 5432 | 5432 | PostgreSQL 17 database |
| MongoDB | mongodb | 27017 | 27017 | MongoDB document store |
| DynamoDB Local | dynamodb-local | 8000 | 8000 | Local AWS DynamoDB emulator |
| LocalStack | local-stack | 4566 | 4566 | AWS Cloud Service Emulator (S3, SQS, Lambda, etc.) |
| Gitea | gitea | 3080, 2222 | 3000, 22 | Self-hosted Git service |

---

## Default Credentials

* Grafana: ```admin / admin123```
* PostgreSQL: ```postgres / passw0rd!```
* MongoDB: ```admin / passw0rd!```
* Jenkins: Initial admin password accessible via volume or container logs (docker logs jenkins)

---

## 📦 Detailed Service Breakdown

### 1. Monitoring & Observability Stack


#### Prometheus (`prometheus`)
* **Role:** Time-series metrics datastore and alerting daemon.
* **Internal Endpoint:** `http://prometheus:9090`
* **Host UI Endpoint:** `http://localhost:9090`
* **Configuration:** Scrapes targets defined in `./config/prometheus/prometheus.yml`.
* **Data Persistence:** Mounts `~/local-dev-data/prometheus/` to persist TSDB blocks across container lifecycles.

#### Grafana (`grafana`)
* **Role:** Central visualization engine for system metrics and telemetry dashboards.
* **Internal Endpoint:** `http://grafana:3000`
* **Host UI Endpoint:** `http://localhost:3000`
* **Default Credentials:** `admin` / `admin123`
* **Provisioning:** Automatically loads datasources from `./config/grafana/provisioning/datasources/` and dashboards from `./config/grafana/provisioning/dashboards/`.

#### Metrics Exporters
* **Postgres Exporter (`postgres-exporter`):** Connects to `postgres-local:5432` and exposes PostgreSQL engine metrics at `http://postgres-exporter:9187/metrics`.
* **MongoDB Exporter (`mongodb-exporter`):** Connects to `mongodb:27017` and exposes document database metrics at `http://mongodb-exporter:9216/metrics`.
* **Kafka Exporter (`kafka-exporter`):** Connects to `kafka:9092` to scrape broker throughput, consumer group lag, and topic offsets at `http://kafka-exporter:9308/metrics`.
* **Node Exporter (`node-exporter`):** Mounts host system metrics (/proc, /sys) to expose kernel, memory, CPU, and disk utilization at `http://node-exporter:9100/metrics`.

---

### 2. Event Streaming & Schema Governance

#### Apache Kafka - KRaft Mode (`kafka`)
* **Role:** Distributed event-streaming platform operating in modern ZooKeeper-less (KRaft) mode.
* **Internal Listener:** `kafka:9092` (for containerized consumers/producers)
* **Host Listener:** `localhost:9094` (for host applications/IDEs)
* **Data Persistence:** Mounts `~/local-dev-data/kafka/` for log segment retention.

#### Apicurio Registry v3 & UI (`apicurio-registry-v3`, `apicurio-registry-v3-ui`)
* **Role:** Schema governance platform supporting Avro, JSON Schema, and Protobuf.
* **Description:** Version 3 adds a schema editor removing the requirement for other tools.
* **API Endpoint:** `http://localhost:8083` (Internal: `http://apicurio-registry-v3:8080`)
* **Web Management UI:** `http://localhost:8084`

#### Kafka
* **Role:** Message Broker
* **Listeners:** 
  *   **Inside Docker Network:**   kafka:29092
  *   **Outside Docker Network:**  host.docker.internal:9092
  *   **Local (outside Docker):**  localhost:9094


#### Kafka UI (`kafka-ui`)
* **Role:** Management console for inspecting Kafka topics, message payloads, consumer group lag, and schema bindings.
* **Host UI Endpoint:** `http://localhost:8085`

---

### 3. Datastores & Cloud Emulation

#### PostgreSQL (`postgres-local`)
* **Engine:** PostgreSQL 17
* **Host Endpoint:** `localhost:5432`
* **Credentials:** User `postgres`, Password `passw0rd!`

#### MongoDB (`mongodb`)
* **Host Endpoint:** `localhost:27017`
* **Credentials:** User `admin`, Password `passw0rd!`

#### AWS Cloud Emulators
* **LocalStack (`local-stack`):** AWS emulator providing S3, SQS, SNS, Lambda, EventBridge, and Secrets Manager. Accessible at `http://localhost:4566`.
* **AWS Services:** Which AWS services are emulated is controlled by the ***SERVICES*** environment variable.  Note that the supplied docker-compose.yaml file starts these services: **apigateway, events, iam, lambda, s3, sqs, sts**. Add / Remove services as required.  
 
  It is also possible to specify an astrick **'\*'** which will start all supported services.  This is discouraged because it will consume a large amount of local resources. It will also start the local-stack internal DynamoDB instance.  There will then be two instancaces of DynamoDB running; this is managable but certainly not the desirred configuration.  It is best practice to only start the services you need.

* **Community Fork:** The main local-stack project now requires a paid subscription to enable persistence.  For that reason we are using the gresau/localstack-persist fork under the Apache-2.0 license.  It is likely this fork will lose its correlation to AWS over time. But for the medium term it is more than adaquate for all the most popular AWS features.
* **DynamoDB Local (`dynamodb-local`):** Dedicated NoSQL DynamoDB instance at `http://localhost:8000`.  Note that DynamoDB can run under local-stack but that introduces even more persistence complexities.  But running it as an external pod we can keep the supported native Amazon container.

---

### 4. CI/CD & Developer Services

#### Jenkins (`jenkins`)
* **Role:** CI/CD Automation server built with pre-loaded Docker CLI and Helm CLI binaries.
* **Host Web UI:** `http://localhost:8090`
* **Agent Port:** `50000`
* **Initialization:** To retrieve the initial admin password:
  ```bash
  docker logs jenkins 2>&1 | grep -A 2 "Jenkins initial setup is required"
  ```
* **Pipelines:** build pipelines are not supplied for any projects.  They must be created to reflect your development needs. 

#### Self-Hosted Gitea (`gitea`)
* **Role:** Self-hosted Git repository service.
* **Rationale:** The intention is to supply a local git UI server to manage repos that are NAS hosted over smb. This is somewhat of an edge case and may be more effort than it is worth.  Specifically, git is not performant over smb.
* **Host Web UI:** `http://localhost:3080`
* **SSH Port:** `2222`

#### Local Container Registry (`registry`)
* **Role:** Local v2 OCI Docker image registry.
* **Host Endpoint:** `127.0.0.1:5001`
* **Description:** This is the missing piece to allow local kubernetes to find container images.  It functions somewhat analogous to Artifactory.

---

## 📊 Grafana Configuration & Connection Guide

### Accessing Grafana
1. Open your browser and navigate to `http://localhost:3000`.
2. Login with:
   * **Username:** `admin`
   * **Password:** `admin123`
3. Skip or set a new password on initial prompt.
4. Import community dashboards from grafana.com.  Suggestions supplied below.

---

## 🛠 Prometheus & Grafana Configuration Files

This configuration **is supplied** in the project.  It provides the datasources for the major software packages supplied in this project.  If you are using the package unmodified this plus the recommended Grafana dashboards provide an excellent starting point.  

### 1. Prometheus Scraping Configuration (Supplied)
File: `./config/prometheus/prometheus.yml`

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'postgres'
    static_configs:
      - targets: ['postgres-exporter:9187']

  - job_name: 'mongodb'
    static_configs:
      - targets: ['mongodb-exporter:9216']

  - job_name: 'kafka'
    static_configs:
      - targets: ['kafka-exporter:9308']

  - job_name: 'node'
    static_configs:
      - targets: ['node-exporter:9100']
```

### 2. Grafana Automatic Datasource Provisioning
File: `./config/grafana/provisioning/datasources/prometheus.yml`

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: false
```

### 3. Grafana Automatic Dashboard Provisioning
File: `./config/grafana/provisioning/dashboards/dashboards.yml`

```yaml
apiVersion: 1

providers:
  - name: 'Default Dashboards'
    orgId: 1
    folder: ''
    type: file
    disableDeletion: false
    editable: true
    options:
      path: /var/lib/grafana/dashboards
```

---

## 📈 Recommended Grafana Dashboards

To import community dashboards into Grafana:
1. Go to **Grafana UI -> Dashboards -> Import**.
2. Enter the **Dashboard ID** and click **Load**.
3. Select `Prometheus` as the datasource and click **Import**.

| Service Target | Dashboard ID | Key Metrics Monitored |
| :--- | :--- | :--- |
| **Node Exporter (Host/OS)** | **1860** | CPU Usage, Memory Pressure, Disk I/O, Network Throughput |
| **PostgreSQL Exporter** | **9628** | Active Connections, Transactions/sec, Lock Waits, Cache Hit Ratio |
| **MongoDB Exporter** | **2583** | Oplog Lag, Operations/sec, Memory Allocation, Connections |
| **Kafka Exporter** | **7589** | Consumer Group Lag, Topic Messages/sec, Partition Offsets |
| **Prometheus Self-Monitoring** | **3662** | TSDB Compaction Time, Scrape Duration, Ingestion Rate |

---

## 🚀 Operations & Management Quick Reference

### Bootstrapping the Stack
```bash
# 1. Create external bridge network
docker network create local-dev
```
### Launch stack
```bash
# Use -build only if linux packages of Jenkins are updating
docker compose -f docker-compose-local-dev.yaml up -d --build
```

### Once deployed the easiest way to start and stop the environment
```bash
docker compose -p local-dev start

docker compose -p local-dev stop
```

### Teardown
All persistent data is safely stored in your local-dev-data directory.  Deleting the containers will not affect your databases.  This does mean if the goal is a clean slate you will have to manually delete the ```data``` directory tree.

```bash
# Stop containers (retains volume data)
docker compose -f docker-compose-local-dev.yaml down


# Complete wipe (deletes local data volumes)
docker compose -f docker-compose-local-dev.yaml down -v
```
