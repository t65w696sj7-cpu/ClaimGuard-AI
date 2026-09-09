# ClaimGuard AI — Architecture Decisions

## Purpose

This document records the major architecture decisions made for ClaimGuard AI following the initial NorthStar Mutual stakeholder discovery exercise.

Each decision is tied to a business, security, reliability, performance, or AI requirement rather than selecting technology without a defined need.

---

## AD-01 — Preserve Claims Before Confirmation

### Decision
ClaimGuard must securely preserve an accepted claim before telling the customer that the claim has been successfully received.

### Reason
A customer should never receive a successful submission confirmation for a claim that could subsequently be lost because of a system failure.

### Business Impact
Customers receive reliable confirmation and should not need to resubmit successfully accepted claims.

---

## AD-02 — Separate Claim Intake From AI Processing

### Decision
Claim submission must operate independently from AI processing.

### Reason
Customers should not have to wait for AI analysis before ClaimGuard confirms receipt of their claim. AI outages must not prevent claim submission.

### Tradeoff
AI results may not be immediately available after submission.

### Business Impact
Claim intake remains available even when AI processing is delayed or unavailable.

---

## AD-03 — Priority-Based Processing Queue

### Decision
Claims awaiting downstream processing will enter an organized priority-based processing queue.

Initial priority will be determined using NorthStar-defined business rules, while authorized management may escalate or override priority when appropriate.

### Reason
Catastrophe events may produce claim volumes exceeding immediate AI processing capacity.

### Tradeoff
Lower-priority claims may wait longer during extreme demand.

### Business Impact
Critical and high-priority claims can receive attention while protecting the system from overload.

---

## AD-04 — Separate Structured Data From Documents

### Decision
Structured claim information will be stored separately from large claim documents.

Structured information may include claim number, policy information, claim status, priority, timestamps, assignment, and processing state.

Photos, PDFs, police reports, estimates, and other large documents will use scalable object storage.

### Reason
Structured transactional data and large unstructured files have different storage and access requirements.

### Business Impact
The architecture can scale document storage independently while maintaining efficient access to structured claim information.

---

## AD-05 — Preserve Original Evidence

### Decision
Original submitted documents and images must be preserved without destructive modification.

Enhanced or optimized copies may be generated for viewing, document processing, OCR, image enhancement, or AI analysis.

### Reason
Compression, enhancement, or other processing should not destroy or alter the original evidence submitted with the claim.

### Business Impact
NorthStar retains original claim evidence while still allowing ClaimGuard to create processing-friendly versions.

---

## AD-06 — Storage Lifecycle Management

### Decision
Frequently accessed active-claim documents will remain in fast-access storage.

Eligible closed-claim documents may transition to lower-cost archival storage according to NorthStar retention requirements.

### Reason
Closed claims may need to be retained for long periods but accessed infrequently.

### Tradeoff
Archived information may take longer to retrieve.

### Business Impact
NorthStar can reduce long-term storage costs without prematurely deleting required claim information.

---

## AD-07 — Role-Based and Claim-Level Access

### Decision
ClaimGuard will use role-based access controls combined with claim-level authorization.

Customers, adjusters, managers, security/audit personnel, and system administrators will receive different permissions based on their responsibilities.

### Reason
Successful authentication alone should not provide unrestricted access to customer information.

### Business Impact
Sensitive claim information is accessible only to authorized users with a legitimate business need.

---

## AD-08 — Multi-Factor Authentication

### Decision
Privileged workforce access will require multi-factor authentication in addition to primary credentials.

### Reason
A stolen username and password should not automatically provide access to sensitive claims information.

### Business Impact
The architecture provides additional protection against account compromise.

---

## AD-09 — Suspicious Activity Containment

### Decision
ClaimGuard should detect abnormal access behavior such as unusual bulk downloads and support rapid containment of the affected account or session.

Security personnel must be alerted and relevant audit evidence preserved.

### Reason
A successfully authenticated account may still be compromised.

### Business Impact
Potential breaches can be contained without unnecessarily shutting down claim access for the entire organization.

---

## AD-10 — AI Must Remain Advisory

### Decision
ClaimGuard AI will assist with summarization, document analysis, missing-information detection, and identification of conflicting information.

AI will not independently approve or deny claims.

### Reason
Consequential insurance decisions require authorized human judgment.

### Business Impact
NorthStar gains AI-assisted efficiency while retaining human decision authority.

---

## AD-11 — Ground AI Responses in Claim Evidence

### Decision
AI-generated factual statements should be grounded in authorized claim documentation and provide supporting source references when possible.

Unsupported AI-generated statements must not be presented as confirmed claim facts.

### Reason
Generative AI may produce information that is not supported by the underlying documents.

### Business Impact
Adjusters can verify important AI-generated information against source evidence.

---

## AD-12 — Conflicting Evidence Requires Human Review

### Decision
When authorized claim documents contain conflicting information, ClaimGuard should identify and present the conflict rather than determining which party is truthful.

The claim may be escalated for human review and additional evidence gathering.

### Reason
Conflicting legitimate evidence cannot safely be resolved through unsupported AI inference.

### Business Impact
ClaimGuard supports adjusters without replacing their responsibility for consequential judgments.

---

## Next Architecture Phase

These decisions will now be mapped to appropriate AWS services.

Service selection will consider:

- Security
- Scalability
- Reliability
- Performance
- Operational complexity
- Cost
- AI safety
- Business requirements

AWS services will be selected because they satisfy documented requirements, not simply because they are available.
