# Bulk Certificate Generator

A simple backend API for generating course completion certificates in bulk.

The application accepts a list of recipients, creates a certificate generation job, generates PDF certificates in the background, tracks the progress, and allows completed certificates to be downloaded.

This project is designed as a simple backend/placement project with a focus on clean code, REST APIs, database usage, validation, PDF generation, and testing.

---

## Features

- Create bulk certificate generation jobs
- Validate recipient information
- Generate one PDF certificate per recipient
- Generate certificates in the background
- Track job status and progress
- Track successful and failed recipients
- Download generated certificates
- Store job and certificate information in SQLite
- REST API using FastAPI
- Automatic API documentation with Swagger
- Basic automated tests with Pytest

---

## Technology Stack

| Technology | Purpose |
|---|---|
| Python | Programming language |
| FastAPI | REST API framework |
| SQLite | Database |
| SQLAlchemy | Database ORM |
| Pydantic | Request validation |
| ReportLab | PDF certificate generation |
| Pytest | Testing |

---

## Project Flow

```text
Client
  |
  | POST /api/jobs
  v
FastAPI
  |
  +---- Pydantic Validation
  |
  +---- SQLite / SQLAlchemy
  |
  +---- Background Task
             |
             v
      Certificate Service
             |
             v
          ReportLab
             |
             v
       PDF Certificate
```

### Request Flow

```text
1. Client sends certificate request
            |
            v
2. FastAPI validates the request
            |
            v
3. Job is created in SQLite
            |
            v
4. API returns Job ID
            |
            v
5. Background task starts
            |
            v
6. Certificates are generated
            |
            v
7. Job progress is updated
            |
            v
8. Client checks job status
            |
            v
9. Client downloads completed certificate
```

---

## Project Structure

```text
bulk-certificate-generator/
│
├── app/
│   ├── main.py
│   ├── database.py
│   ├── models.py
│   ├── schemas.py
│   ├── jobs.py
│   └── certificate_service.py
│
├── certificates/
│
├── tests/
│   ├── test_jobs.py
│   ├── test_certificate.py
│   └── test_validation.py
│
├── requirements.txt
├── README.md
└── .gitignore
```

### Main Files

#### `app/main.py`

Creates the FastAPI application and registers the API routes.

#### `app/database.py`

Contains the SQLite database configuration and SQLAlchemy session setup.

#### `app/models.py`

Contains the database models:

- Job
- Certificate

#### `app/schemas.py`

Contains Pydantic request and response schemas and input validation.

#### `app/jobs.py`

Contains the job-related API endpoints and background processing logic.

#### `app/certificate_service.py`

Responsible for creating certificate PDFs using ReportLab.

#### `certificates/`

Stores generated PDF certificates.

#### `tests/`

Contains automated tests.

---

# Database Design

The project uses two main tables.

## Job

Stores information about a certificate generation job.

```text
Job
-------------------------
id
course_name
completion_date
status
total_count
completed_count
failed_count
created_at
```

## Certificate

Stores information about each recipient.

```text
Certificate
-------------------------
id
job_id
name
email
status
certificate_id
certificate_path
error_message
created_at
```

Relationship:

```text
Job
 |
 | 1
 |
 | many
 v
Certificate
```

One job can have multiple certificate records.

---

# Job Status

The application uses four job states:

```text
PENDING
PROCESSING
COMPLETED
FAILED
```

Flow:

```text
PENDING
   |
   v
PROCESSING
   |
   +----> COMPLETED
   |
   +----> FAILED
```

### PENDING

The job has been created but processing has not started.

### PROCESSING

Certificates are currently being generated.

### COMPLETED

At least one certificate was generated successfully.

A completed job can still contain some failed recipients.

### FAILED

All recipients failed during certificate generation.

---

# Progress

Progress is calculated using:

```text
progress = ((completed + failed) / total) * 100
```

For example:

```text
Total       = 5
Completed   = 3
Failed      = 1

Progress = ((3 + 1) / 5) * 100
         = 80%
```

When every recipient has either succeeded or failed:

```text
Progress = 100%
```

---

# Certificate Generation

Each certificate is generated as a PDF using ReportLab.

The certificate contains:

- Certificate of Completion
- Recipient name
- Course name
- Completion date
- Unique certificate ID
- Issuer/signature text

Example certificate ID:

```text
CERT-2026-000001
CERT-2026-000002
CERT-2026-000003
```

PDF files are stored in:

```text
certificates/
```

Example:

```text
certificates/CERT-2026-000001.pdf
```

The database stores the PDF file path rather than the PDF binary data.

---

# API Endpoints

## 1. Create Certificate Job

```http
POST /api/jobs
```

### Request

```json
{
  "course_name": "Python Backend Development",
  "completion_date": "2026-10-08",
  "recipients": [
    {
      "name": "Alice Johnson",
      "email": "alice@example.com"
    },
    {
      "name": "Bob Smith",
      "email": "bob@example.com"
    }
  ]
}
```

### Response

```json
{
  "job_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "PENDING",
  "total": 2,
  "completed": 0,
  "failed": 0
}
```

The API returns the job ID immediately.

Certificate generation happens in the background.

---

# 2. Get Job Status

```http
GET /api/jobs/{job_id}
```

### Example Response

```json
{
  "job_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "COMPLETED",
  "total": 2,
  "completed": 2,
  "failed": 0,
  "progress": 100,
  "recipients": [
    {
      "name": "Alice Johnson",
      "email": "alice@example.com",
      "status": "COMPLETED",
      "certificate_id": "CERT-2026-000001"
    },
    {
      "name": "Bob Smith",
      "email": "bob@example.com",
      "status": "COMPLETED",
      "certificate_id": "CERT-2026-000002"
    }
  ]
}
```

---

