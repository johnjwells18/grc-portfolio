# PROMPT & OUTPUT HANDLING STANDARD
## BlueNova Health: Data Governance, Logging, and Retention

**Document ID:** HELIX-POL-LOG-001  
**Classification:** Internal Policy - Security Sensitive  
**Effective Date:** January 15, 2025  
**Last Reviewed:** February 2025  
**Next Review Date:** July 15, 2025  
**Owner:** Privacy Officer & Security Lead  
**Approvals:**
- ✅ Chief Privacy Officer
- ✅ Chief Information Security Officer
- ✅ Chief Compliance Officer
- ✅ Chief Medical Officer (Clinical appropriateness)

---

## 1. PURPOSE & SCOPE

### 1.1 Purpose
This standard defines how Helix Clinical Note Summarizer (HCNS) processes, logs, stores, and retains:
- **Prompts** (patient notes, clinician queries, context sent to AI model)
- **Model Outputs** (AI-generated summaries, responses)
- **Metadata** (user IDs, timestamps, system logs, error traces)

**Goals:**
1. **Privacy Compliance:** Minimize data exposure; protect PHI/PII from unauthorized access and retention
2. **Audit Compliance:** Maintain forensic evidence for incident investigation and regulatory audit
3. **Vendor Accountability:** Ensure third-party vendors (LLM, vector store) don't retain or misuse data
4. **Operational Visibility:** Capture sufficient logs to detect anomalies, troubleshoot issues, investigate incidents

### 1.2 Scope
This policy applies to:
- **In-Scope Data:** Patient notes, clinician queries, AI model outputs, access logs, API metadata
- **In-Scope Systems:** HCNS API, prompt encoding layer, OpenAI API integration, vector store (Pinecone), CloudWatch, S3 audit logs
- **In-Scope Personnel:** Engineering (implementation), IT (operations), Security (monitoring), Compliance (retention verification)
- **Out-of-Scope:** General email/messaging logs, employee activity outside HCNS, external system logs (EHR, SSO)

---

## 2. LOGGING REQUIREMENTS

### 2.1 What Data Gets Logged & Where

**Logging Matrix:** For each data element, specify whether it's logged, to what destination, encryption status, and retention period.

#### **Data Element: User/Clinician Identity**

| Aspect | Logged? | Details | Storage | Encryption | Retention | Rationale |
|---|---|---|---|---|---|---|
| **Clinician ID** (Okta sub, UUID) | ✅ YES | OAuth 2.0 sub claim from Okta | CloudWatch + S3 | TLS + KMS | 7 years | Access control audit; trace which clinician made request |
| **Clinician Full Name/Email** | ❌ NO | Do not capture; use ID only | N/A | N/A | N/A | Privacy minimization; only need ID for audit trail |
| **IP Address** | ✅ YES | Source IP of API request (for anomaly detection) | CloudWatch | TLS + KMS | 90 days | Detect impossible travel, account compromise |
| **User-Agent / Device Info** | ✅ YES | Browser/app identifier | CloudWatch | TLS + KMS | 90 days | Detect unauthorized device access |

#### **Data Element: Request & Response Metadata**

| Aspect | Logged? | Details | Storage | Encryption | Retention | Rationale |
|---|---|---|---|---|---|---|
| **Timestamp** (request received) | ✅ YES | ISO 8601 format, UTC | CloudWatch + S3 | TLS + KMS | 7 years | Build audit timeline; detect timing anomalies |
| **Endpoint/API Method** | ✅ YES | e.g., POST /api/v1/summarize | CloudWatch + S3 | TLS + KMS | 7 years | Track which features are used |
| **Request Status** (200, 400, 401, 500) | ✅ YES | HTTP status code | CloudWatch + S3 | TLS + KMS | 7 years | Detect failed auth, API errors |
| **Response Status** (success/failure, error code) | ✅ YES | Business-logic status (e.g., "hallucination_detected", "timeout") | CloudWatch + S3 | TLS + KMS | 7 years | Understand outcome of request |
| **Request Latency** | ✅ YES | Time in milliseconds (ms) to process | CloudWatch + S3 | TLS + KMS | 90 days | Performance monitoring, detect slow API |
| **Bytes Sent/Received** | ✅ YES | Request size, response size (not content) | CloudWatch | TLS + KMS | 90 days | Capacity planning, detect data exfiltration attempts |

