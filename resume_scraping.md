# Resume Scraping & Job Application Intake — Logic Design

> [!NOTE]
> This document defines the complete logic, state machine, node flow, and edge case
> analysis for the new Application Intake pipeline that prepends to the existing
> Offer Letter Automation workflow.

---

## 1. APPLICATION STATES

Every application record in the **Applications Sheet** will carry exactly one `status` value at any time.

| State | Meaning | Set By |
|---|---|---|
| `pending` | Application received, logged, not yet evaluated for selection | Step 9 — Log to Sheet |
| `no_resume` | Email was a job application but had no attachment | Step 5b — Attachment Check |
| `parse_failed` | Resume attached but text too short / unreadable | Step 6b — Text Validation |
| `duplicate` | Same email + role combo already exists in Applications Sheet | Step 8b — Duplicate Check |
| `role_not_open` | Applied role doesn't exist or has 0 open seats configured | Step 10b — Role Config Check |
| `low_confidence` | AI classified as job application but confidence < 0.85 | Step 2b — Confidence Gate |
| `waitlisted` | Position already filled (all slots taken) | Step 11 — Selection Node |
| `selected` | Passed selection, added to Employees sheet | Step 12 — Selection Node |
| `offer_triggered` | Confirmed present in Employees sheet, offer letter workflow started | External workflow |

### State Transition Diagram

```
Email Received
      │
      ▼
[AI Classify]
   ├─ NOT application ──────────────────────────────► AUTO-REPLY (no state logged)
   ├─ confidence < 0.85 ──────► low_confidence ──────► HR Review Queue
      │
      ▼
[Has Attachment?]
   ├─ No ──────────────────────► no_resume ──────────► "Please resend" email
      │
      ▼
[Extract Text]
   ├─ text < 100 chars ─────────► parse_failed ───────► HR Notified
      │
      ▼
[AI Parse Resume → JSON]
      │
      ▼
[Duplicate Check: email+role in Sheet?]
   ├─ YES ──────────────────────► duplicate ──────────► "Already received" email
      │
      ▼
[Role in Openings Config?]
   ├─ NO ───────────────────────► role_not_open ───────► "No openings" email
      │
      ▼
[Log to Applications Sheet]  ← status: pending
      │
      ▼
[Submit to Google Form]
      │
      ▼
[Read Applications Sheet → Count selected for role]
      │
      ▼
[count >= openings[role]?]
   ├─ YES ──────────────────────► waitlisted ──────────► Acknowledgement email
   └─ NO ───────────────────────► selected ────────────► Add to Employees Sheet
                                                               │
                                                               ▼
                                                   [status: offer_triggered]
                                               (picked up by Offer Letter workflow)
```

---

## 2. NODE-BY-NODE LOGIC

### NODE 1 — IMAP Email Trigger
- **Polls:** Every 5 minutes
- **Reads:** Subject, Body (plain text + HTML), Attachments (binary), Sender email, Received timestamp
- **Output fields:** `email_from`, `email_subject`, `email_body`, `attachments[]`, `received_at`

---

### NODE 2 — AI Agent: Classify Email
- **Input:** `email_subject` + `email_body` (first 1500 chars)
- **Prompt instruction:**
  ```
  You are an HR email classifier. Read the email below and determine:
  1. Is this a job application? (true/false)
  2. If yes, what role are they applying for? (exact string or "unknown")
  3. Your confidence score (0.0 to 1.0)

  Respond ONLY in JSON: {"is_job_application": bool, "applied_role": string, "confidence": float}
  ```
- **Output fields:** `is_job_application`, `applied_role`, `confidence`

---

### NODE 2b — Confidence Gate (IF node)
- **Condition A:** `confidence >= 0.85` AND `is_job_application == true` → continue
- **Condition B:** `confidence < 0.85` AND `is_job_application == true` → `low_confidence` path
- **Condition C:** `is_job_application == false` → not-application path

---

### NODE 3 — [Branch: Not Application]
- Send auto-reply via Hostinger SMTP
- Mark email as read
- **END**

### NODE 3b — [Branch: Low Confidence]
- Append row to Applications Sheet: `status=low_confidence`, `hr_review=TRUE`
- Send internal Slack/email alert to HR: "Uncertain application — manual review needed"
- **END**

---

### NODE 4 — Extract Attachment
- **Reads:** `attachments[]` from IMAP trigger output
- **Checks:** `attachments.length > 0`
- **Accepted types:** `application/pdf`, `application/msword`, `application/vnd.openxmlformats-officedocument.wordprocessingml.document`

---

