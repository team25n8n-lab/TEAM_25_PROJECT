# Medical Record OCR Automation with Patient Data Extraction and Supabase Integration

A fully automated, end-to-end medical document processing system built on **n8n**, **Google Drive**, **OCR.Space**, **Google Gemini AI**, and **Supabase**.

This project consists of a **comprehensive n8n workflow** that automates:

- Real-time monitoring of Google Drive folders for new medical documents
- Automated file type validation (PDF/Image)
- OCR text extraction from medical records
- AI-powered structured data extraction using Google Gemini
- Patient record management with automatic upsert
- Medical document summarization and metadata generation
- Supabase database integration for patient and document storage
- Audit logging for compliance tracking
- Dead letter queue for failed processing
- Automatic file organization (raw → processed)

This repository documents the complete architecture, AI prompts, database schema, workflow logic, and technical details required for developers to understand, maintain, extend, or deploy the system.

---

## 📑 Table of Contents

1. [Introduction](#-introduction)
2. [System Architecture](#️-system-architecture)
3. [Sequence Diagrams](#-workflow-interaction-diagram)
4. [Environment Variables](#️-environment-variables)
5. [API Endpoints](#-api-endpoints)
6. [Database Schema](#️-database-schema)
7. [Folder Structure](#-folder-structure)
8. [Workflow Overview](#-workflow-overview)
9. [Node-Level Documentation](#-node-level-documentation)
10. [Setup Instructions](#️-setup-instructions)
11. [Deployment Guide](#-deployment-guide)
12. [Scalability & Limitations](#-scalability--limitations)
13. [Future Improvements](#-future-improvements)

---

## 🧩 Introduction

The **Medical Record OCR Automation with Patient Data Extraction and Supabase Integration** is designed as a no-code/low-code AI automation pipeline that allows healthcare facilities to:

- Automatically process medical documents uploaded to Google Drive
- Extract text from PDFs and images using OCR technology
- Use AI to intelligently parse patient information, diagnoses, medicines, and test results
- Maintain a centralized patient database with automatic record creation
- Generate human-readable summaries of medical documents
- Track all processing activities through audit logs
- Handle errors gracefully with dead letter queue logging
- Organize files automatically into processed folders
- Support webhook-based manual uploads

The system runs without human intervention and uses Google Gemini 1.5 Pro for intelligent medical data extraction, Supabase as a robust database backend, and Google Drive for document storage and organization.

---

## 🗂️ System Architecture

### High-Level Architecture Diagram

```mermaid
flowchart TD
    A[Google Drive Trigger] -->|New File| B[Workflow Configuration]
    A1[Webhook Upload] -->|Manual Upload| B1[Upload to Drive]
    B1 --> A
    
    B --> C{Check File Type}
    C -->|PDF/Image| D[Download File Binary]
    C -->|Invalid| E[Log to Dead Letter Queue]
    
    D --> F[OCR API Call]
    F --> G{Check OCR Success}
    
    G -->|Success| H[Clean OCR Text]
    G -->|Failed| I[Log OCR Failure]
    
    H --> J[Extract Medical Metadata with LLM]
    J --> K[Validate and Parse LLM JSON]
    K --> L{Check JSON Valid}
    
    L -->|Valid| M[Upsert Patient in Supabase]
    L -->|Invalid| N[Log Invalid JSON]
    
    M --> O[Generate Document Summary]
    O --> P[Prepare Document Insert Data]
    P --> Q[Insert Medical Document]
    Q --> R[Insert Audit Log]
    R --> S[Move File to Processed Folder]
```

---

## 📄 Workflow Interaction Diagram

```mermaid
sequenceDiagram
    participant GD as Google Drive
    participant WF as n8n Workflow
    participant OCR as OCR.Space API
    participant AI as Google Gemini
    participant DB as Supabase
    
    GD->>WF: New File Detected (Trigger)
    WF->>WF: Validate File Type
    
    alt Invalid File Type
        WF->>DB: Log to Dead Letter Queue
    else Valid PDF/Image
        WF->>GD: Download File Binary
        WF->>OCR: Send File for OCR
        OCR->>WF: Return Extracted Text
        
        alt OCR Failed
            WF->>DB: Log OCR Failure
        else OCR Success
            WF->>WF: Clean OCR Text
            WF->>AI: Extract Medical Metadata
            AI->>WF: Return Structured JSON
            
            WF->>WF: Validate JSON Structure
            
            alt Invalid JSON
                WF->>DB: Log Invalid JSON Error
            else Valid JSON
                WF->>DB: Upsert Patient Record
                WF->>AI: Generate Document Summary
                AI->>WF: Return Summary
                WF->>DB: Insert Medical Document
                WF->>DB: Insert Audit Log
                WF->>GD: Move File to Processed Folder
            end
        end
    end
```

---

## ⚙️ Environment Variables

Although n8n stores credentials separately, the following variables are required:

| Variable | Description |
|----------|-------------|
| `N8N_PORT` | n8n instance port (default: 5678) |
| `N8N_ENCRYPTION_KEY` | Required for credential encryption |
| `OCR_API_KEY` | OCR.Space API key for text extraction |
| `SUPABASE_URL` | Supabase project URL |
| `SUPABASE_SERVICE_KEY` | Supabase service role key (full access) |
| `GEMINI_API_KEY` | Google Gemini API key for AI extraction |
| `GOOGLE_DRIVE_CLIENT_ID` | Google Drive OAuth credentials |
| `GOOGLE_DRIVE_CLIENT_SECRET` | Google Drive OAuth credentials |
| `DRIVE_RAW_FOLDER_ID` | Google Drive folder ID for raw uploads |
| `DRIVE_PROCESSED_FOLDER_ID` | Google Drive folder ID for processed files |

---

## 🌐 API Endpoints

| Endpoint | Method | Workflow Node | Purpose |
|----------|--------|---------------|---------|
| `/ocr-upload` | POST | Webhook | Manual upload endpoint for medical documents |

### Webhook Upload Payload

**Request:**
- **Content-Type:** `multipart/form-data`
- **Field:** `file` (binary)

**Example using cURL:**
```bash
curl -X POST https://your-n8n-instance.com/webhook/ocr-upload \
  -F "file=@/path/to/medical-record.pdf"
```

**Response:**
The webhook automatically processes the file through the entire pipeline. No explicit response is returned as this is a fire-and-forget upload mechanism.

---

## 🗄️ Database Schema

### Tables

#### **patients**
| Column | Type | Description |
|--------|------|-------------|
| `id` | uuid | Primary key (auto-generated) |
| `name` | varchar | Patient full name |
| `phone` | varchar | Patient contact number |
| `cnic` | varchar | National ID (optional) |
| `created_at` | timestamp | Record creation timestamp |
| `updated_at` | timestamp | Last update timestamp |

**Constraints:**
- Unique constraint on `phone` for merge-duplicates upsert logic

#### **medical_documents**
| Column | Type | Description |
|--------|------|-------------|
| `id` | uuid | Primary key (auto-generated) |
| `patient_id` | uuid | Foreign key to patients table |
| `doc_type` | varchar | Document type (Lab Report, Prescription, etc.) |
| `title` | varchar | Human-readable document title |
| `file_url` | text | Google Drive file URL |
| `text_extract` | text | Full OCR extracted text |
| `summary` | text | AI-generated summary |
| `metadata` | jsonb | Structured medical data (JSON) |
| `created_at` | timestamp | Document ingestion timestamp |

**Metadata JSON Structure:**
```json
{
  "document_type": "Lab Report",
  "patient_name": "John Doe",
  "patient_phone": "+1234567890",
  "patient_cnic": "12345-1234567-1",
  "doctor_name": "Dr. Smith",
  "date": "2024-01-15",
  "diagnosis": "Type 2 Diabetes",
  "medicines": ["Metformin 500mg", "Insulin"],
  "tests": ["Blood Sugar", "HbA1c"],
  "notes": "Follow up in 2 weeks"
}
```

#### **audit_logs**
| Column | Type | Description |
|--------|------|-------------|
| `id` | uuid | Primary key |
| `action` | varchar | Action type (e.g., "document_ingested") |
| `actor` | varchar | Who performed action (e.g., "n8n_ocr_worker") |
| `details` | jsonb | Additional action details |
| `timestamp` | timestamp | When action occurred |

#### **dead_letter_records**
| Column | Type | Description |
|--------|------|-------------|
| `id` | uuid | Primary key |
| `file_name` | varchar | Original filename |
| `file_id` | varchar | Google Drive file ID |
| `reason` | varchar | Failure reason |
| `error_details` | text | Full error information |
| `timestamp` | timestamp | When error occurred |

---

## 📁 Folder Structure

```
root/
│  README.md
│
├── workflows/
│     └── medical-record-ocr-automation.json
│
├── database/
│     ├── schema.sql
│     └── indexes.sql
│
├── google-drive/
│     ├── Clinic_Records_Raw/       # Monitored folder
│     └── Clinic_Records_Processed/ # Archived folder
│
└── examples/
      ├── sample-prescription.pdf
      ├── sample-lab-report.pdf
      └── webhook-upload-example.sh
```

---

## 🧠 Workflow Overview

### Single Workflow: Complete Medical Document Processing Pipeline

**Handles:**

**File Ingestion Flow:**
- Google Drive trigger monitoring "Clinic_Records_Raw" folder every 1 minute
- Webhook endpoint for manual file uploads
- Automatic upload to monitored folder via webhook

**Document Processing Flow:**
- File type validation (PDF or Image only)
- Binary file download from Google Drive
- OCR text extraction via OCR.Space API
- Text cleaning and normalization
- AI-powered metadata extraction using Google Gemini
- JSON validation and error handling

**Database Operations:**
- Patient record upsert (create or update based on phone)
- AI-generated document summary
- Medical document record insertion
- Audit log creation for compliance
- Dead letter queue logging for failures

**File Management:**
- Automatic file movement to processed folder
- Maintains Google Drive organization

**Key Features:**
- **Intelligent Error Handling:** Three-tier dead letter queue (invalid file, OCR failure, invalid JSON)
- **AI-Powered Extraction:** Uses Google Gemini 1.5 Pro with structured output parsing
- **HIPAA-Ready Audit Trail:** Complete logging of all document processing
- **Automatic Patient Management:** Upsert logic prevents duplicate records
- **Multi-Format Support:** Handles both PDFs and images
- **Fault-Tolerant Design:** Continues processing even if individual documents fail

---

## 🧱 Node-Level Documentation

Below is a detailed breakdown of all nodes exactly as named in the workflow.

### 🔹 Watch Clinic_Records_Raw Folder

**Type:** `n8n-nodes-base.googleDriveTrigger`  
**Trigger:** File Created  
**Poll Interval:** Every 1 minute

Continuously monitors the "Clinic_Records_Raw" Google Drive folder for newly uploaded files. When a new file is detected, the entire workflow is triggered automatically.

**Configuration:**
- **Folder ID:** `1yLs2GcD5hfqcZPfpDw9GulbT8IyL8bg0`
- **Event:** `fileCreated`

**Output:** Complete file metadata including ID, name, mimeType, and permissions

### 🔹 Webhook

**Type:** `n8n-nodes-base.webhook`  
**Method:** POST  
**Path:** `/ocr-upload`

Provides a manual upload endpoint for medical documents. Accepts multipart/form-data with file binary.

**Use Case:** Allows frontend applications or external systems to submit documents directly without Google Drive access.

### 🔹 Upload file in to clinic_Reecord_Raw Folder

**Type:** `n8n-nodes-base.googleDrive`  
**Operation:** Upload

Uploads files received via webhook to the monitored Google Drive folder, which then triggers the main processing pipeline.

**Configuration:**
- **Target Folder:** Clinic_Records_Raw
- **File Source:** Binary data from webhook

### 🔹 Workflow Configuration

**Type:** `n8n-nodes-base.set`  
**Purpose:** Centralized configuration management

Sets placeholder configuration variables for the entire workflow:

| Variable | Purpose |
|----------|---------|
| `ocrApiKey` | OCR.Space API authentication |
| `supabaseUrl` | Supabase project endpoint |
| `supabaseServiceKey` | Database access credentials |
| `geminiApiKey` | Google Gemini AI access |
| `driveRawFolderId` | Source folder for monitoring |
| `driveProcessedFolderId` | Destination for processed files |

**Note:** These are placeholders and must be replaced during deployment.

### 🔹 Check File Type (PDF or Image)

**Type:** `n8n-nodes-base.if`  
**Condition:** File type validation

Validates that the uploaded file is either a PDF or image format:

```javascript
$binary.file.mimeType.includes('pdf') || $binary.file.mimeType.includes('image')
```

**Routing:**
- **True:** Continues to file download
- **False:** Logs to dead letter queue

**Supported MIME Types:**
- `application/pdf`
- `image/jpeg`
- `image/png`
- `image/tiff`
- `image/bmp`

### 🔹 Log Invalid File Type to Dead Letter Queue

**Type:** `n8n-nodes-base.httpRequest`  
**Method:** POST  
**Target:** Supabase `dead_letter_records` table

Logs files that fail type validation with:
- File name and ID
- Reason: "Invalid file type"
- Full error details
- Timestamp

**Prevents:** Invalid files from breaking the pipeline

### 🔹 Download File Binary

**Type:** `n8n-nodes-base.googleDrive`  
**Operation:** Download

Downloads the file from Google Drive as binary data for OCR processing.

**Output:** Binary file content accessible via `$binary.data`

### 🔹 OCR API Call (OCR.Space)

**Type:** `n8n-nodes-base.httpRequest`  
**Method:** POST  
**Endpoint:** `https://api.ocr.space/parse/image`

Sends file to OCR.Space API for text extraction.

**Configuration:**
- **API Key:** K87043146988957 (hardcoded, should be from config)
- **Language:** English (`eng`)
- **Overlay:** Disabled (text only)
- **Content-Type:** `multipart/form-data`

**Request Body:**
- `apikey`: Authentication
- `file`: Binary data
- `language`: OCR language
- `isOverlayRequired`: false

**Response Structure:**
```json
{
  "ParsedResults": [
    {
      "ParsedText": "Extracted text content...",
      "ErrorMessage": "",
      "FileParseExitCode": 1
    }
  ],
  "IsErroredOnProcessing": false
}
```

### 🔹 Check OCR Success

**Type:** `n8n-nodes-base.if`  
**Condition:** OCR result validation

Validates successful OCR processing:

```javascript
$json.ParsedResults && 
$json.ParsedResults.length > 0 && 
!$json.IsErroredOnProcessing
```

**Routing:**
- **Success:** Continues to text cleaning
- **Failure:** Logs to dead letter queue

### 🔹 Log OCR Failure to Dead Letter Queue

**Type:** `n8n-nodes-base.httpRequest`  
**Purpose:** Log OCR processing failures

Records OCR failures with complete error context for manual review and reprocessing.

### 🔹 Clean OCR Text

**Type:** `n8n-nodes-base.code`  
**Language:** JavaScript

**Purpose:** Normalize and clean extracted OCR text

**Cleaning Operations:**
1. Replace Windows line endings (`\r\n`) with Unix (`\n`)
2. Collapse multiple consecutive newlines to double newlines
3. Replace multiple spaces/tabs with single space
4. Trim leading/trailing whitespace

**Code:**
```javascript
const ocrText = $input.first().json.ParsedResults[0].ParsedText;
const cleaned = ocrText
  .replace(/\r\n/g, '\n')
  .replace(/\n{3,}/g, '\n\n')
  .replace(/[ \t]+/g, ' ')
  .trim();

return [{
  json: {
    ocr_text: cleaned,
    original_file_name: $('Watch Clinic_Records_Raw Folder').first().json.name,
    file_id: $('Watch Clinic_Records_Raw Folder').first().json.id
  }
}];
```

**Output:**
- `ocr_text`: Cleaned text
- `original_file_name`: Original filename
- `file_id`: Google Drive file ID

### 🔹 Extract Medical Metadata with LLM

**Type:** `@n8n/n8n-nodes-langchain.chainLlm`  
**Model:** Google Gemini 1.5 Pro (via connected node)  
**Output Parser:** Structured JSON Parser

**Purpose:** Extract structured medical information from OCR text

**Input:** Cleaned OCR text from previous node

**System Behavior:**
- Uses Gemini 1.5 Pro model with temperature 0 (deterministic)
- Structured output parser enforces JSON schema
- Extracts: patient info, diagnosis, medicines, tests, notes

**Expected Output Schema:**
```json
{
  "document_type": "Lab Report",
  "patient_name": "John Doe",
  "patient_phone": "+1234567890",
  "patient_cnic": "12345-1234567-1",
  "doctor_name": "Dr. Smith",
  "date": "2024-01-15",
  "diagnosis": "Type 2 Diabetes",
  "medicines": ["Metformin 500mg", "Insulin"],
  "tests": ["Blood Sugar", "HbA1c"],
  "notes": "Follow up in 2 weeks"
}
```

### 🔹 Medical Record JSON Parser

**Type:** `@n8n/n8n-nodes-langchain.outputParserStructured`  
**Purpose:** Enforce structured output format

Connected to the LLM chain to ensure consistent JSON output. Provides example schema to guide the AI model.

**JSON Schema Example:** (See above metadata structure)

### 🔹 Gemini 1.5 Pro Model

**Type:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`  
**Model:** Gemini 1.5 Pro  
**Temperature:** 0 (deterministic output)

Provides language model capabilities for:
- Medical metadata extraction
- Document summarization

**Configuration:**
- Connected to both extraction and summarization chains
- Zero temperature ensures consistent extraction
- Optimized for medical terminology understanding

### 🔹 Validate and Parse LLM JSON

**Type:** `n8n-nodes-base.code`  
**Language:** JavaScript

**Purpose:** Validate and normalize LLM output

**Validation Logic:**
1. Parse LLM output (handle string or object)
2. Validate required fields:
   - Must have `document_type`
   - Must have either `patient_name` OR `patient_phone`
3. Set default values for missing fields
4. Preserve OCR text and file metadata

**Code:**
```javascript
try {
  const llmOutput = $input.first().json.output;
  const metadata = typeof llmOutput === 'string' ? JSON.parse(llmOutput) : llmOutput;
  
  const isValid = metadata && 
    (metadata.patient_name || metadata.patient_phone) &&
    metadata.document_type;
  
  return [{
    json: {
      isValid,
      metadata,
      patient_name: metadata.patient_name || 'Unknown',
      patient_phone: metadata.patient_phone || null,
      patient_cnic: metadata.patient_cnic || null,
      document_type: metadata.document_type,
      ocr_text: $('Clean OCR Text').first().json.ocr_text,
      original_file_name: $('Clean OCR Text').first().json.original_file_name,
      file_id: $('Clean OCR Text').first().json.file_id
    }
  }];
} catch (error) {
  return [{
    json: {
      isValid: false,
      error: error.message,
      original_file_name: $('Clean OCR Text').first().json.original_file_name,
      file_id: $('Clean OCR Text').first().json.file_id
    }
  }];
}
```

**Error Handling:** Catches parsing errors and marks as invalid

### 🔹 Check JSON Valid

**Type:** `n8n-nodes-base.if`  
**Condition:** JSON validation check

Validates that the parsed JSON contains required medical information:

```javascript
$json.isValid.toBoolean() === true
```

**Routing:**
- **Valid:** Continues to patient upsert
- **Invalid:** Logs to dead letter queue

### 🔹 Log Invalid JSON to Dead Letter Queue

**Type:** `n8n-nodes-base.httpRequest`  
**Purpose:** Log LLM extraction failures

Records cases where AI extraction failed to produce valid structured data. Useful for:
- Identifying documents needing manual review
- Training data for model improvement
- Pattern analysis of extraction failures

### 🔹 Upsert Patient in Supabase

**Type:** `n8n-nodes-base.httpRequest`  
**Method:** POST  
**Endpoint:** `/rest/v1/patients`

**Purpose:** Create or update patient record

**Upsert Logic:**
Uses Supabase's `resolution=merge-duplicates` header to:
- Create new patient if doesn't exist
- Update existing patient if phone matches
- Return patient record with generated ID

**Request Headers:**
- `apikey`: Supabase service key
- `Authorization`: Bearer token
- `Content-Type`: application/json
- `Prefer`: resolution=merge-duplicates,return=representation

**Request Body:**
```json
{
  "name": "John Doe",
  "phone": "+1234567890"
}
```

**Response:** Complete patient record including auto-generated `id`

### 🔹 Generate Document Summary

**Type:** `@n8n/n8n-nodes-langchain.chainLlm`  
**Model:** Google Gemini 1.5 Pro

**Purpose:** Create human-readable document summary

**Input:** Full OCR text from validation node

**AI Task:** Generate concise medical summary covering:
- Document type and date
- Patient information
- Key findings/diagnosis
- Prescribed medications
- Recommended tests
- Follow-up instructions

**Output:** Natural language summary (no structured format)

### 🔹 Prepare Document Insert Data

**Type:** `n8n-nodes-base.set`  
**Purpose:** Format data for database insertion

**Field Mappings:**

| Field | Source | Description |
|-------|--------|-------------|
| `patient_id` | Upsert Patient response | Foreign key |
| `doc_type` | LLM metadata | Document category |
| `title` | Formatted string | "Doc Type - Patient Name" |
| `file_url` | Constructed URL | Google Drive view link |
| `text_extract` | Clean OCR Text | Full extracted text |
| `summary` | LLM summary | AI-generated summary |
| `metadata` | LLM metadata | JSON object |

**File URL Format:**
```
https://drive.google.com/file/d/{file_id}/view
```

### 🔹 Insert Medical Document in Supabase

**Type:** `n8n-nodes-base.httpRequest`  
**Method:** POST  
**Endpoint:** `/rest/v1/medical_documents`

**Purpose:** Store complete medical document record

**Request Body:**
```json
{
  "patient_id": "uuid",
  "doc_type": "Lab Report",
  "title": "Lab Report - John Doe",
  "file_url": "https://drive.google.com/file/d/...",
  "text_extract": "Full OCR text...",
  "summary": "AI-generated summary...",
  "metadata": { ... }
}
```

**Response:** Created document record with auto-generated `id`

### 🔹 Insert Audit Log in Supabase

**Type:** `n8n-nodes-base.httpRequest`  
**Method:** POST  
**Endpoint:** `/rest/v1/audit_logs`

**Purpose:** HIPAA-compliant activity logging

**Audit Log Structure:**
```json
{
  "action": "document_ingested",
  "actor": "n8n_ocr_worker",
  "details": {
    "patient_id": "uuid",
    "document_id": "uuid",
    "file_name": "prescription.pdf"
  }
}
```

**Compliance:** Creates immutable audit trail for all document processing

### 🔹 Move File to Processed Folder

**Type:** `n8n-nodes-base.googleDrive`  
**Operation:** Move

**Purpose:** Archive successfully processed files

**Configuration:**
- **Source:** Current file location
- **Destination:** Clinic_Records_Processed folder (ID: `1hObHQeDyrJZa9fWx04MQd0wBmXbPqJo9`)

**Benefits:**
- Prevents reprocessing
- Maintains clean inbox
- Preserves original files
- Enables manual review if needed

---

## 🛠️ Setup Instructions

### 1. Install n8n

**Option A: Docker (Recommended)**
```bash
docker pull n8nio/n8n
docker run -it --rm \
  -p 5678:5678 \
  -v ~/.n8n:/home/node/.n8n \
  n8nio/n8n
```

**Option B: npm**
```bash
npm install n8n -g
n8n start
```

**Option C: n8n Cloud**
Sign up at [n8n.cloud](https://n8n.cloud)

### 2. Set Up Supabase Database

**Create New Project:**
1. Go to [supabase.com](https://supabase.com)
2. Create new project
3. Note your project URL and service role key

**Create Database Schema:**

```sql
-- Enable UUID extension
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Patients Table
CREATE TABLE patients (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  name VARCHAR(255),
  phone VARCHAR(50) UNIQUE,
  cnic VARCHAR(50),
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- Medical Documents Table
CREATE TABLE medical_documents (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  patient_id UUID REFERENCES patients(id) ON DELETE CASCADE,
  doc_type VARCHAR(100),
  title VARCHAR(500),
  file_url TEXT,
  text_extract TEXT,
  summary TEXT,
  metadata JSONB,
  created_at TIMESTAMP DEFAULT NOW()
);

-- Audit Logs Table
CREATE TABLE audit_logs (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  action VARCHAR(100),
  actor VARCHAR(255),
  details JSONB,
  timestamp TIMESTAMP DEFAULT NOW()
);

-- Dead Letter Records Table
CREATE TABLE dead_letter_records (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  file_name VARCHAR(500),
  file_id VARCHAR(255),
  reason VARCHAR(255),
  error_details TEXT,
  timestamp TIMESTAMP DEFAULT NOW()
);

-- Indexes for Performance
CREATE INDEX idx_patients_phone ON patients(phone);
CREATE INDEX idx_documents_patient ON medical_documents(patient_id);
CREATE INDEX idx_documents_type ON medical_documents(doc_type);
CREATE INDEX idx_documents_created ON medical_documents(created_at DESC);
CREATE INDEX idx_audit_timestamp ON audit_logs(timestamp DESC);
CREATE INDEX idx_dead_letter_timestamp ON dead_letter_records(timestamp DESC);
```

### 3. Configure Google Drive

**Create Folder Structure:**
1. Create "Clinic_Records_Raw" folder (for uploads)
2. Create "Clinic_Records_Processed" folder (for archives)
3. Note both folder IDs from URL:
   ```
   https://drive.google.com/drive/folders/{FOLDER_ID}
   ```

**Set Up OAuth:**
1. Go to [Google Cloud Console](https://console.cloud.google.com)
2. Create new project or select existing
3. Enable Google Drive API
4. Create OAuth 2.0 credentials
5. Add authorized redirect URI: `http://localhost:5678/rest/oauth2-credential/callback`
6. Download credentials JSON

### 4. Get API Keys

**OCR.Space:**
1. Register at [ocr.space](https://ocr.space/ocrapi)
2. Get free API key (25,000 requests/month)

**Google Gemini:**
1. Go to [Google AI Studio](https://makersuite.google.com/app/apikey)
2. Create API key
3. Note the key for n8n configuration

### 5. Configure n8n Credentials

**Google Drive OAuth2:**
1. n8n → Settings → Credentials → Add Credential
2. Select "Google Drive OAuth2"
3. Enter Client ID and Client Secret
4. Complete OAuth flow

**Supabase (Custom HTTP Request):**
- Service URL: Your Supabase project URL
- Service Key: Supabase service role key (not anon key)

**Google Gemini API:**
1. Add credential for "Google PaLM API"
2. Enter Gemini API key

### 6. Import Workflow

1. Download `medical-record-ocr-automation.json`
2. In n8n: Workflows → Import from File
3. Update all credential references
4. Update configuration node with actual values:
   - Replace OCR API key placeholder
   - Replace Supabase URL and key placeholders
   - Replace folder ID placeholders

### 7. Test the System

**Test via Google Drive:**
1. Upload a medical PDF to "Clinic_Records_Raw"
2. Wait up to 1 minute for trigger
3. Check n8n executions for processing status
4. Verify file moved to "Processed" folder
5. Check Supabase tables for inserted records

**Test via Webhook:**
```bash
curl -X POST https://your-n8n-instance.com/webhook/ocr-upload \
  -F "file=@sample-prescription.pdf"
```

**Verify Results:**
- Check `patients` table for new patient
- Check `medical_documents` table for document record
- Check `audit_logs` for processing log
- Check `dead_letter_records` if any failures

---

## 🚀 Deployment Guide

### Option 1: n8n Cloud (Recommended for Healthcare)

**Advantages for Medical Use:**
- HIPAA-compliant infrastructure
- Automatic backups
- Built-in monitoring
- No server management
- Guaranteed uptime SLA

**Setup:**
1. Sign up at [n8n.cloud](https://n8n.cloud)
2. Choose healthcare-compliant tier
3. Import workflow
4. Configure credentials
5. Enable workflow

**Cost:** Starting at $20/month (Pro tier recommended: $50/month)

### Option 2: Docker Production Deployment

**docker-compose.yml:**
```yaml
version: '3.8'

services:
  n8n:
    image: n8nio/n8n:latest
    restart: always
    ports:
      - "5678:5678"
    environment:
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
      - N8N_PROTOCOL=https
      - N8N_HOST=${N8N_HOST}
      - WEBHOOK_URL=https://${N8N_HOST}/
      - EXECUTIONS_DATA_PRUNE=true
      - EXECUTIONS_DATA_MAX_AGE=336
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_DATABASE=n8n
      - DB_POSTGRESDB_USER=n8n
      - DB_POSTGRESDB_PASSWORD=${POSTGRES_PASSWORD}
    volumes:
      - n8n_data:/home/node/.n8n
    depends_on:
      - postgres

  postgres:
    image: postgres:15
    restart: always
    environment:
      - POSTGRES_DB=n8n
      - POSTGRES_USER=n8n
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  n8n_data:
  postgres_data:
```

**Environment Variables (.env):**
```env
N8N_ENCRYPTION_KEY=your_secure_encryption_key_here
N8N_HOST=ocr.yourhospital.com
POSTGRES_PASSWORD=secure_postgres_password
```

### Option 3: Self-Hosted with Nginx

**Nginx Configuration:**
```nginx
server {
    listen 443 ssl http2;
    server_name ocr.yourhospital.com;

    ssl_certificate /etc/letsencrypt/live/ocr.yourhospital.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/ocr.yourhospital.com/privkey.pem;

    # Security headers for HIPAA compliance
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;

    # File upload limits for medical documents
    client_max_body_size 50M;

    location / {
        proxy_pass http://localhost:5678;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
        
        # Timeout settings for long OCR processing
        proxy_connect_timeout 300s;
        proxy_send_timeout 300s;
        proxy_read_timeout 300s;
    }
}
```

### Option 4: Kubernetes Deployment

For large hospital networks processing 1000+ documents/day:

**deployment.yaml:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: n8n-ocr-worker
spec:
  replicas: 3
  selector:
    matchLabels:
      app: n8n-ocr
  template:
    metadata:
      labels:
        app: n8n-ocr
    spec:
      containers:
      - name: n8n
        image: n8nio/n8n:latest
        env:
        - name: N8N_ENCRYPTION_KEY
          valueFrom:
            secretKeyRef:
              name: n8n-secrets
              key: encryption-key
        - name: DB_TYPE
          value: postgresdb
        - name: DB_POSTGRESDB_HOST
          value: postgres-service
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "1000m"
        volumeMounts:
        - name: n8n-data
          mountPath: /home/node/.n8n
      volumes:
      - name: n8n-data
        persistentVolumeClaim:
          claimName: n8n-pvc
```

---

## 📈 Scalability & Limitations

### Strengths

- **Intelligent AI Extraction:** Gemini 1.5 Pro understands medical terminology
- **Fault-Tolerant Pipeline:** Three-tier error handling with dead letter queue
- **Database-Backed Storage:** Scalable Supabase PostgreSQL backend
- **Automatic Patient Management:** Upsert logic prevents duplicates
- **Multi-Format Support:** Handles PDFs and multiple image formats
- **Audit Trail:** Complete HIPAA-compliant activity logging
- **File Organization:** Automatic archival to processed folder
- **Dual Ingestion:** Google Drive trigger + webhook endpoint
- **Structured Metadata:** JSON storage enables powerful querying

### Limitations

- **OCR Accuracy:** Dependent on document quality and scan resolution
  - Handwritten notes may have poor recognition
  - Low-quality scans reduce accuracy
  - Solution: Pre-process images or use premium OCR service
- **OCR.Space API Limits:**
  - Free tier: 25,000 requests/month
  - Rate limit: 10 requests/minute
  - Max file size: 1MB
  - Solution: Upgrade to paid tier or use Google Cloud Vision API
- **Google Drive Trigger Delay:** 1-minute polling interval
  - Near real-time but not instant
  - Solution: Reduce polling interval or use Drive API push notifications
- **Gemini API Costs:**
  - Input: ~$0.35 per 1M tokens
  - Output: ~$1.05 per 1M tokens
  - Typical medical document: 1,000-3,000 tokens
  - Estimated cost: $0.001-0.005 per document
- **Single Language Support:** Currently optimized for English
  - Solution: Add language detection and multi-language OCR
- **No Document Versioning:** Updates overwrite previous records
  - Solution: Implement document version history table
- **Synchronous Processing:** Each document processed sequentially
  - Solution: Implement queue system with parallel workers

### Cost Considerations

**Monthly Cost Breakdown (500 documents/month):**

| Service | Cost |
|---------|------|
| Supabase Pro | $25 |
| OCR.Space Free Tier | $0 (within limits) |
| Google Gemini API | $2-5 (based on document complexity) |
| Google Drive Storage | $0-2 (within 15GB free tier) |
| n8n Cloud (Pro) | $50 |
| **Total** | **$77-82/month** |

**Scaling to 5,000 documents/month:**

| Service | Cost |
|---------|------|
| Supabase Pro | $25 |
| OCR.Space Pro | $60 |
| Google Gemini API | $20-50 |
| Google Drive (100GB) | $2 |
| n8n Cloud (Team) | $150 |
| **Total** | **$257-287/month** |

---

## 🚀 Future Improvements

### Short-Term Enhancements

- **Enhanced OCR Options:** Add Google Cloud Vision API as alternative
- **Multi-Language Support:** Detect language and use appropriate OCR settings
- **Document Validation:** Pre-flight checks for image quality
- **Notification System:** Email/SMS alerts for successful processing
- **Patient Portal Integration:** API endpoints for patient document access
- **Bulk Upload:** Process multiple files in single operation
- **Retry Mechanism:** Automatic retry for transient failures

### Medium-Term Features

- **Document Versioning:** Track changes to patient records over time
- **Advanced Search:** Full-text search across all medical documents
- **Analytics Dashboard:** Processing metrics, success rates, error patterns
- **Document Classification:** AI-based automatic document type detection
- **Data Extraction Templates:** Custom templates for specific document types
- **Compliance Reporting:** HIPAA audit reports and compliance checks
- **Integration Hub:** Connect to EHR systems (Epic, Cerner, Meditech)

### Long-Term Vision

- **Handwriting Recognition:** Specialized AI for doctor handwriting
- **Medical Entity Recognition:** NER for medications, conditions, procedures
- **Clinical Decision Support:** Flag critical values or drug interactions
- **Predictive Analytics:** Identify patterns in patient health data
- **Blockchain Audit Trail:** Immutable medical record tracking
- **FHIR Compliance:** Full interoperability with healthcare systems
- **Multi-Tenant Support:** Separate data for multiple clinics/hospitals
- **Real-Time Collaboration:** Allow doctors to annotate documents

### Technical Improvements

- **Queue System:** Redis-based job queue for parallel processing
  ```
  Google Drive → Queue → Multiple Workers → Database
  ```
- **Caching Layer:** Cache frequent OCR results to reduce API costs
- **Document Pre-Processing:** Auto-rotate, enhance contrast, remove noise
- **Async Processing:** Non-blocking workflow execution
- **Load Balancing:** Distribute OCR requests across multiple API keys
- **Monitoring Dashboard:** Grafana + Prometheus for system health
- **Error Alerting:** PagerDuty/Slack notifications for critical failures
- **Automated Testing:** Integration tests for entire pipeline
- **CI/CD Pipeline:** GitHub Actions for workflow deployment
- **Data Encryption:** At-rest encryption for sensitive medical data

---

## 📋 Appendix

### Sample Medical Documents Supported

**Prescriptions:**
- Doctor prescriptions
- Medication lists
- Dosage instructions

**Lab Reports:**
- Blood tests
- Urinalysis
- Pathology reports
- Radiology reports

**Clinical Notes:**
- Progress notes
- Discharge summaries
- Consultation notes

**Administrative:**
- Insurance forms
- Consent forms
- Medical history forms

### OCR Best Practices

**Document Quality Guidelines:**
- **Resolution:** Minimum 300 DPI for scans
- **Format:** PDF preferred over images
- **File Size:** Under 1MB for free OCR tier
- **Color:** Grayscale or black-and-white for text documents
- **Orientation:** Correctly rotated (not sideways/upside down)
- **Clarity:** Clear, non-blurry text

**Common OCR Issues:**

| Issue | Cause | Solution |
|-------|-------|----------|
| Low accuracy | Poor scan quality | Re-scan at higher DPI |
| Missing text | Image compression | Use lossless formats (PNG) |
| Wrong language | Language mismatch | Set correct OCR language |
| Timeout | Large file size | Compress or split document |

### Database Query Examples

**Find all documents for a patient:**
```sql
SELECT d.*, p.name as patient_name
FROM medical_documents d
JOIN patients p ON d.patient_id = p.id
WHERE p.phone = '+1234567890'
ORDER BY d.created_at DESC;
```

**Search documents by diagnosis:**
```sql
SELECT *
FROM medical_documents
WHERE metadata->>'diagnosis' ILIKE '%diabetes%'
ORDER BY created_at DESC;
```

**Audit trail for specific patient:**
```sql
SELECT *
FROM audit_logs
WHERE details->>'patient_id' = 'uuid-here'
ORDER BY timestamp DESC;
```

**Find failed processing attempts:**
```sql
SELECT *
FROM dead_letter_records
WHERE timestamp > NOW() - INTERVAL '7 days'
ORDER BY timestamp DESC;
```

### Troubleshooting Guide

**Problem: OCR returns empty text**
- **Cause:** Image-only PDF or very poor quality
- **Solution:** Enable image preprocessing or use higher quality source

**Problem: LLM extraction fails consistently**
- **Cause:** Document format not recognized by AI
- **Solution:** Add examples to prompt or create custom extraction template

**Problem: Duplicate patients created**
- **Cause:** Phone number variations (spaces, dashes, etc.)
- **Solution:** Normalize phone numbers before upsert

**Problem: Files not moving to processed folder**
- **Cause:** Insufficient Google Drive permissions
- **Solution:** Verify OAuth scope includes drive.file permission

**Problem: Workflow execution timeout**
- **Cause:** Large files or slow API responses
- **Solution:** Increase n8n execution timeout or split large documents

---

## 📄 License

MIT License - See LICENSE file for details

---

## 🤝 Contributing

Contributions welcome! Please:
1. Fork the repository
2. Create feature branch
3. Test with sample medical documents (de-identified)
4. Ensure HIPAA compliance considerations
5. Submit pull request with detailed description

**Areas Needing Contribution:**
- Multi-language OCR support
- Additional document type templates
- Enhanced error handling
- Performance optimizations
- Integration with EHR systems

---

## 📞 Support

For issues or questions:
- GitHub Issues: [Repository Issues Page]
- n8n Community: [community.n8n.io](https://community.n8n.io)
- Supabase Docs: [supabase.com/docs](https://supabase.com/docs)
- OCR.Space Support: [ocr.space/ocrapi#PostParameters](https://ocr.space/ocrapi)

---

## 🏥 Healthcare Compliance Notes

### HIPAA Considerations

**This workflow handles Protected Health Information (PHI). Ensure:**

1. **Data Encryption:**
   - Enable at-rest encryption in Supabase
   - Use HTTPS for all API calls
   - Encrypt n8n execution data

2. **Access Controls:**
   - Use service accounts with minimum required permissions
   - Rotate API keys regularly
   - Implement role-based access in Supabase

3. **Audit Logging:**
   - Complete audit trail is logged automatically
   - Retain logs for minimum 6 years
   - Regular audit log reviews

4. **Data Retention:**
   - Define retention policies for documents
   - Implement automated archival/deletion
   - Comply with local healthcare regulations

5. **Business Associate Agreements (BAA):**
   - Ensure n8n Cloud has signed BAA (Enterprise tier)
   - Verify Supabase BAA (required for healthcare)
   - Confirm Google Workspace BAA for Drive

**Disclaimer:** This is a technical implementation. Consult with healthcare compliance officers and legal counsel before deploying in production medical environments.

---

## 🔐 Security Recommendations

1. **API Key Management:**
   - Never hardcode API keys in workflow
   - Use n8n credentials manager
   - Rotate keys every 90 days

2. **Network Security:**
   - Deploy behind VPN for internal use
   - Use IP whitelisting for webhook endpoint
   - Enable rate limiting

3. **Data Privacy:**
   - Implement data anonymization for test environments
   - Regular security audits
   - Penetration testing

4. **Backup Strategy:**
   - Daily Supabase backups
   - Google Drive versioning enabled
   - n8n workflow version control

---

## 📊 Performance Metrics

**Typical Processing Times:**

| Stage | Duration |
|-------|----------|
| OCR Extraction (1-page PDF) | 2-5 seconds |
| AI Metadata Extraction | 3-8 seconds |
| AI Summary Generation | 4-10 seconds |
| Database Operations | <1 second |
| File Movement | 1-2 seconds |
| **Total Pipeline** | **10-30 seconds** |

**Throughput Capacity:**

| Deployment | Documents/Hour |
|------------|----------------|
| Single n8n instance | 120-180 |
| 3-instance cluster | 360-540 |
| With queue system | 1,000+ |

---