#### **Data Element: Prompts & Clinical Input**

| Aspect | Logged? | Details | Storage | Encryption | Retention | Rationale |
|---|---|---|---|---|---|---|
| **Patient Note (Raw)** | ❌ NO | Full clinical note text NEVER logged | N/A | N/A | N/A | Contains PHI; exceeds necessity; exceeds utility |
| **Patient Note (Redacted)** | ❌ NO | Even with redaction, risk of PHI recovery high; not logged | N/A | N/A | N/A | Abundance of caution |
| **Prompt Hash (SHA-256)** | ✅ YES (optional) | Deterministic hash of full prompt; enables forensics without storing data | CloudWatch | TLS + KMS | 90 days | Forensic investigation (e.g., "was this jailbreak prompt tried?"); not reversible |
| **Prompt Length (tokens)** | ✅ YES | Number of tokens in prompt (proxy for size) | CloudWatch + S3 | TLS + KMS | 7 years | Model monitoring (prompt size trends), capacity planning |
| **PII Detected (Category)** | ✅ YES (tokenized) | Redacted log entry: "SSN, DOB detected and redacted" | CloudWatch | TLS + KMS | 30 days | Alert if abnormal patterns (e.g., many SSNs in one note = potential exfiltration attempt) |
| **Clinician Query** (if free-text) | ⚠️ CONDITIONAL | Log if short/low-risk (e.g., "Summarize cardiac history"); do not log if contains PII or sensitive clinical data | CloudWatch | TLS + KMS | 30 days | Feature usage; avoid capturing sensitive clinician queries |

#### **Data Element: Model Outputs**

| Aspect | Logged? | Details | Storage | Encryption | Retention | Rationale |
|---|---|---|---|---|---|---|
| **Summary Output (Raw)** | ❌ NO | Full AI-generated summary NEVER logged | N/A | N/A | N/A | Contains PHI + hallucinated info; creates liability |
| **Summary Output (Truncated)** | ❌ NO | Even truncation doesn't mitigate PHI risk; not logged | N/A | N/A | N/A | Abundance of caution |
| **Output Length (tokens)** | ✅ YES | Number of tokens in response | CloudWatch + S3 | TLS + KMS | 7 years | Model monitoring, output distribution trends |
| **Confidence Score** | ✅ YES | Model confidence (0.0-1.0) output for this request | CloudWatch + S3 | TLS + KMS | 7 years | Hallucination detection tuning, model health |
| **Hallucination Flag** | ✅ YES | Boolean flag: was hallucination detected? | CloudWatch + S3 | TLS + KMS | 7 years | Track hallucination rates, trends over time |
| **Manual Review Required Flag** | ✅ YES | Boolean: does clinician need to review output? | CloudWatch + S3 | TLS + KMS | 7 years | Track high-risk outputs requiring validation |
| **Clinical Appropriateness Score** | ✅ YES | Internal model metric (0-100); indicates likelihood output is clinically sensible | CloudWatch + S3 | TLS + KMS | 7 years | Quality monitoring; alert if consistently low |

#### **Data Element: Model & System Configuration**

| Aspect | Logged? | Details | Storage | Encryption | Retention | Rationale |
|---|---|---|---|---|---|---|
| **Model Version** | ✅ YES | e.g., "gpt-4-turbo-2024-01-15" | CloudWatch + S3 | TLS + KMS | 7 years | Trace model changes; enable reproducibility |
| **Model Temperature** | ✅ YES | Temperature parameter (0.1 for consistency) | CloudWatch + S3 | TLS + KMS | 7 years | Detect unauthorized parameter changes |
| **API Provider Status** | ✅ YES | OpenAI API: "success", "timeout", "rate_limit", "error" | CloudWatch | TLS + KMS | 90 days | Vendor SLA monitoring |
| **API Response Time** (vendor) | ✅ YES | Milliseconds for OpenAI API call | CloudWatch | TLS + KMS | 90 days | Vendor performance monitoring |
| **Fallback Activated?** | ✅ YES | Boolean: was fallback (cached/local model) used? | CloudWatch + S3 | TLS + KMS | 7 years | Track system resilience |