# 3. Download Certificate

```http
GET /api/certificates/{certificate_id}
```

Returns the generated PDF certificate.

### Possible responses

```text
200 OK
```

Certificate exists and is ready.

```text
404 Not Found
```

Certificate does not exist.

```text
409 Conflict
```

Certificate exists but generation is not complete.

---

# Validation

The API validates:

- Course name
- Recipient name
- Email address
- Completion date
- Recipient count
- Duplicate emails

The following are rejected:

- Empty course name
- Empty recipient list
- More than 1000 recipients
- Invalid email
- Duplicate emails in the same request
- Missing recipient name
- Missing email
- Future completion date

Invalid requests return:

```text
422 Unprocessable Entity
```

The same email can be used in different jobs.

---

# Failure Handling

A failure for one recipient does not stop the remaining certificates.

Example:

```text
Alice      -> COMPLETED
Bob        -> FAILED
Charlie    -> COMPLETED
```

The job can still finish as:

```text
COMPLETED
```

The failed recipient contains an error message.

If all recipients fail:

```text
FAILED
```

Simple processing logic:

```python
for recipient in recipients:
    try:
        generate_certificate(recipient)
        mark_completed(recipient)
    except Exception:
        mark_failed(recipient)
```

---

# Background Processing

FastAPI `BackgroundTasks` is used to generate certificates after the API returns the job ID.

This keeps the example simple.

The flow is:

```text
POST /api/jobs
       |
       v
Create Job
       |
       v
Return Job ID
       |
       v
Background Task
       |
       v
Generate PDFs
       |
       v
Update Database
```

### Why BackgroundTasks?

Generating many PDF files can take time.

Instead of making the client wait for every certificate:

1. The API creates the job.
2. The API returns the job ID.
3. Certificate generation continues in the background.
4. The client can check the job status.

For this simple project, FastAPI `BackgroundTasks` is sufficient.

For a large production system, a proper distributed task queue could be introduced.

---

# Setup

## 1. Clone the repository

```bash
git clone <your-repository-url>
cd bulk-certificate-generator
```

Replace `<your-repository-url>` with the actual GitHub repository URL.

---

## 2. Create a virtual environment

### Windows

```bash
python -m venv venv
venv\Scripts\activate
```

### Linux/macOS

```bash
python3 -m venv venv
source venv/bin/activate
```

---

## 3. Install dependencies

```bash
pip install -r requirements.txt
```

---

# Run the Application

Start FastAPI using:

```bash
uvicorn app.main:app --reload
```

The application will normally run at:

```text
http://127.0.0.1:8000
```

---

# API Documentation

FastAPI automatically provides Swagger documentation.

Open:

```text
http://127.0.0.1:8000/docs
```

You can test the APIs directly from Swagger.

OpenAPI specification:

```text
http://127.0.0.1:8000/openapi.json
```

---

# Testing

Run the test suite using:

```bash
pytest
```

The tests cover important functionality such as:

- Job creation
- Request validation
- Certificate generation
- Job status
- Certificate download
- Invalid job handling

---

# HTTP Error Codes

| Status | Meaning |
|---|---|
| `200` | Successful request |
| `404` | Job or certificate not found |
| `409` | Certificate is not ready |
| `422` | Invalid input |
| `500` | Unexpected server error |

Internal stack traces and filesystem paths should not be exposed to API users.

---

# Design Decisions

## Why FastAPI?

FastAPI is lightweight, easy to use, and automatically provides API documentation through Swagger.

It also supports background tasks and Pydantic validation.

## Why SQLite?

SQLite is simple and requires no separate database server.

It is suitable for this small project and local development.

For a larger production application, PostgreSQL would be a better choice.

## Why SQLAlchemy?

SQLAlchemy provides a clean way to work with the database using Python objects and relationships.

## Why Pydantic?

Pydantic makes request validation simple and integrates naturally with FastAPI.

## Why ReportLab?

ReportLab allows PDF certificates to be generated directly using Python.

## Why BackgroundTasks?

Certificate generation may take time, especially when many recipients are provided.

BackgroundTasks allows the API to return a job ID while processing continues.

---

# Limitations

This project intentionally keeps the architecture simple.

Current limitations include:

- SQLite is intended for simple/local usage.
- FastAPI BackgroundTasks is not a distributed job queue.
- Certificates are stored on the local filesystem.
- No authentication is implemented.
- No email delivery is implemented.
- No frontend is included.

These limitations are acceptable for this student/placement project.

---

# Future Improvements

Possible future improvements:

- PostgreSQL database
- Celery or another task queue
- Redis
- Cloud/object storage such as S3
- User authentication
- Email certificate delivery
- Certificate verification endpoint
- Certificate revocation
- Admin dashboard
- Pagination
- Rate limiting
- Monitoring and metrics

---

# Interview Explanation

A simple explanation of the project:

> "This project is a FastAPI-based bulk certificate generation system. A client sends a course name, completion date, and a list of recipients through a REST API. The API validates the request and creates a job in SQLite using SQLAlchemy. It immediately returns a job ID and processes certificate generation in the background using FastAPI BackgroundTasks. Each certificate is generated as a PDF using ReportLab and stored in the certificates folder. The database tracks the status of each recipient and the overall job progress. Another API allows the client to check the job status, and a separate endpoint allows completed certificates to be downloaded."

---

# Learning Outcomes

This project demonstrates understanding of:

- REST API development
- FastAPI
- HTTP methods and status codes
- Request validation
- Pydantic
- SQL databases
- SQLAlchemy ORM
- Database relationships
- Background processing
- PDF generation
- File handling
- Error handling
- Automated testing
- API documentation

---

# Author

**Guruprasath K C**

B.E. Electrical & Electronics Engineering

VSB Engineering College, Karur
