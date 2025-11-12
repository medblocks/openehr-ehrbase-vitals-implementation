# OpenEHR Vitals - EHRbase Tutorial

A complete tutorial for setting up and using EHRbase with OpenEHR to store and query patient vital signs data (Blood Pressure).

## Overview

This project demonstrates how to:
- Set up EHRbase (OpenEHR clinical data repository) with PostgreSQL
- Upload OpenEHR templates
- Create Electronic Health Records (EHRs)
- Store vital signs data (Blood Pressure measurements)
- Query clinical data using AQL (Archetype Query Language)

## Architecture

```
┌─────────────────┐
│   EHRbase API   │  Port 8080
│  (OpenEHR CDR)  │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   PostgreSQL    │  Port 5432
│    Database     │
└─────────────────┘
```

## Quick Start

### Prerequisites

- Docker and Docker Compose installed
- Postman (desktop app or web workspace)

### 1. Start the Services

```bash
docker-compose up -d
```

This will start:
- **PostgreSQL 14** on port 5432
- **EHRbase** on port 8080

### 2. Enable UUID Support in Postgres

```bash
docker exec ehrdb psql -U ehrbase -d ehrbase -c "CREATE EXTENSION IF NOT EXISTS \"uuid-ossp\";"
```

This creates the `uuid_generate_v4()` function required by EHRbase migrations.

### 3. Verify Services are Running

```bash
docker ps
```

You should see both `ehrdb` and `ehrbase` containers running.

### 4. Import the Postman Assets

- In Postman, click **Import → Files** and choose:
  - `postman/OpenEHR Vitals Tutorial.postman_collection.json`
  - `postman/openEHR Local.postman_environment.json`
- Select the `OpenEHR Local` environment and confirm the variables:
  - `base_url` should remain `http://localhost:8080/ehrbase/rest`.
  - Update `patient_id`, `composer_name`, and `measurement_time` if you want different defaults.

### 5. Check EHRbase Status

- Open the `Check EHRbase Status` request in the collection and press **Send**.
- Expect a `200 OK` response confirming the server is healthy.

## Template Management

### Upload the Blood Pressure Template

- Run `Upload Blood Pressure Template`.
- Ensure the body points at `templates/blood_pressure.opt`. Postman will stream the file automatically.
- Expect a `201 Created` or `204 No Content` response.

### List Available Templates

- Run `List Templates` to confirm the upload. A `200 OK` response with the template metadata indicates success.

## Creating an EHR (Electronic Health Record)

Create an EHR for a patient without touching the CLI:

- Run `Create EHR` to register the patient. The request inherits user credentials from the collection.
- Run `Find EHR by Subject`. The test script stores the returned `ehr_id` in the environment for later requests. You can also copy it manually from the response.

## Recording Vital Signs

### Create a Blood Pressure Measurement

- Run `Create Blood Pressure Measurement` to log a baseline reading. Variables such as `measurement_time` and `composer_name` come from the environment—change them there or edit the body before sending.
- Run `Create High BP Measurement` for a 140/90 example. Feel free to duplicate and tailor additional scenarios.

## Querying Data with AQL

### Query All Blood Pressure Readings for a Patient

- Run `AQL – All Blood Pressure Readings` to retrieve every measurement captured for the current `ehr_id`.

## Authentication

The system uses HTTP Basic Authentication with two sets of credentials:

- **Username**: `ehrbase-user`
- **Password**: `SuperSecretPassword`

## Project Structure

```
openerh-vitals/
├── docker-compose.yml          # Docker services configuration
├── init-db.sh/
│   └── init-db.sh             # Database initialization script
├── templates/
│   └── blood_pressure.opt     # Blood pressure OpenEHR template
└── README.md                   # This file
```

## Troubleshooting

### Check Container Logs

```bash
# EHRbase logs
docker logs openerh-vitals-ehrbase-1

# PostgreSQL logs
docker logs openerh-vitals-ehrdb-1
```

### Restart Services

```bash
docker-compose down
docker-compose up -d
```

### Reset Database (Warning: Deletes all data)

```bash
docker-compose down -v
docker-compose up -d
```