#### **Data Element: Security & Anomalies**

| Aspect | Logged? | Details | Storage | Encryption | Retention | Rationale |
|---|---|---|---|---|---|---|
| **Auth Success/Failure** | ✅ YES | OAuth token validation: success or failure reason | CloudWatch + S3 | TLS + KMS | 7 years | Detect brute force, compromised credentials |
| **Failed Auth Attempts** (per user/IP) | ✅ YES | Count of failed logins in 1-min window | CloudWatch | TLS + KMS | 90 days | Trigger rate limiting; alert SOC |
| **MFA Status** | ✅ YES | MFA verified, MFA skipped (why?), MFA failed | CloudWatch + S3 | TLS + KMS | 7 years | Compliance; ensure MFA enforcement |
| **Rate Limit Exceeded?** | ✅ YES | Boolean: was request rate-limited? | CloudWatch | TLS + KMS | 90 days | Detect DoS attacks; capacity monitoring |
| **Input Validation Failure** | ✅ YES | Reason: "injection_pattern_detected", "token_limit_exceeded", etc. | CloudWatch | TLS + KMS | 90 days | Security monitoring; alert on repeated attacks |
| **Data Residue Detected?** | ✅ YES | If memory/temp file cleanup fails, log incident | CloudWatch | TLS + KMS | 90 days | Investigate data residue incidents |
| **Error Message** (sanitized) | ✅ YES | Generic error message to user; no sensitive stack traces in logs | CloudWatch | TLS + KMS | 90 days | Troubleshooting; avoid exposing system details |

---

### 2.2 Example Log Entry (Fully Compliant)

```json
{
  "timestamp": "2025-02-15T14:32:45.123Z",
  "clinician_id": "user_8a9f2c1d",
  "ip_address": "203.0.113.42",
  "endpoint": "POST /api/v1/summarize",
  "request_auth_status": "success",
  "mfa_verified": true,
  "request_size_bytes": 2847,
  "prompt_length_tokens": 892,
  "pii_detected": ["DOB_REDACTED", "SSN_REDACTED"],
  "prompt_hash_sha256": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
  "openai_api_call_status": "success",
  "openai_response_time_ms": 3200,
  "model_version": "gpt-4-turbo-2024-01-15",
  "model_temperature": 0.1,
  "response_length_tokens": 412,
  "confidence_score": 0.87,
  "hallucination_detected": false,
  "manual_review_required": false,
  "clinical_appropriateness_score": 92,
  "fallback_model_used": false,
  "request_latency_ms": 3450,
  "http_status_code": 200,
  "business_status": "success",
  "error_message": null,
  "rate_limit_exceeded": false
}
```

**What's NOT in the log:**
- ❌ Clinician name (only ID)
- ❌ Patient name, MRN, DOB (PII detected & redacted at ingestion)
- ❌ Full prompt text (hash only)
- ❌ Full summary text (length only)
- ❌ Stack traces or system-level errors (generic messages only)

---

## 3. EXTERNAL API LOGGING (OPENAI)

### 3.1 OpenAI API Configuration

**Critical Control:** HCNS must NOT allow OpenAI to log or retain prompts/responses.

#### **Implementation:**

```python
# Python code snippet showing control implementation
response = openai.ChatCompletion.create(
    model="gpt-4-turbo-2024-01-15",
    messages=[{"role": "user", "content": redacted_prompt}],
    temperature=0.1,
    max_tokens=500,
    # ========== CRITICAL: DISABLE LOGGING ==========
    disable_logs=True,  # Prevent OpenAI from logging/training on this request
    # ================================================
    timeout=30
)
```

#### **Verification (Quarterly Audit):**
1. Code review: Confirm `disable_logs=True` is set for all production API calls
2. Vendor attestation: OpenAI confirms no BlueNova data is in training corpus or operator logs
3. API documentation review: Confirm API version still supports `disable_logs` parameter
4. Compliance report: Document verification in quarterly GRC report

### 3.2 Prompt Content Before OpenAI Call

**Process:**

