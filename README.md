# This is a school Project.
# CSE 511: Data Processing at Scale — Project 1


---

## Overview

This project builds a scalable, highly available data processing pipeline using graph databases and modern distributed systems tooling. It is split into two phases:

- **Phase 1** — Docker + Neo4j: Load NYC Yellow Cab trip data into a Neo4j graph database inside a Docker container, then implement PageRank and BFS graph algorithms.
- **Phase 2** — Kubernetes + Kafka: Extend the pipeline to a fully orchestrated, streaming architecture using Minikube, Kafka, and Neo4j running inside Kubernetes.

---

## Repository Structure

```
.
├── Phase 1/
│   ├── Dockerfile               # Docker image: sets up Neo4j, loads data
│   ├── data_loader.py           # Loads NYC Trip (March 2022) data into Neo4j
│   └── interface.py             # PageRank & BFS implementations (Phase 1)
│
├── Phase 2/
│   ├── zookeeper-setup.yaml     # Kubernetes Deployment + Service for Zookeeper
│   ├── kafka-setup.yaml         # Kubernetes Deployment + Service for Kafka
│   ├── neo4j-values.yaml        # Helm values for Neo4j standalone deployment
│   ├── kafka-neo4j-connector.yaml  # Kubernetes Deployment for Kafka Connect → Neo4j
│   └── interface.py             # PageRank & BFS implementations (Phase 2, over streaming data)
│
└── README.md
```

---

## Phase 1: Docker + Neo4j

### Dataset
**NYC TLC Yellow Cab Trip Records — March 2022**

### Graph Schema

| Element | Label / Type | Properties |
|---|---|---|
| Node | `Location` | `name` (integer LocationID) |
| Relationship | `TRIP` | `distance` (float), `fare` (float), `pickup_dt` (datetime), `dropoff_dt` (datetime) |

Each unique `PULocationID` and `DOLocationID` becomes a `Location` node. Each trip row becomes a `TRIP` relationship between the pickup and dropoff location nodes.

### Setup & Run

```bash
# Build the Docker image
docker build -t cse511-project1-phase1 .

# Run the container (Neo4j exposed on ports 7474 and 7687)
docker run -p 7474:7474 -p 7687:7687 cse511-project1-phase1
```

> **Note:** Allow 2–4 minutes after container start for Neo4j to become available.

Neo4j browser will be accessible at: [http://localhost:7474](http://localhost:7474)  
Default credentials: `neo4j / project1phase1`

### Verify Data Load

```cypher
-- View schema
CALL db.schema.visualization();

-- Browse sample data
MATCH (n) RETURN n LIMIT 25;
```

### Algorithms (interface.py — Phase 1)

**PageRank** — Ranks `Location` nodes by importance based on incoming trip relationships. Uses Neo4j GDS. Returns the node with the maximum and minimum PageRank score. Configurable via `max_iter` and `weight_property`.

**Breadth-First Search (BFS)** — Traverses the graph from a `start_node` to one or more `target_nodes`. Uses Neo4j GDS BFS procedure. Configurable via `start_node` and `target_nodes`.

---

## Phase 2: Kubernetes + Kafka

### Architecture

```
Source Data Stream
        │
        ▼
  [Load Balancer]
        │
        ▼
┌─────────────────────────────────────────────────────────┐
│                     Kubernetes (Minikube)                │
│                                                         │
│  Deployment (kafka) ◄──────► Deployment (zookeeper)    │
│       │                                                 │
│       ▼                                                 │
│  Deployment (kafka-connect) ──► StatefulSet (neo4j)     │
│                                        │                │
└────────────────────────────────────────│────────────────┘
                                         │
                                   [Load Balancer]
                                         │
                                  interface.py / tester.py
```

**Data flow:** `data_producer.py` → Kafka (via Load Balancer) → Kafka Connect → Neo4j → `interface.py` (analytics)

### Prerequisites

- `minikube` (with sufficient resources — increase defaults if needed)
- `helm` with the Neo4j repository added
- `kubectl`
- Python 3 with `neo4j`, `kafka-python` packages

### Step-by-Step Setup

#### Step 1 — Start Minikube & Deploy Kafka + Zookeeper

```bash
minikube start --memory=4096 --cpus=2

kubectl apply -f zookeeper-setup.yaml
kubectl apply -f kafka-setup.yaml
```

Key Kafka configuration used:
- Image: `confluentinc/cp-kafka:7.3.3`
- `KAFKA_BROKER_ID`: `1`
- `KAFKA_ZOOKEEPER_CONNECT`: `zookeeper-service:2181`
- `KAFKA_ADVERTISED_LISTENERS`: `PLAINTEXT://localhost:9092, PLAINTEXT_INTERNAL://kafka-service:29092`
- `KAFKA_AUTO_CREATE_TOPICS_ENABLE`: `true`

#### Step 2 — Deploy Neo4j via Helm

```bash
helm install neo4j neo4j/neo4j -f neo4j-values.yaml
kubectl apply -f neo4j-service.yaml
```

Neo4j is deployed in **standalone mode** with the **GDS plugin** installed.  
Password: `project1phase2`  
Internal access: `neo4j-service:7474` / `neo4j-service:7687`

#### Step 3 — Deploy Kafka Connect → Neo4j Connector

```bash
kubectl apply -f kafka-neo4j-connector.yaml
```

Uses the custom image `veedata/kafka-neo4j-connect` (Kafka Connect with the Neo4j connector plugin pre-installed). Translates Kafka topic messages into Neo4j-compatible writes.

#### Step 4 — Expose Ports & Run Pipeline

```bash
# Expose Neo4j outside Minikube (refer to grader.md for exact commands)
kubectl port-forward svc/neo4j-service 7474:7474 7687:7687 &
kubectl port-forward svc/kafka-service 9092:9092 &

# Produce data
python data_producer.py

# Run analytics
python interface.py
```

### Algorithms (interface.py — Phase 2)

Same **PageRank** and **BFS** implementations as Phase 1, now operating over data streamed in real-time through Kafka into the Kubernetes-hosted Neo4j instance.

---

## Dependencies

| Tool | Version / Notes |
|---|---|
| Neo4j | Standalone, GDS plugin v2.3.1 |
| Kafka | `confluentinc/cp-kafka:7.3.3` |
| Minikube | Latest stable |
| Helm | Latest stable (neo4j chart repo) |
| Python | 3.x — `neo4j`, `pandas`, `pyarrow`, `kafka-python` |

---

## Submission

| Item | File |
|---|---|
| Phase 1 code (Canvas) | `<asurite>.zip` → `Dockerfile`, `interface.py`, `data_loader.py` |
| Phase 2 code (Canvas) | `<asurite>-Project1-Phase2.zip` → `zookeeper-setup.yaml`, `kafka-setup.yaml`, `neo4j-values.yaml`, `kafka-neo4j-connector.yaml`, `interface.py` |
| Report (Canvas) | `<asurite>-Project1-Phase2.pdf` (3–4 pages, <20% plagiarism) |
| GitHub | Branch `phase-2` in the Fall 2024 CSE511 org repo |

---

## Notes

- Neo4j Phase 1 password: `project1phase1`
- Neo4j Phase 2 password: `project1phase2`

---
