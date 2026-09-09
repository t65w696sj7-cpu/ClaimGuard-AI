# ClaimGuard AI — AWS Architecture

## Purpose

This document defines the initial AWS architecture for ClaimGuard AI based on the business requirements and architecture decisions established for NorthStar Mutual Insurance.

The architecture is designed to support secure claim intake, durable storage, asynchronous processing, controlled document access, human review, and responsible AI integration.

---

## 1. Customer Authentication

Customers authenticate before accessing ClaimGuard.

### Proposed AWS Service

- Amazon Cognito

### Responsibilities

- Customer sign-in
- Multi-factor authentication
- Session management
- Controlled access to claim submission functions

### Architecture Principle

Users should only receive access necessary to interact with their own claims.

---

## 2. Claim Intake API

ClaimGuard uses a secure API layer to receive structured claim information and coordinate document uploads.

### Proposed AWS Services

- Amazon API Gateway
- AWS Lambda

### Responsibilities

The intake workflow can:

- Receive structured claim information
- Validate required fields
- Create the initial claim record
- Generate temporary authorization for document uploads
- Associate uploaded documents with the correct claim
- Confirm successful claim acceptance only after required information is safely stored

---

## 3. Structured Claim Storage

Structured claim information is stored separately from large claim documents.

### Proposed AWS Service

- Amazon Aurora PostgreSQL or Amazon RDS for PostgreSQL

### Example Structured Data

- Claim number
- Customer identity reference
- Policy number
- Accident date
- Claim status
- Priority
- Assigned adjuster
- Processing state
- Document references
- Timestamps

### Architecture Principle

The relational database should store structured claim records and references to documents rather than storing large document files directly.

---

## 4. Original Evidence Storage

Customer-submitted files are stored in Amazon S3.

### Proposed AWS Service

- Amazon S3

### Example Documents

- Police reports
- Repair estimates
- Photographs
- Scanned documents
- Supporting PDFs

### Architecture Principle

Original customer evidence should remain preserved and should not be overwritten by AI or document-processing workflows.

Original files and AI-generated or processed files should remain separated.

---

## 5. Direct Document Upload

Large documents should not be unnecessarily routed through the application processing layer.

### Proposed Design

ClaimGuard issues temporary authorization that allows the customer to upload approved documents directly to Amazon S3.

### Proposed AWS Capability

- Amazon S3 pre-signed URLs

### Benefits

- Reduces application processing load
- Avoids sending large files through Lambda
- Supports large document uploads efficiently
- Maintains controlled access to S3
- Allows upload authorization to expire automatically

---

## 6. Document Validation

A successful upload does not automatically mean a document is trusted.

After upload, ClaimGuard should validate the document before downstream processing.

### Validation Checks

- File exists
- Supported file type
- File size
- Claim association
- Duplicate detection
- Security inspection
- Required document metadata

---

## 7. Duplicate Document Detection

ClaimGuard should prevent unnecessary processing and storage of exact duplicate documents.

### Design

The system can create a cryptographic fingerprint for each uploaded file and compare it with files already associated with the claim.

If an exact duplicate exists:

- Do not process the document again
- Reuse the existing document reference
- Record the additional upload attempt for audit purposes

If documents appear similar but are not exact duplicates:

- Flag them for adjuster review
- Allow the adjuster to determine the active version
- Preserve prior document history

---

## 8. Separate Processed Document Storage

AI-generated and transformed artifacts should remain separate from original customer evidence.

### Processed Artifacts May Include

- OCR output
- Extracted text
- AI summaries
- Resized images
- Enhanced images
- Document embeddings
- Derived metadata

### Architecture Principle

The system must always be able to distinguish:

1. What the customer originally submitted
2. What ClaimGuard or the AI generated from that submission

---

## 9. Asynchronous Claims Processing

Claims should not be sent directly from the customer intake process into AI processing.

After a claim is safely accepted, a processing message is placed into a queue.

### Proposed AWS Service

- Amazon SQS

### Benefits

- Protects the system during catastrophe-level claim spikes
- Decouples claim intake from AI processing
- Allows claims to wait safely
- Allows downstream systems to process claims at a controlled rate

### Priority Design

Because Amazon SQS does not provide native message priority, ClaimGuard may use multiple queues such as:

- Critical
- High
- Standard
- Low

Routing logic can place claims into the appropriate queue based on NorthStar business rules.

Authorized management may override claim priority when necessary.

---

## 10. Missing Information Workflow

Missing information should be detected before expensive AI processing occurs.

### Workflow

1. Claim enters processing
2. Required fields and documents are checked
3. Missing information is detected
4. Claim is flagged
5. Assigned adjuster is notified
6. Adjuster reviews the issue
7. Adjuster decides whether customer contact is required

### Architecture Principle

AI may identify missing information, but the adjuster remains responsible for determining the appropriate action.

---

## 11. Incremental Reprocessing

New documents should not cause the entire claim to restart processing.

When new information is submitted:

1. Identify what has already been processed
2. Process only the new or changed document
3. Compare new extracted information with the existing claim state
4. Merge compatible information
5. Flag conflicts for adjuster review

### Business Benefit

This reduces:

- Duplicate AI processing
- Processing time
- Cloud cost
- Unnecessary queue traffic

---

## 12. Conflict Detection

ClaimGuard may detect significant differences between new and existing information.

Example:

An original repair estimate is $7,200 and a new estimate is $18,500.

ClaimGuard should:

- Preserve both documents
- Detect the significant difference
- Notify the assigned adjuster
- Request verification before further escalation

### Architecture Principle

AI identifies anomalies.

Authorized humans determine what those anomalies mean.

ClaimGuard should not automatically label a customer or claim as fraudulent.

---

## 13. Least-Privilege Document Access

AI and processing components should not receive unrestricted access to all documents.

### Design Principle

When a claim is being processed:

1. Identify the documents required for that claim
2. Authorize access only to those documents
3. Process the claim
4. Log the access
5. Avoid broader access than necessary

### Proposed AWS Capabilities

- IAM roles
- IAM policies
- Temporary credentials
- S3 object-level permissions

---

## Initial AWS Intake Flow

Customer  
↓  
Amazon Cognito  
↓  
Amazon API Gateway  
↓  
AWS Lambda  
↓  
Structured claim data → Amazon Aurora/RDS PostgreSQL  
↓  
Temporary upload authorization  
↓  
Customer uploads documents directly to Amazon S3  
↓  
Document validation and duplicate detection  
↓  
Claim safely accepted  
↓  
Amazon SQS  
↓  
Downstream processing

---

## Current Architecture Status

Completed:

- Customer authentication approach
- Structured and unstructured data separation
- Direct S3 document upload design
- Original and processed document separation
- Duplicate handling
- Queue-based processing
- Missing-information workflow
- Incremental reprocessing
- Conflict review
- Least-privilege document access

Next:

- Design document extraction and OCR layer
- Design AI and RAG layer
- Design adjuster workflow
- Design security monitoring
- Design notification services
- Design backup and disaster recovery
- Design observability and cost controls