```
Raw Patient Note (contains PHI)
         ↓
Input Validation (check length, format)
         ↓
PII Redaction Layer (SSN → [SSN_REDACTED], DOB → [DOB_REDACTED])
         ↓
Prompt Encoding (format into system prompt + user message)
         ↓
Request Logging (log metadata only: length, hash, status)
         ↓
OpenAI API Call (with disable_logs=True, using redacted content)
         ↓
Response Received (log metadata: length, confidence, hallucination flag)
         ↓
Response Validation (check for hallucinations, safety violations)
         ↓
Return to Clinician (via HCNS API)
```

**Key Point:** Raw clinical note is NEVER sent to OpenAI; only redacted version is transmitted.

---

## 4. LOG ACCESS & PROTECTION

### 4.1 Who Can Access Logs?

| Role | Access Level | Approval Required | Audit Log |
|---|---|---|---|
| **Security Lead** | Full logs (7-year archive) | None (role-based) | ✅ Yes |
| **Compliance Officer** | Audit logs only (user access, auth, admin actions) | None (role-based) | ✅ Yes |
| **CISO** | Full logs; access for investigation | Self-approved | ✅ Yes |
| **System Administrator** | Infrastructure/performance logs only (no audit data) | CISO approval | ✅ Yes |
| **IT Help Desk** | Aggregate metrics only; no individual log access | Denied (no access) | ✅ N/A |
| **Engineering/DevOps** | Logs for deployed code only (errors, performance) | Engineering Lead approval | ✅ Yes |
| **External Auditors** (SOC 2, HIPAA) | Limited logs per audit scope | Compliance Officer + CISO approval | ✅ Yes (redacted) |
| **Law Enforcement** | Per legal warrant/subpoena | Legal counsel approval | ✅ Yes |

### 4.2 Access Request Process

**Step 1: Request Submission**
- Requestor fills out form: Name, role, business justification, date range needed, data sensitivity level
- Example: "Investigating suspected account compromise on 2025-02-15; need full audit logs for user_8a9f2c1d for 2025-02-10 to 2025-02-16"

**Step 2: Approval**
- Security Lead (or CISO for sensitive requests) reviews request
- Approves/denies within 24 hours
- Denial reasons: insufficient justification, out-of-retention window, excessive scope

**Step 3: Access Granted (Time-Limited)**
- Access window: Default 24 hours; max 7 days
- Logging: Requestor identity, reason, timestamp, duration tracked
- Monitoring: Automated alert if requestor exports logs or accesses outside approved window

**Step 4: Audit Trail**
- All log access requests and approvals logged in immutable audit log
- Quarterly review: Security Lead verifies no unauthorized log access

---

### 4.3 Log Storage & Encryption

#### **CloudWatch Logs (Short-Term, Operational)**
- **Retention:** 90 days (then deleted)
- **Encryption:** TLS in-transit; KMS encryption at-rest (AWS-managed key)
- **Access Control:** IAM policy restricts to Security Lead, CISO, System Administrator
- **Immutability:** Read-only to authorized users (no deletion, no modification)

#### **S3 Audit Log Bucket (Long-Term, Archive)**
- **Retention:** 7 years (HIPAA legal hold requirement)
- **Encryption:** TLS in-transit; KMS encryption at-rest (customer-managed key)
- **Immutability:** S3 Object Lock (WORM = Write-Once-Read-Many)
  - Objects cannot be deleted or modified once written
  - Governance mode: CISO can override (with audit trail)
  - Compliance mode: Cannot be overridden, even by root account
- **Versioning:** Enabled; old versions retained for legal holds
- **Access Control:** Only Security Lead, CISO, Compliance Officer can read
- **Lifecycle Policy:**
  - 0–30 days: S3 Standard (hot tier; fast access)
  - 30–365 days: S3 Intelligent-Tiering (auto-optimize access pattern)
  - 365–2555 days (7 years): S3 Glacier (cold storage; ~$0.004/GB/month; 12-hour retrieval)

#### **Deletion After 7-Year Retention**
- Automated S3 lifecycle policy deletes objects on day 2555 (7 years + 1 day)
- Deletion logged in CloudTrail (immutable log of AWS API actions)
- Compliance Officer verifies deletion monthly