### NODE 5 — Has Attachment? (IF node)
- **TRUE:** Has at least one valid attachment → continue
- **FALSE:** No attachment or wrong file type → `no_resume` path

### NODE 5b — [Branch: No Resume]
- Append row to Applications Sheet: `email=sender`, `applied_role`, `status=no_resume`, `applied_at=now`
- Send reply: "We received your application email but couldn't find your resume. Please reply with your resume attached."
- **END**

---

### NODE 6 — Extract Text From File
- **Input:** Binary data of first valid attachment
- **Method:** `n8n-nodes-base.extractFromFile` (handles PDF + DOCX)
- **Output:** `resume_raw_text` (string)

---

### NODE 6b — Validate Extracted Text (Code Node)
```javascript
const text = $json.resume_raw_text || '';
const isReadable = text.trim().length >= 100;
return [{ json: { ...items[0].json, resume_raw_text: text, text_valid: isReadable } }];
```
- **IF `text_valid == false`:** → `parse_failed` path

### NODE 6c — [Branch: Parse Failed]
- Append row to Applications Sheet: `status=parse_failed`, `email=sender`, `applied_at=now`
- Send internal HR alert: "Resume received from [sender] but could not be read. Manual processing needed."
- **END**

---

### NODE 7 — AI Agent: Parse Resume
- **Input:** `resume_raw_text` (trimmed to first 3000 chars if longer)
- **Prompt instruction:**
  ```
  Extract the following fields from this resume text.
  Respond ONLY in JSON with these exact keys:
  {
    "name": string,
    "email": string,
    "phone": string,
    "applied_role": string (use the role from classification if resume is ambiguous),
    "total_experience_years": number,
    "current_or_last_company": string,
    "highest_education": string,
    "key_skills": [array of strings, max 10],
    "resume_summary": string (2 sentences max)
  }
  If a field is not found, use null.
  ```
- **Output fields:** `name`, `email`, `phone`, `applied_role`, `total_experience_years`, `current_or_last_company`, `highest_education`, `key_skills`, `resume_summary`

> [!NOTE]
> `applied_role` in resume parse is cross-checked with `applied_role` from Node 2 classification. Node 2's value takes priority if they conflict.

---

### NODE 8 — Validate Parsed Fields (Code Node)
```javascript
const d = $json;
const required = ['name', 'email', 'applied_role'];
const missing = required.filter(f => !d[f] || d[f] === null);
const isValid = missing.length === 0;
return [{ json: { ...d, parse_valid: isValid, missing_fields: missing } }];
```

---

### NODE 8b — Duplicate Check (Google Sheets Read)
- **Query:** Read Applications Sheet, filter where `email == parsed_email` AND `applied_role == applied_role`
- **IF match found:** → `duplicate` path

### NODE 8c — [Branch: Duplicate]
- Update existing row: `duplicate_attempt_at = now`, `duplicate_count += 1`
- Send reply: "We already have your application for [role] received on [original_date]. We'll be in touch!"
- **END**

---

### NODE 9 — Role Config Check (Code Node)
The openings config is stored as a hardcoded object (or read from a Config Sheet):
```javascript
const OPENINGS = {
  "Software Engineer": 2,
  "HR Executive": 1,
  "Marketing Lead": 1,
  "Data Analyst": 0,    // 0 = closed
};

const role = $json.applied_role;
const slots = OPENINGS[role];
const roleExists = slots !== undefined;
const roleOpen = roleExists && slots > 0;

return [{ json: { ...$json, role_slots: slots, role_exists: roleExists, role_open: roleOpen } }];
```
- **IF `role_open == false`:** → `role_not_open` path

### NODE 9b — [Branch: Role Not Open]
- Append row: `status=role_not_open`, `email`, `applied_role`, `applied_at`
- Send reply: "Thank you for your interest in [role], but we currently don't have openings for this position. We'll keep your profile for future reference."
- **END**

---

### NODE 10 — Log to Applications Sheet (Google Sheets Append)
**Columns written:**
| Column | Value |
|---|---|
| `name` | from parsed resume |
| `email` | from parsed resume |
| `phone` | from parsed resume |
| `applied_role` | from classifier |
| `experience_years` | from parsed resume |
| `skills` | joined string |
| `education` | from parsed resume |
| `current_company` | from parsed resume |
| `resume_summary` | from parsed resume |
| `applied_at` | `$now.toISO()` |
| `source_email` | original sender email |
| `status` | `pending` |
| `ats_score` | *(blank — reserved for future)* |
| `hr_review` | `FALSE` |

---

