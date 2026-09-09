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


---

## 14. Document Intelligence and Extraction

ClaimGuard must be able to extract useful information from scanned police reports, repair estimates, claim forms, photographs, and other supporting documents without modifying the original evidence.

### Proposed AWS Service

- Amazon Textract

### Responsibilities

Amazon Textract can be used to extract:

- Names
- Dates
- Claim-related identifiers
- Police report numbers
- Vehicle information
- Accident locations
- Form fields
- Tables
- Narrative text
- Repair estimate information

### Architecture Principle

Original evidence remains unchanged.

Extracted information and processing artifacts are stored separately from the original customer-submitted documents.

---

## 15. Extraction Confidence and Human Verification

ClaimGuard should not automatically trust low-confidence extracted information.

### Workflow

1. Document is processed
2. Information is extracted
3. Extraction confidence is evaluated
4. High-confidence information may continue through processing
5. Low-confidence or ambiguous information is flagged
6. Assigned adjuster reviews the original document
7. Adjuster corrects or verifies the extracted value

### Example

If ClaimGuard reads a handwritten accident location as:

Harford Road — 65% confidence

the field should be marked:

Adjuster Verification Required

### Architecture Principle

When confidence is insufficient, human verification takes priority over automated interpretation.

---

## 16. AI-Assisted Claim Pre-Review

ClaimGuard is designed to reduce the amount of time adjusters spend manually organizing and reading claim files.

Claims may be pre-reviewed before an adjuster opens them.

### Pre-Review May Include

- Organizing claim documents
- Identifying missing information
- Extracting important facts
- Summarizing long documents
- Comparing new and previous documents
- Identifying discrepancies
- Determining claim priority based on approved business rules
- Preparing an adjuster claim brief

### Example Claim Brief

The adjuster may see:

- Claim number
- Customer or claimant name
- Date of birth
- Date of loss
- Reason for claim
- Priority
- Identification status
- Police report status
- Missing information
- Conflicting information
- Documents available
- Items requiring human review

Sensitive information such as Social Security numbers should be masked unless full access is specifically required and authorized.

### Architecture Principle

ClaimGuard prepares the claim.

The adjuster remains responsible for consequential claim decisions.

---

## 17. Retrieval-Augmented Generation

ClaimGuard should not repeatedly send an entire claim package through an AI model every time an adjuster asks a question.

Previously processed information should be indexed and retrieved when needed.

### Proposed AWS Services

- Amazon Bedrock
- Amazon Bedrock Knowledge Bases
- Vector search capability such as Amazon OpenSearch Serverless

### Processing Flow

1. Documents are extracted and processed
2. Processed text is divided into searchable chunks
3. Metadata is attached to every chunk
4. Chunks are indexed for semantic retrieval
5. Adjuster asks a question
6. ClaimGuard filters retrieval to the authorized claim
7. Relevant chunks are retrieved
8. Amazon Bedrock generates an answer grounded in those sources

### Example Question

Why was this repair estimate flagged?

Instead of processing the entire claim again, ClaimGuard retrieves the relevant current estimate, previous estimate, extracted dollar amounts, and discrepancy information.

### Business Benefits

- Faster AI responses
- Lower AI processing cost
- Reduced duplicate processing
- More focused answers
- Better support for large adjuster workloads

---

## 18. RAG Metadata and Claim Isolation

Every indexed document chunk should contain metadata that identifies where the information belongs.

### Example Metadata

- Claim ID
- Document ID
- Document type
- Source document
- Page number
- Section
- Upload date
- Processing version
- Assigned adjuster
- Claim priority

### Architecture Principle

Text describes what the information means.

Metadata identifies where the information belongs.

Retrieval must be restricted to the appropriate claim and authorized personnel.

ClaimGuard should never retrieve information from an unrelated claim simply because the text is semantically similar.

---

## 19. Grounded AI Responses

ClaimGuard should provide evidence alongside AI-generated answers.

When an adjuster asks a question, the response should identify the documents used to produce the answer.

### Example

ClaimGuard may report:

The latest repair estimate is significantly higher than the previous estimate.

Sources:

- Current Repair Estimate — Page 2 — Repair Items
- Previous Repair Estimate — Page 1 — Estimate Total

### Source Metadata

AI responses should preserve references to:

- Source document
- Page number
- Section
- Document version
- Claim ID

### Architecture Principle

AI responses should be verifiable.

The adjuster should be able to review the original evidence supporting an AI-generated answer instead of being expected to trust the AI response alone.