---

## 5. LOG REDACTION RULES

### 5.1 Automatic Redaction (Applied to All Logs)

| PII Pattern | Regex Match | Replacement | Example |
|---|---|---|---|
| **SSN** | `\d{3}-\d{2}-\d{4}` | `[SSN_REDACTED]` | "123-45-6789" → "[SSN_REDACTED]" |
| **Medical Record Number (MRN)** | `\b[A-Z]{2}\d{6}\b` | `[MRN_REDACTED]` | "CD123456" → "[MRN_REDACTED]" |
| **Date of Birth** | `\d{1,2}/\d{1,2}/\d{4}` | `[DOB_REDACTED]` | "01/15/1990" → "[DOB_REDACTED]" |
| **Phone Number** | `\d{3}-\d{3}-\d{4}` | `[PHONE_REDACTED]` | "555-123-4567" → "[PHONE_REDACTED]" |
| **Email Address** | `[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+` | `[EMAIL_REDACTED]` | "john.doe@hospital.com" → "[EMAIL_REDACTED]" |
| **Patient Full Name** (if captured) | Pattern-based + dictionary | `[PATIENT_NAME_REDACTED]` | "Jane Smith" → "[PATIENT_NAME_REDACTED]" |
| **Clinician Full Name** (if captured) | Pattern-based + dictionary | `[CLINICIAN_NAME_REDACTED]` | "Dr. Robert Johnson" → "[CLINICIAN_NAME_REDACTED]" |

### 5.2 Implementation (Lambda Function)

Redaction happens in real-time as logs are written to CloudWatch/S3:

```python
# Python pseudocode for log redaction function
import re

def redact_pii(log_entry):
    """Apply all redaction rules to log entry before storage"""
    redaction_rules = {
        'ssn': (r'\d{3}-\d{2}-\d{4}', '[SSN_REDACTED]'),
        'mrn': (r'\b[A-Z]{2}\d{6}\b', '[MRN_REDACTED]'),
        'dob': (r'\d{1,2}/\d{1,2}/\d{4}', '[DOB_REDACTED]'),
        # ... more rules
    }
    
    redacted = log_entry
    for rule_name, (pattern, replacement) in redaction_rules.items():
        redacted = re.sub(pattern, replacement, redacted, flags=re.IGNORECASE)
    
    return redacted

# Apply redaction before S3 write
redacted_log = redact_pii(raw_log)
s3_client.put_object(Bucket='helix-audit-logs', Key=f'logs/{timestamp}.json', Body=redacted_log)
```

### 5.3 Verification & Testing

- **Weekly:** Automated test runs redaction rules on sample logs
- **Monthly:** Manual review of redaction effectiveness (sampling)
- **Quarterly:** Audit: verify no PII in stored logs (sample 100 log entries)

---

## 6. DATA RETENTION POLICY

### 6.1 Retention Schedule by Data Type

| Data Type | Retention Period | Rationale | Deletion Method |
|---|---|---|---|
| **Audit Logs** (user access, auth, admin) | 7 years | HIPAA § 164.312(b) legal hold | Automated S3 lifecycle policy |
| **API Logs** (metadata, status, errors) | 7 years | Incident investigation, reproducibility | Automated S3 lifecycle policy |
| **Security Logs** (auth, anomalies, rate limit) | 7 years | Forensic investigation, compliance | Automated S3 lifecycle policy |
| **Performance Logs** (latency, errors, metrics) | 90 days | Trend analysis, capacity planning | Automated CloudWatch retention |
| **Prompt/Response Hash** (forensic only) | 90 days | Forensic incident investigation window | Automated CloudWatch + S3 deletion |
| **IP Address** (from logs) | 90 days | Detect impossible travel, account compromise | Automated CloudWatch deletion |
| **Vendor API Call Status** (OpenAI) | 90 days | SLA monitoring, vendor accountability | Automated CloudWatch deletion |
| **PII Detection Alerts** | 30 days | Alert trend analysis; reduce storage risk | Automated CloudWatch + S3 deletion |

### 6.2 Legal Hold & Exception to Deletion

