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
- curl (for API testing)

### 1. Start the Services

```bash
docker-compose up -d
```

This will start:
- **PostgreSQL 14** on port 5432
- **EHRbase** on port 8080

### 2. Verify Services are Running

```bash
docker ps
```

You should see both `ehrdb` and `ehrbase` containers running.

### 3. Check EHRbase Status

```bash
curl -u ehrbase-admin:SuperSecretAdminPassword \
  http://localhost:8080/ehrbase/rest/status
```

## Template Management

### Upload the Blood Pressure Template

The blood pressure template is already included in the `templates/` directory.

```bash
curl -X POST \
  -H "Content-Type: application/xml" \
  -u ehrbase-admin:SuperSecretAdminPassword \
  --data-binary @templates/blood_pressure.opt \
  http://localhost:8080/ehrbase/rest/openehr/v1/definition/template/adl1.4
```

### List Available Templates

```bash
curl -u ehrbase-admin:SuperSecretAdminPassword \
  http://localhost:8080/ehrbase/rest/openehr/v1/definition/template/adl1.4 | jq .
```

## Creating an EHR (Electronic Health Record)

Create an EHR for a patient:

```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -u ehrbase-user:SuperSecretPassword \
  -d '{
    "_type": "EHR_STATUS",
    "archetype_node_id": "openEHR-EHR-EHR_STATUS.generic.v1",
    "name": {
      "value": "EHR Status"
    },
    "subject": {
      "external_ref": {
        "id": {
          "_type": "GENERIC_ID",
          "value": "patient-123",
          "scheme": "id_scheme"
        },
        "namespace": "examples",
        "type": "PERSON"
      }
    },
    "is_modifiable": true,
    "is_queryable": true
  }' \
  http://localhost:8080/ehrbase/rest/openehr/v1/ehr | jq .
```

### Get EHR ID by Patient ID

```bash
curl -u ehrbase-user:SuperSecretPassword \
  "http://localhost:8080/ehrbase/rest/openehr/v1/ehr?subject_id=patient-123&subject_namespace=examples" | jq -r '.ehr_id.value'
```

## Recording Vital Signs

### Create a Blood Pressure Measurement

Replace `YOUR_EHR_ID` with the actual EHR ID from the previous step:

```bash
curl -X POST \
  -u ehrbase-user:SuperSecretPassword \
  -H "Content-Type: application/json" \
  -H "Prefer: return=representation" \
  -d '{
    "blood_pressure/blood_pressure:0/any_event:0/systolic|magnitude": 118,
    "blood_pressure/blood_pressure:0/any_event:0/systolic|unit": "mm[Hg]",
    "blood_pressure/blood_pressure:0/any_event:0/diastolic|magnitude": 76,
    "blood_pressure/blood_pressure:0/any_event:0/diastolic|unit": "mm[Hg]",
    "blood_pressure/blood_pressure:0/any_event:0/time": "2025-11-07T14:00:00Z",
    "ctx/language": "en",
    "ctx/territory": "US",
    "ctx/composer_name": "Nurse Jane"
  }' \
  "http://localhost:8080/ehrbase/rest/openehr/v1/ehr/YOUR_EHR_ID/composition?templateId=blood_pressure&format=FLAT" | jq .
```

### Example with Different Values

```bash
# High Blood Pressure Example (140/90)
curl -X POST \
  -u ehrbase-user:SuperSecretPassword \
  -H "Content-Type: application/json" \
  -H "Prefer: return=representation" \
  -d '{
    "blood_pressure/blood_pressure:0/any_event:0/systolic|magnitude": 140,
    "blood_pressure/blood_pressure:0/any_event:0/systolic|unit": "mm[Hg]",
    "blood_pressure/blood_pressure:0/any_event:0/diastolic|magnitude": 90,
    "blood_pressure/blood_pressure:0/any_event:0/diastolic|unit": "mm[Hg]",
    "blood_pressure/blood_pressure:0/any_event:0/time": "2025-11-08T10:30:00Z",
    "ctx/language": "en",
    "ctx/territory": "US",
    "ctx/composer_name": "Dr. Smith"
  }' \
  "http://localhost:8080/ehrbase/rest/openehr/v1/ehr/YOUR_EHR_ID/composition?templateId=blood_pressure&format=FLAT" | jq .
```

## Querying Data with AQL

### Query All Blood Pressure Readings for a Patient

```bash
curl -X POST \
  -u ehrbase-user:SuperSecretPassword \
  -H "Content-Type: application/json" \
  -d '{
    "q": "SELECT c/uid/value as composition_id, c/context/start_time as measurement_time, o/data[at0001]/events[at0006]/data[at0003]/items[at0004]/value/magnitude as systolic, o/data[at0001]/events[at0006]/data[at0003]/items[at0005]/value/magnitude as diastolic FROM EHR e CONTAINS COMPOSITION c CONTAINS OBSERVATION o[openEHR-EHR-OBSERVATION.blood_pressure.v2] WHERE e/ehr_id/value = '\''YOUR_EHR_ID'\''"
  }' \
  "http://localhost:8080/ehrbase/rest/openehr/v1/query/aql" | jq .
```

### Query Latest Blood Pressure Reading

```bash
curl -X POST \
  -u ehrbase-user:SuperSecretPassword \
  -H "Content-Type: application/json" \
  -d '{
    "q": "SELECT TOP 1 c/context/start_time as measurement_time, o/data[at0001]/events[at0006]/data[at0003]/items[at0004]/value/magnitude as systolic, o/data[at0001]/events[at0006]/data[at0003]/items[at0005]/value/magnitude as diastolic FROM EHR e CONTAINS COMPOSITION c CONTAINS OBSERVATION o[openEHR-EHR-OBSERVATION.blood_pressure.v2] WHERE e/ehr_id/value = '\''YOUR_EHR_ID'\'' ORDER BY c/context/start_time DESC"
  }' \
  "http://localhost:8080/ehrbase/rest/openehr/v1/query/aql" | jq .
```

### Query High Blood Pressure Readings (Systolic > 130)

```bash
curl -X POST \
  -u ehrbase-user:SuperSecretPassword \
  -H "Content-Type: application/json" \
  -d '{
    "q": "SELECT c/context/start_time as measurement_time, o/data[at0001]/events[at0006]/data[at0003]/items[at0004]/value/magnitude as systolic, o/data[at0001]/events[at0006]/data[at0003]/items[at0005]/value/magnitude as diastolic FROM EHR e CONTAINS COMPOSITION c CONTAINS OBSERVATION o[openEHR-EHR-OBSERVATION.blood_pressure.v2] WHERE o/data[at0001]/events[at0006]/data[at0003]/items[at0004]/value/magnitude > 130"
  }' \
  "http://localhost:8080/ehrbase/rest/openehr/v1/query/aql" | jq .
```

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