<div align="center">
  
# 🚀 KESKUR: Distributed File Transaction Protocol
**Technical Architecture & System Design Whitepaper**

![FastAPI](https://img.shields.io/badge/FastAPI-005571?style=for-the-badge&logo=fastapi)
![Next JS](https://img.shields.io/badge/Next-black?style=for-the-badge&logo=next.js&logoColor=white)
![React Native](https://img.shields.io/badge/react_native-%2320232a.svg?style=for-the-badge&logo=react&logoColor=%2361DAFB)
![Postgres](https://img.shields.io/badge/postgres-%23316192.svg?style=for-the-badge&logo=postgresql&logoColor=white)
![Redis](https://img.shields.io/badge/redis-%23DD0031.svg?style=for-the-badge&logo=redis&logoColor=white)

**Date:** February 28, 2026  |  **Version:** 2.1.1 (Production Release)  |  **Status:** Active

</div>

---

## 1. Executive Summary

**FileFlow** is a next-generation file exchange platform that reimagines digital file sharing through the lens of financial transaction protocols. Unlike traditional cloud storage systems (Dropbox, Google Drive) that focus on *synchronization*, FileFlow focuses on **transactional ownership transfer**.

The system implements a "UPI for Files" architecture, where every file exchange is an atomic, verified transaction with an immutable audit trail. This approach ensures security, traceability, and zero-redundancy data management, making it ideal for high-trust environments.

---

## 2. System Architecture

FileFlow employs a **Modern Headless Microservices Architecture**, decoupling the frontend interfaces from the core logic and storage layers.

### 2.1 High-Level Component Diagram

```mermaid
graph TD
    User((User))
    
    subgraph "Presentation Layer (Clients)"
        Mobile["Mobile App<br>(React Native + Expo)"]
        Web["Web Dashboard<br>(Next.js 14 + Shadcn)"]
    end
    
    subgraph "Edge/Network Layer"
        CDN["Cloudflare CDN"]
        LB["Load Balancer / Nginx"]
        WAF["Web Application Firewall"]
    end
    
    subgraph "Application Layer (Backend)"
        API["FastAPI Core Server"]
        Auth["Authentication Service"]
        Trans["Transaction Engine"]
        Worker["Celery Async Workers"]
    end
    
    subgraph "Data Persistence Layer"
        PG[("PostgreSQL 15<br>Metadata & Ledger")]
        Redis[("Redis 7<br>Cache & Session Store")]
        B2[("Backblaze B2<br>Immutable Object Store")]
        ES[("Elasticsearch<br>Index & Search")]
    end
    
    User <--> Mobile & Web
    Mobile & Web <--> LB
    LB <--> API
    
    API <--> Auth
    API <--> Trans
    API <--> PG
    API <--> Redis
    API --> Worker
    
    API -.->|Presigned URLs| B2
    Worker --> ES
    Worker --> B2
```

### 2.2 Technology Stack Specification

| Component | Technology | Rationale |
| :--- | :--- | :--- |
| **Mobile Client** | **React Native (0.74)** + Expo (51) | Cross-platform native performance, OTA updates, native share sheets. |
| **Web Client** | **Next.js 14** + TypeScript | Server-Side Rendering (SSR) for speed, SEO, strict type safety. |
| **Styling** | **Tailwind CSS** + shadcn/ui | Unified design system across Web and Mobile. |
| **Backend API** | **FastAPI** (Python 3.10) | High-performance async I/O, auto-generated OpenAPI docs, Pydantic validation. |
| **Database** | **PostgreSQL 15** | ACID compliance is essential for the transactional ledger system. |
| **Object Storage** | **Backblaze B2** | S3-compatible, immutable object locking, effectively infinite scalability ($0.005/GB). |
| **Caching** | **Redis 7** | Sub-millisecond latency for OTPs, session management, and rate limiting. |
| **PDF Engine** | **PyMuPDF & ReportLab** | Used server-side for flattening e-signatures visually onto PDFs. |
| **ORM** | **SQLAlchemy** (Async) | Robust database abstraction with full async support. |

---

## 3. Data Models & Schema Design (ERD)

The database schema is designed to enforce strict referential integrity. All primary keys are **UUIDv4**.

```mermaid
erDiagram
    USERS ||--o{ FILES : owns
    USERS ||--o{ FOLDERS : owns
    USERS ||--o{ SHARES_SENT : initiates
    USERS ||--o{ SHARES_RECEIVED : receives
    
    FOLDERS ||--o{ FILES : contains
    FOLDERS ||--o| FOLDERS : parent
    
    FILES ||--|{ SHARES : transferred_in
    
    USERS {
        uuid id PK
        string email UK
        string phone UK
        bigint storage_quota_bytes
        bigint storage_used_bytes
    }
    
    FILES {
        uuid id PK
        uuid owner_user_id FK
        uuid folder_id FK
        string filename
        string storage_key "Canonical B2 Path"
        bigint size_bytes
        string mime_type
        enum status "pending, uploaded, deleted"
    }
    
    SHARES {
        uuid id PK
        string transaction_id UK "TXN-1234"
        uuid sender_user_id FK
        uuid recipient_user_id FK
        uuid file_id FK
        timestamp created_at
    }
    
    FOLDERS {
        uuid id PK
        uuid owner_user_id FK
        string name
        string parent_folder_id FK
    }
```

---

## 4. API Specification & File Flows (v1)

The system exposes a RESTful API with strict adherence to HTTP semantics and JSON structure.

### 4.1 Authentication Module (Passwordless OTP)
*   **POST** `/api/v1/auth/send-otp`
    *   **Payload**: `phone_number`
    *   **Mechanism**: Generates 6-digit code, stores in Redis (TTL 300s), dispatches via Twilio SMS.
*   **POST** `/api/v1/auth/verify-otp`
    *   **Mechanism**: Validates OTP. If phone is new, automatically provisions user account + default folders. Returns stateless JWT `access_token`.

### 4.2 Secure Direct-to-Cloud Upload Pipeline
To prevent server bottlenecks, FileFlow uses direct-to-cloud uploads.
1.  **POST** `/api/v1/files/upload/init`
    *   **Logic**: Verifies User Quota. Generates unique `storage_key`.
    *   **Output**: Backblaze B2 `upload_url` and `authorization_token`.
2.  **Client Action**: Client performs `PUT` request directly to the `upload_url`, bypassing the FastAPI server.
3.  **POST** `/api/v1/files/upload/{id}/complete`
    *   **Logic**: Backend updates DB status to `uploaded` and deducts user quota.

### 4.3 Transaction Engine (Zero-Copy Sharing)
*   **POST** `/api/v1/shares`
    *   **Input**: `{ file_id: "...", recipient_phone: "..." }`
    *   **Process (Atomic Transaction)**:
        1.  **Lock**: Verify Sender ownership. Verify Recipient existence.
        2.  **Execute**: 
            *   Create specific new `File` pointer for Recipient with the exact same `storage_key` as Sender (Zero-Copy).
            *   Create `Share` Ledger Entry.
            *   Update Recipient Quota.
        3.  **Commit**: Return Transaction Receipt.

### 4.4 Internal Native e-Signatures
Users can sign PDF documents directly without external tools.
1. User draws a signature on the Mobile/Web UI, saved as a Base64 image to B2.
2. User overlays a signature box over a PDF viewer and hits "Apply".
3. **POST** `/files/{id}/sign`: Backend pulls the raw PDF to RAM, uses `ReportLab` to draw the image and timestamp, merges it with `PyMuPDF`, and re-uploads it as a brand new unique locked file.

---

## 5. Security Protocols

### 5.1 Zero-Trust Architecture
*   **Presigned URLs**: The API Server never handles file binary data for downloads. All file viewing invokes `GET /files/{id}/view` returning a temporary Presigned URL valid for exactly 1 hour. There are no static links.
*   **JWT Authentication**: Stateless authentication using RSA/HS256 signed JSON Web Tokens.
*   **Rate Limiting**: Intelligent throttling via Redis (using `slowapi`) to prevent DDoS attacks.

### 5.2 Data Protection
*   **At-Rest**: Files in B2 are encrypted.
*   **Input Sanitization**: Pydantic models validate and sanitize all payloads before database interaction.
*   **Immutability**: Once a file is signed or sent, the zero-copy logic ensures it cannot be maliciously edited without wiping the database record.

---

## 6. Scalability & Deployment

### 6.1 Scaling Strategy
*   **Stateless API**: FastAPI relies completely on Postgres/Redis, allowing the API instances to horizontally scale infinitely behind an Nginx Load Balancer or within Kubernetes.
*   **CDN Integration**: Future-ready edge caching for public assets.

### 6.2 Local Development Setup
**Backend (`backend/`)**
```bash
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt
# Requires: DATABASE_URL, REDIS_URL, B2_KEY_ID, B2_APPLICATION_KEY, TWILIO env vars
uvicorn app.main:app --reload --port 8000
```

**Web Dashboard (`web/`)**
```bash
npm install
# Requires: NEXT_PUBLIC_API_URL=http://localhost:8000/api/v1
npm run dev
```

**Mobile App (`mobile/`)**
```bash
npm install
# Update /src/services/api.js with local network IP or Ngrok URL
npx expo start -c
```