**Scenario:** Investigation opens or litigation threatened
- **Action:** Compliance Officer places "legal hold" on relevant logs
- **Effect:** Logs subject to legal hold are NOT deleted even after retention expires
- **Duration:** Hold remains until legal matter resolved or 10 years (max), whichever comes first
- **Evidence:** Legal hold documented in Compliance system with legal matter reference

---

## 7. COMPLIANCE VERIFICATION

### 7.1 Weekly Verification Tasks

| Task | Owner | Success Criteria | Evidence |
|---|---|---|---|
| **Log Integrity Check** | Infra Lead | No corruption, no missing entries; append-only verified | Integrity check report; S3 Object Lock validation |
| **Encryption Status** | Infra Lead | All logs encrypted (TLS, KMS); no unencrypted backups | AWS CloudTrail showing KMS key usage |
| **Access Control Verification** | Security Lead | Only authorized users accessed logs; all access logged | Access log audit; IAM policy review |

### 7.2 Monthly Verification Tasks

| Task | Owner | Success Criteria | Evidence |
|---|---|---|---|
| **Access Log Review** | Security Lead | All log access justified, approved, time-limited; no suspicious access | Access request log summary |
| **Retention Policy Verification** | Compliance Officer | Logs deleted per schedule; no premature deletion, no over-retention | S3 lifecycle policy execution log |
| **Redaction Effectiveness** | Security Lead | Sample logs show PII redacted; no PII leakage | Manual review of 50–100 log entries |

### 7.3 Quarterly Verification Tasks

| Task | Owner | Success Criteria | Evidence |
|---|---|---|---|
| **OpenAI Logging Audit** | Compliance Officer | OpenAI confirms disable_logs=True; no BlueNova data in OpenAI logs | OpenAI attestation letter; API config audit |
| **Full Audit** (HIPAA § 164.312(b)) | Compliance Officer | All controls operating as designed; no findings | Audit report with test results |
| **Compliance Report** | Compliance Officer | Evidence of logging compliance; controls effective | Quarterly GRC report |

---

## 8. RELATED POLICIES & STANDARDS

This policy references:
- **HELIX-POL-AUP-001:** AI Acceptable Use Standard (user responsibilities)
- **HELIX-SEC-REQS:** Security & Privacy Requirements for Engineering (technical implementation)
- **BlueNova Health Privacy Policy:** Data minimization, user rights
- **BlueNova Health Data Classification Policy:** What data is PHI, PII, confidential
- **BlueNova Health Incident Response Policy:** Breach notification, investigation
- **HIPAA Security Rule (§ 164.312):** Audit controls, encryption, access controls

---

## 9. DEFINITIONS

| Term | Definition |
|---|---|
| **Prompt** | User input (patient note, clinician query) sent to HCNS |
| **Model Output** | AI-generated response from HCNS |
| **Metadata** | Contextual information about request/response (timestamp, user ID, status code) |
| **Logging** | Capturing and storing system events, requests, errors, and access |
| **Redaction** | Automated removal or masking of sensitive data (PII, PHI) from logs |
| **Retention** | How long logs are stored before deletion |
| **Legal Hold** | Preservation of logs beyond normal retention due to litigation or investigation |
| **Immutable Log** | Log that cannot be deleted or modified after creation; append-only |
| **Forensics** | Investigation of security incidents using logs and system artifacts |
| **Compliance Audit** | Review of logging controls against regulatory requirements (HIPAA, SOC 2) |

---

## 10. APPROVAL & SIGN-OFF

**Approved By:**

| Role | Name | Signature | Date |
|---|---|---|---|
| **Chief Privacy Officer** | [Privacy Officer Name] | ___________ | 01/15/2025 |
| **Chief Information Security Officer** | [CISO Name] | ___________ | 01/15/2025 |
| **Chief Compliance Officer** | [Compliance Officer Name] | ___________ | 01/15/2025 |
| **Chief Medical Officer** | [CMO Name] | ___________ | 01/15/2025 |

**Document Control:**
- Version: 1.0
- Effective Date: January 15, 2025
- Last Updated: February 2025
- Next Review: July 15, 2025
- Status: APPROVED & ACTIVE

---

**END OF POLICY DOCUMENT**