---

## Updated Processing Flow

Customer  
↓  
Amazon Cognito  
↓  
Amazon API Gateway / AWS Lambda  
↓  
Amazon Aurora PostgreSQL + Amazon S3 Original Evidence  
↓  
Amazon SQS  
↓  
Amazon Textract  
↓  
Extraction Confidence Check  
↓  
Human Verification When Required  
↓  
Processed / Derived Storage  
↓  
Chunking + Metadata  
↓  
Amazon Bedrock Knowledge Base / Vector Index  
↓  
Relevant Claim Information Retrieved  
↓  
Amazon Bedrock  
↓  
Grounded Claim Brief or Adjuster Answer  
↓  
Source Document + Page / Section  
↓  
Human Adjuster

---

## 20. Adjuster Dashboard and Workload Prioritization

ClaimGuard should organize the adjuster's workload so the most urgent claims and verification issues are visible immediately.

### Dashboard Sections

- Critical
- High
- Needs Verification
- Standard
- Low

### Architecture Principle

Claim urgency should be the primary ordering factor.

Waiting time may be used as a secondary factor so older claims are not forgotten.

Authorized personnel may override system-assigned priority when necessary.

---

## 21. Verification Reasons on Claim Cards

Claims requiring human verification should display the reason for the flag before the adjuster opens the claim.

### Example Verification Reasons

- Low-confidence document extraction
- Missing document
- Repair estimate conflict
- Possible duplicate
- Conflicting customer information
- New information requiring verification

### Architecture Principle

The adjuster should understand why a claim requires attention without opening the entire file first.

---

## 22. Source-Directed Review

When ClaimGuard identifies a problem in a document, the adjuster should be taken directly to the evidence that caused the flag.

### Example Workflow

1. Adjuster selects a verification flag
2. ClaimGuard opens the original source document
3. The relevant page or section is displayed
4. The information that triggered the flag is highlighted
5. The adjuster may review the entire original document if needed

### Example

Verification Required — Accident Location

Extracted Value:

Harford Road

Confidence:

65%

Source:

Police Report — Page 7

### Adjuster Actions

- Verify
- Correct
- Escalate

---

## 23. Correction Audit History

When an adjuster corrects extracted information, ClaimGuard should preserve both the original extracted value and the verified correction.

### Example

Original Extraction:

Harford Road — 65% confidence

Verified Correction:

Hartford Road

Source:

Police Report — Page 7

### Architecture Principle

Corrections should not erase processing history.

ClaimGuard should maintain an auditable record of what the system originally extracted and what the authorized adjuster changed.

---

## 24. Dependency-Aware Reprocessing

An adjuster correction should not require the entire claim to be processed again.

ClaimGuard should determine which downstream components depended on the corrected information and refresh only those components.

### Example

If an accident location changes from:

Harford Road

to:

Hartford Road

ClaimGuard may need to refresh:

- Claim brief
- Extracted claim data
- RAG index
- Conflict checks
- AI-generated summaries that referenced the location

Unrelated documents and completed processing should not be repeated unnecessarily.

### Architecture Principle

Reprocess affected dependencies, not the entire claim.

### Business Benefits

- Lower cloud cost
- Faster corrections
- Reduced duplicate AI processing
- More efficient claims operations

---

## 25. Conflicting Evidence Handling

ClaimGuard should not automatically overwrite one source of evidence when another source contains different information.

### Example

Police Report — Page 7:

Hartford Road

Customer Claim Form:

Harford Road

### Workflow

1. Preserve both values
2. Preserve both source documents
3. Flag the discrepancy
4. Show the adjuster where each value originated
5. Require human review
6. Allow the adjuster to determine whether further verification is necessary

### Architecture Principle

Conflicting evidence should be preserved and traced to its source.

ClaimGuard identifies the conflict.

Authorized personnel determine what the conflict means and what action should be taken.

---

## Adjuster Workflow

Adjuster signs in  
↓  
Prioritized workload displayed  
↓  
Critical / High / Needs Verification / Standard / Low  
↓  
Adjuster selects claim  
↓  
AI Claim Brief + Original Evidence displayed  
↓  
Verification flag selected  
↓  
Relevant source document opens to exact page / section  
↓  
Adjuster verifies, corrects, or escalates  
↓  
Correction stored in audit history  
↓  
Affected dependencies identified  
↓  
Only affected components reprocessed  
↓  
New conflicts checked  
↓  
Claim brief and retrieval index updated