### NODE 11 — Submit to Google Form (HTTP Request)
- **Method:** POST to Form submit URL
- **Body:** Pre-filled `entry.XXXXXXXX` fields mapped from parsed resume data
- `onError: continueRegularOutput` — form failure should not block selection

---

### NODE 12 — Count Selected Applicants (Google Sheets Read + Code)
```javascript
// Read all rows in Applications Sheet where applied_role matches
// and status == "selected"
const allRows = $input.all();
const role = $('Log to Applications Sheet').first().json.applied_role;
const selectedCount = allRows.filter(r =>
  r.json.applied_role === role && r.json.status === 'selected'
).length;

const slots = $('Role Config Check').first().json.role_slots;
const canSelect = selectedCount < slots;

return [{ json: { selectedCount, slots, canSelect } }];
```

---

### NODE 13 — Is Selected? (IF node)
- **TRUE (`canSelect == true`):** → Selected path
- **FALSE:** → Waitlisted path

---

### NODE 14a — [Branch: Selected]
1. **Update Applications Sheet:** `status=selected`, `selected_at=$now.toISO()`
2. **Add to Employees Sheet** (the one used by Offer Letter workflow):
   | Column | Value |
   |---|---|
   | `name` | parsed name |
   | `email` | parsed email |
   | `phone_number` | parsed phone |
   | `role` | applied_role |
   | `department` | *(mapped from role or left blank)* |
   | `salary` | *(blank — HR to fill)* |
   | `start_date` | *(blank — HR to fill)* |
   | `reporting_manager` | *(blank — HR to fill)* |
   | `sent` | `FALSE` |
   | `status` | *(empty — triggers offer letter workflow on next cycle)* |

> [!IMPORTANT]
> `salary`, `start_date`, and `reporting_manager` are left blank intentionally.
> The Offer Letter workflow will still process, but these fields need to be
> filled by HR before the schedule trigger picks them up. Add a validation
> step or instruct HR to complete those fields before activating the row.

---

### NODE 14b — [Branch: Waitlisted]
1. **Update Applications Sheet:** `status=waitlisted`, `waitlisted_at=$now.toISO()`
2. **Send acknowledgement email** via Hostinger SMTP:
   "Thank you [name] for applying for [role]. We've received your application and will reach out if a position opens up."

---

## 3. COMPLETE STATE TABLE

| Application Scenario | Final Status | Action Taken |
|---|---|---|
| Not a job application | *(not logged)* | Auto-reply sent |
| Low AI confidence | `low_confidence` | HR flagged for manual review |
| Job application, no attachment | `no_resume` | "Please resend" email sent |
| Unreadable/corrupted resume | `parse_failed` | HR notified |
| Duplicate application | `duplicate` | "Already received" reply sent |
| Unknown/closed role | `role_not_open` | "No openings" reply sent |
| Valid — slot available | `selected` → `offer_triggered` | Added to Employees Sheet → Offer letter workflow runs |
| Valid — slots full | `waitlisted` | Acknowledgement email sent |

---

## 4. DRY RUN — 5 EDGE CASES

---

### ✅ EDGE CASE 1 — Email with no resume attached

**Input:**
```
From: john@example.com
Subject: Application for Marketing Lead
Body: "Hello, I would like to apply for the Marketing Lead position. I am very interested."
Attachments: none
```

**Step-by-step trace (V1 logic — pre-fix):**
```
Node 1: Email received ✅
Node 2: AI → {is_job_application:true, applied_role:"Marketing Lead", confidence:0.92} ✅
Node 2b: confidence >= 0.85 ✅ → continue
Node 4: Extract attachment → attachments array = [] 
Node 5: Has attachment? → FALSE
```

**Result:** ✅ Routes correctly to `no_resume` path (Node 5b).
Log row written. "Please resend with resume" email sent. **PASS.**

---

### ✅ EDGE CASE 2 — Same person applies twice for same role

**Input (Day 1):**
```
From: priya@gmail.com | Role: Software Engineer | Status in Sheet: selected
```
**Input (Day 3 — Duplicate):**
```
From: priya@gmail.com | Subject: Application for Software Engineer (again)
Body: "Hi, sending my resume again for the Software Engineer role."
Attachments: resume_v2.pdf
```

**Step-by-step trace:**
```
Node 1: Email received ✅
Node 2: {is_job_application:true, applied_role:"Software Engineer", confidence:0.95} ✅
Node 5: Has attachment? → TRUE ✅
Node 6: Text extracted successfully ✅
Node 7: AI parses → name:Priya, email:priya@gmail.com ✅
Node 8b: Duplicate check → Sheet has row where email=priya@gmail.com AND role=Software Engineer ✅
```

