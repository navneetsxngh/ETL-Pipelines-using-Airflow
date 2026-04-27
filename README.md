# 🚀 ETL Pipeline — NASA APOD → PostgreSQL with Apache Airflow

[![Apache Airflow](https://img.shields.io/badge/Apache%20Airflow-2.x-017CEE?logo=apacheairflow&logoColor=white)](https://airflow.apache.org/)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-13-336791?logo=postgresql&logoColor=white)](https://www.postgresql.org/)
[![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker&logoColor=white)](https://docs.docker.com/compose/)
[![Astronomer](https://img.shields.io/badge/Astronomer-Runtime%203.2-7352FF)](https://www.astronomer.io/)
[![Python](https://img.shields.io/badge/Python-3.9+-3776AB?logo=python&logoColor=white)](https://www.python.org/)

---

## 📌 Project Overview

This project implements a production-ready **ETL (Extract, Transform, Load) pipeline** using **Apache Airflow**, orchestrated via the **Astronomer runtime**. The pipeline:

- **Extracts** daily astronomy data from [NASA's APOD API](https://api.nasa.gov/)
- **Transforms** the JSON response to extract relevant fields (`title`, `explanation`, `url`, `date`)
- **Loads** the cleaned data into a **PostgreSQL** database

The entire setup is containerised using **Docker** and managed with **Astronomer CLI (`astro`)**, enabling consistent, reproducible local development and smooth cloud deployment.

---

## 🏗️ Architecture

```
┌──────────────────────┐        ┌──────────────────────────────────────────┐
│                      │        │              Apache Airflow               │
│   NASA APOD API      │        │  ┌─────────┐  ┌───────────┐  ┌────────┐  │
│  (SimpleHttpOperator)│──────▶ │  │ Extract │─▶│ Transform │─▶│  Load  │  │
│                      │        │  └─────────┘  └───────────┘  └───┬────┘  │
└──────────────────────┘        └───────────────────────────────────┼───────┘
                                                                    │
                                                         ┌──────────▼──────────┐
                                                         │   PostgreSQL DB      │
                                                         │  (nasa_apod table)   │
                                                         └─────────────────────┘
```

### DAG Task Flow

```
create_table >> extract_apod >> transform_apod_data >> load_data_to_postgres
```

---

## 🗂️ Project Structure

```
ETL-Pipelines-using-Airflow/
├── .astro/                     # Astronomer project config
├── dags/
│   └── nasa_apod_postgres.py   # Main ETL DAG
├── tests/
│   └── dags/
│       └── test_dag_example.py # DAG integrity tests
├── Dockerfile                  # Astronomer runtime base image
├── docker-compose.yml          # PostgreSQL service definition
├── requirements.txt            # Python dependencies
├── packages.txt                # OS-level packages (apt)
└── README.md
```

---

## 🛠️ Tech Stack

| Component       | Technology                                  |
|-----------------|---------------------------------------------|
| Orchestration   | Apache Airflow (via Astronomer Runtime 3.2) |
| Database        | PostgreSQL 13                               |
| Containerisation| Docker + Docker Compose                     |
| API Source      | NASA APOD API                               |
| Language        | Python 3.9+                                 |
| Airflow Hooks   | `PostgresHook`, `SimpleHttpOperator`        |
| Task Pattern    | TaskFlow API (`@task` decorator)            |

---

## ⚙️ Prerequisites

Before you start, ensure you have the following installed:

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) (v20+)
- [Astronomer CLI](https://docs.astronomer.io/astro/cli/install-cli) (`astro`)
- A [NASA API Key](https://api.nasa.gov/) (free to register)
- Git

---

## 🚀 Local Development Setup

### 1. Clone the Repository

```bash
git clone https://github.com/navneetsxngh/ETL-Pipelines-using-Airflow.git
cd ETL-Pipelines-using-Airflow
```

### 2. Start the Airflow Environment

```bash
astro dev start
```

This spins up the Airflow webserver, scheduler, triggerer, and the Astronomer-managed Postgres metadata database on the `airflow_default` Docker network.

### 3. Start the PostgreSQL Container

In a separate terminal, start the application Postgres instance:

```bash
docker-compose up -d
```

> This creates a `postgres_db` container attached to the `airflow_default` network, making it accessible from within Airflow tasks.

### 4. Configure Airflow Connections

Open the Airflow UI at **http://localhost:8080** (default credentials: `admin` / `admin`).

**Add the HTTP connection (NASA API):**

| Field       | Value                        |
|-------------|------------------------------|
| Conn ID     | `nasa_apod_api`              |
| Conn Type   | `HTTP`                       |
| Host        | `https://api.nasa.gov`       |
| Extra       | `{"api_key": "YOUR_API_KEY"}`|

**Add the Postgres connection:**

| Field       | Value              |
|-------------|--------------------|
| Conn ID     | `postgres_default` |
| Conn Type   | `Postgres`         |
| Host        | `postgres_db`      |
| Schema      | `postgres`         |
| Login       | `postgres`         |
| Password    | `postgres`         |
| Port        | `5432`             |

### 5. Enable and Trigger the DAG

In the Airflow UI, locate `nasa_apod_etl` (or your DAG ID), toggle it **ON**, and click **Trigger DAG**.

---

## ☁️ AWS Deployment Guide

### Recommended Architecture on AWS

```
GitHub Repo
     │
     ▼
AWS ECR (Docker Image)
     │
     ▼
Amazon MWAA (Managed Workflows for Apache Airflow)
     │
     ├──▶ Amazon RDS (PostgreSQL) — Application Database
     └──▶ Amazon S3 (DAG storage + logs)
```

### Step-by-Step AWS Deployment

#### Step 1 — Set Up Amazon RDS (PostgreSQL)

1. Go to **AWS Console → RDS → Create Database**.
2. Choose **PostgreSQL 13**, instance class `db.t3.micro` (or higher for production).
3. Set DB name: `postgres`, username: `postgres`, password: `<your-secure-password>`.
4. Enable **Public Access** only if testing; use **VPC Security Groups** for production.
5. Note down the **RDS Endpoint URL**.

#### Step 2 — Set Up Amazon S3 for DAGs and Logs

```bash
aws s3 mb s3://your-airflow-bucket
aws s3 mb s3://your-airflow-bucket/dags
aws s3 mb s3://your-airflow-bucket/logs

# Upload your DAGs
aws s3 cp dags/ s3://your-airflow-bucket/dags/ --recursive
```

#### Step 3 — Deploy Amazon MWAA (Managed Airflow)

1. Go to **AWS Console → Amazon MWAA → Create Environment**.
2. Name: `nasa-apod-etl-airflow`
3. S3 Bucket: `s3://your-airflow-bucket`
4. DAGs folder: `dags/`
5. **Requirements file**: Upload `requirements.txt` to S3 and point MWAA to it.
6. Airflow version: `2.9.x` (or latest supported)
7. VPC: Use the same VPC as your RDS instance.
8. Create the environment (takes ~20–30 minutes).

#### Step 4 — Set Up Environment Variables / Connections in MWAA

Once the environment is active, open the **Airflow UI via MWAA** and add the same connections as in local setup, replacing `postgres_db` host with your **RDS Endpoint**.

#### Step 5 — (Optional) Push Custom Docker Image to ECR

If you customise the Dockerfile:

```bash
# Authenticate Docker with ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin <account-id>.dkr.ecr.us-east-1.amazonaws.com

# Build and push
docker build -t nasa-apod-etl .
docker tag nasa-apod-etl:latest <account-id>.dkr.ecr.us-east-1.amazonaws.com/nasa-apod-etl:latest
docker push <account-id>.dkr.ecr.us-east-1.amazonaws.com/nasa-apod-etl:latest
```

#### Step 6 — CI/CD with GitHub Actions (Optional)

Create `.github/workflows/deploy.yml`:

```yaml
name: Deploy DAGs to S3

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Sync DAGs to S3
        run: |
          aws s3 sync dags/ s3://your-airflow-bucket/dags/ --delete
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          AWS_DEFAULT_REGION: us-east-1
```

---

## 🧪 Running Tests

```bash
# Run DAG integrity tests locally
astro dev pytest tests/
```

Or run directly with pytest inside the container:

```bash
docker exec -it <airflow-scheduler-container> pytest tests/dags/
```

---

## 🔧 Environment Variables Reference

| Variable                 | Description                        | Default     |
|--------------------------|------------------------------------|-------------|
| `POSTGRES_USER`          | PostgreSQL username                | `postgres`  |
| `POSTGRES_PASSWORD`      | PostgreSQL password                | `postgres`  |
| `POSTGRES_DB`            | PostgreSQL database name           | `postgres`  |
| `NASA_API_KEY`           | NASA APOD API key                  | `DEMO_KEY`  |

> ⚠️ **Never commit secrets to version control.** Use `.env` files locally and AWS Secrets Manager / MWAA environment variables in production.

---

## 📋 DAG Details

| Property          | Value                     |
|-------------------|---------------------------|
| DAG ID            | `nasa_apod_etl`           |
| Schedule          | `@daily`                  |
| Start Date        | Configurable              |
| Catchup           | `False`                   |
| Retries           | `1`                       |

---

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch: `git checkout -b feature/your-feature`
3. Commit your changes: `git commit -m 'Add some feature'`
4. Push to the branch: `git push origin feature/your-feature`
5. Open a Pull Request

---

## 📄 License

This project is open-source and available under the [MIT License](LICENSE).

---

## 👤 Author

**Navneet Singh**  
[GitHub](https://github.com/navneetsxngh)