---

## 26. Security Monitoring and Automatic Containment

ClaimGuard should detect abnormal access behavior and automatically contain potentially compromised accounts.

### Example Security Event

An adjuster who normally accesses a limited number of assigned claims suddenly attempts to access hundreds of unrelated claims or download an abnormal volume of customer documents.

### Response Workflow

1. Detect abnormal access behavior
2. Suspend the affected account or session
3. Revoke active access
4. Stop additional document access
5. Notify authorized security and management personnel
6. Preserve security evidence
7. Begin investigation

### Architecture Principle

Contain suspicious activity immediately without automatically assuming that the employee is responsible for the activity.

---

## 27. Successful and Denied Access Logging

ClaimGuard should preserve both successful and unsuccessful access attempts.

### Security Records May Include

- User or service identity
- Claim ID
- Document ID
- Successful access
- Denied access attempt
- Download activity
- Timestamp
- Session information
- Available device or network context
- Action attempted

### Architecture Principle

Successful access identifies information that may have been exposed.

Denied attempts help investigators understand the intended scope and pattern of an attack.

---

## 28. Protected Audit Records

Critical security records should be stored separately from normal application activity and protected from unauthorized modification or deletion.

### Proposed AWS Services

- AWS CloudTrail
- Amazon S3
- Amazon S3 Object Lock where appropriate

### Architecture Principle

A compromised account should not be capable of erasing the evidence of its own activity.

Security records should have defined retention requirements and tightly controlled deletion permissions.

---

## 29. Security Suspension and Account Restoration

A security-suspended account should remain locked even if valid credentials are subsequently provided.

### Recovery Workflow

1. Account or session is suspended
2. Active sessions are revoked
3. Security incident is opened
4. Audit evidence is reviewed
5. User identity is securely re-verified
6. Credentials or MFA are reset when necessary
7. Authorized security personnel approve restoration
8. New authenticated access is established
9. Account may receive heightened monitoring following restoration

### Architecture Principle

Successful authentication does not override an active security suspension.

---

## 30. Fine-Grained Authorization

Being authenticated as an adjuster should not automatically provide access to every claim or every action.

Authorization should evaluate:

- User identity
- User role
- Assigned claim
- Requested resource
- Requested action
- Applicable permissions

### Example

An adjuster may be authorized to:

- View an assigned claim
- Review supporting evidence
- Verify extracted information
- Update permitted claim information

The same adjuster may not be authorized to:

- Access unrelated claims
- Delete original evidence
- Modify audit records
- Change security policies
- Perform unauthorized bulk exports

### Proposed AWS Capabilities

- AWS IAM
- Role-based access control
- Attribute-based access control
- Application-level claim authorization

### Architecture Principle

Authentication establishes identity.

Authorization determines what that identity may do to a specific resource.

---

## 31. Sensitive Data Masking

Access to a claim should not automatically expose every sensitive field contained within that claim.

### Sensitive Information May Include

- Social Security numbers
- Driver's license numbers
- Dates of birth
- Financial information
- Other personally identifiable information

### Example

Normal View:

SSN: ***-**-1234

Full sensitive values should only be revealed when specifically authorized and required for a legitimate workflow.

Sensitive-data access should be logged.

### Architecture Principle

Provide users with the minimum sensitive information necessary to perform their authorized responsibilities.

---

## 32. Encryption

ClaimGuard should protect sensitive information both while it is stored and while it is moving through the system.

### Encryption at Rest

Encryption should be applied where appropriate to:

- Amazon S3 evidence
- Processed document storage
- Amazon Aurora / RDS
- Amazon SQS
- Audit logs
- Backups

### Encryption in Transit

Communication should use secure encrypted connections between:

- Customers and ClaimGuard
- ClaimGuard application components
- AWS services
- Adjusters and the dashboard

### Proposed AWS Capabilities

- AWS Key Management Service (KMS)
- TLS / HTTPS
- Service-specific encryption controls

### Architecture Principle

Sensitive claim information should remain protected both at rest and in transit.

Access to encryption keys should follow least-privilege principles.

---

## Security Incident Flow

Suspicious activity detected
↓
Account / session automatically contained
↓
Active access revoked
↓
Successful and denied attempts preserved
↓
Affected claims and documents identified
↓
Security and authorized management notified
↓
Audit evidence protected
↓
Investigation performed
↓
Identity securely re-verified
↓
Authorized security personnel approve restoration
↓
Access restored with appropriate monitoring