**Result:** ✅ Routes to `duplicate` path (Node 8c).
"Already received your application" reply sent. **PASS.**

---

### ✅ EDGE CASE 3 — Scanned image-only PDF (unreadable resume)

**Input:**
```
From: ravi@gmail.com
Subject: Applying for Data Analyst role
Attachments: resume_scan.pdf (scanned image, no text layer)
```

**Step-by-step trace:**
```
Node 1: Email received ✅
Node 2: {is_job_application:true, applied_role:"Data Analyst", confidence:0.91} ✅
Node 5: Has attachment? → TRUE ✅
Node 6: ExtractFromFile → resume_raw_text = "" (empty, no text layer in scanned PDF)
Node 6b: text.trim().length = 0 → text_valid = false
```

**Result:** ✅ Routes to `parse_failed` path (Node 6c).
HR notified. Row logged with `status=parse_failed`. **PASS.**

> [!TIP]
> Future upgrade: Add an OCR step (e.g. Google Cloud Vision API or Mindee)
> between Node 6 and Node 6b to handle image-based PDFs before declaring parse_failed.

---

### ❌ EDGE CASE 4 — Applied role exists in config but with different casing

**Input:**
```
From: ananya@outlook.com
Subject: Job application - software engineer
AI parsed applied_role from Node 2: "software engineer" (all lowercase)
OPENINGS config key: "Software Engineer" (Title Case)
```

**Step-by-step trace:**
```
Node 9 — Role Config Check:
  const role = "software engineer"
  const slots = OPENINGS["software engineer"] → undefined
  roleExists = false → roleOpen = false → role_not_open path triggered ❌
```

**Result:** ❌ FAIL. A valid role rejected because of casing mismatch.
Ananya gets "no openings" email even though Software Engineer has 2 open slots.

**Fix Applied to NODE 9:**
```javascript
// Normalize role lookup — case insensitive match
const OPENINGS = {
  "software engineer": 2,
  "hr executive": 1,
  "marketing lead": 1,
  "data analyst": 0,
};

const role = ($json.applied_role || '').toLowerCase().trim();
const slots = OPENINGS[role];
const roleExists = slots !== undefined;
const roleOpen = roleExists && slots > 0;

// Store normalized role for downstream nodes
return [{ json: { ...$json, applied_role_normalized: role, role_slots: slots, role_exists: roleExists, role_open: roleOpen } }];
```
> All downstream nodes now use `applied_role_normalized` as the canonical role value.

**Re-run Edge Case 4:**
```
role = "software engineer" (normalized)
OPENINGS["software engineer"] = 2
roleOpen = true ✅ → continues to Node 10
```
**PASS after fix. ✅**

---

### ❌ EDGE CASE 5 — Spam / phishing email that partially looks like a job application

**Input:**
```
From: offers@urgentjobs.biz
Subject: I am applying for your job posting (URGENT)
Body: "Dear HR, I saw your job posting online. I want to apply. Please find my CV attached.
Also, please click this link to verify my credentials: http://malicious-site.xyz/free-cv"
Attachments: invoice.pdf (not a resume — it's a phishing PDF)
```

**Step-by-step trace (V1 logic — pre-fix):**
```
Node 2: AI confidence=0.88 (above threshold), is_job_application=true ❌ (confident but wrong)
Node 5: Has attachment? → invoice.pdf present → TRUE ✅
Node 6: Extract text from invoice.pdf → extracted text of invoice (~300 chars, passes length check) ✅
Node 7: AI parses "resume" → name:null, email:null, applied_role:"unknown", skills:[] 
Node 8: required fields check → name=null → parse_valid=false ❌
```

**V1 Behaviour:** Node 8 parse_valid = false — but there is no handler for this case! V1 logic has `no_resume`, `parse_failed`, `duplicate`, `role_not_open` but nothing for "resume parsed but required fields are null."

**Result:** ❌ FAIL. Workflow reaches a dead end — nothing is logged, no email sent, no error handled. Silent failure.

**Fix — Add NODE 8a: Required Fields Check (IF node)**
```
NODE 8 output: parse_valid = true/false
    ├─ TRUE  → continue to Duplicate Check (Node 8b)
    └─ FALSE → [NEW] status=parse_failed (re-use Node 6c path)
               Log row: status=parse_failed, reason="required_fields_missing"
               HR alert: "Application from [sender] could not be parsed. Manual review."
               END
```

Also add a file-type guard at **NODE 5** to prevent non-resume file types:
```javascript
// In Node 5 (Attachment Check code node)
const ALLOWED_MIME_TYPES = [
  'application/pdf',
  'application/msword',
  'application/vnd.openxmlformats-officedocument.wordprocessingml.document'
];

const validAttachments = attachments.filter(a => ALLOWED_MIME_TYPES.includes(a.mimeType));
const hasValidAttachment = validAttachments.length > 0;

return [{ json: { ...items[0].json, attachments: validAttachments, has_valid_attachment: hasValidAttachment } }];
```
- If `has_valid_attachment = false` → treat as `no_resume`.

**Re-run Edge Case 5:**
```
Node 5: invoice.pdf mimeType = "application/pdf" → passes type filter ✅ (PDF can't be filtered by name)
Node 6: text extracted from invoice (~300 chars) → text_valid = true ✅
Node 7: AI parse → name=null, email=null ❌
Node 8: parse_valid = false → [NEW Node 8a] → parse_failed path ✅
HR notified. Logged with status=parse_failed.
```
**PASS after fix. ✅**

> [!NOTE]
> This is a best-effort defense. A perfectly crafted phishing PDF that looks like a resume
> could pass all checks. Final recommendation: HR should always manually confirm before
> activating an offer letter by filling in `salary`, `start_date`, and `reporting_manager`
> in the Employees Sheet. Those fields being blank is the natural final gate.

---

## 5. POST DRY-RUN LOGIC UPDATES SUMMARY

The following changes were made to V1 logic after dry-run failures:

| Issue Found | Edge Case | Fix Applied |
|---|---|---|
| Role lookup was case-sensitive | EC4 | Normalize `applied_role` to lowercase before OPENINGS lookup |
| No handler for "parsed but fields missing" | EC5 | Added Node 8a — `parse_valid` IF gate → routes to `parse_failed` path |
| Wrong file types accepted as resume | EC5 | Added MIME type whitelist filter in Node 5 attachment check |

---

## 6. FINAL CORRECTED NODE LIST (V2)

```
01  IMAP Email Trigger
02  AI Agent: Classify Email
02b IF: Confidence Gate (≥0.85 + is_application?)
    ├─ not_application → 03  Auto-Reply → END
    └─ low_confidence  → 03b Log low_confidence + HR alert → END
04  Extract Attachment (binary data)
05  Code: Attachment Type Filter + Existence Check
    └─ no valid attachment → 05b Log no_resume + "Please resend" email → END
06  ExtractFromFile (PDF/DOCX → text)
06b Code: Validate text length (≥100 chars)
    └─ text_valid=false → 06c Log parse_failed + HR alert → END
07  AI Agent: Parse Resume → JSON
08  Code: Validate Required Fields (name, email, applied_role)
08a IF: parse_valid?
    └─ false → Log parse_failed + HR alert → END          ← NEW (EC5 fix)
08b Google Sheets: Duplicate Check (email + role)
    └─ duplicate → 08c Log duplicate + "Already received" email → END
09  Code: Role Config Check (case-insensitive)             ← UPDATED (EC4 fix)
    └─ role_not_open → 09b Log role_not_open + "No openings" email → END
10  Google Sheets: Append to Applications Sheet (status=pending)
11  HTTP Request: Submit to Google Form (onError: continue)
12  Google Sheets: Read + Code: Count selected for role
13  IF: canSelect?
    ├─ TRUE  → 14a Update status=selected + Add to Employees Sheet
    └─ FALSE → 14b Update status=waitlisted + Acknowledgement email
```

**Total nodes: 22 (including branches)**

---

## 7. APPLICATIONS SHEET COLUMN SCHEMA

| Column | Type | Notes |
|---|---|---|
| `name` | string | Parsed from resume |
| `email` | string | Parsed from resume |
| `phone` | string | Parsed from resume |
| `applied_role` | string | Normalized lowercase from classifier |
| `experience_years` | number | Parsed from resume |
| `current_company` | string | Parsed from resume |
| `education` | string | Parsed from resume |
| `skills` | string | Comma-joined array |
| `resume_summary` | string | 2-sentence AI summary |
| `applied_at` | ISO datetime | System timestamp |
| `source_email` | string | Original sender address |
| `status` | string | See State Table |
| `ats_score` | number | Blank — reserved for future |
| `selected_at` | ISO datetime | Populated when selected |
| `waitlisted_at` | ISO datetime | Populated when waitlisted |
| `duplicate_count` | number | Times same person re-applied |
| `hr_review` | boolean | TRUE = flagged for manual review |
| `error_reason` | string | Populated on parse_failed cases |

---

*Document version: V2 — Updated after 5-edge-case dry run on 2026-03-06*
