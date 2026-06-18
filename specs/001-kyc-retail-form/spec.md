# Feature Specification: KYC Retail Application Form

**Feature Branch**: `001-kyc-retail-form`

**Created**: 2026-06-17

**Status**: Draft

**Input**: User description: "1) create a form with maximum 5 important input fields for retail
user for KYC validation in a bank 2) A user is allowed to upload only driver's license and US
passport for the application to be KYC approved."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Submit Personal Information (Priority: P1)

A retail bank customer fills out the KYC application form by providing their personal identity
information across 4 required fields — Full Legal Name, Date of Birth, Social Security Number
(SSN), and Home Address — plus 1 optional field: US Phone Number. The 4 required fields are
mandatory; the customer cannot advance without completing them with valid data. The US Phone
Number field is optional but, if provided, must be a valid 10-digit US phone number.

**Why this priority**: Personal information is the foundational requirement for KYC. It
satisfies the US Bank Secrecy Act Customer Identification Program (CIP) minimum: name, date
of birth, address, and identification number. Without it, no identity verification can proceed.

**Independent Test**: Can be fully tested by loading the KYC form, entering data in all 4
required fields (and optionally the US Phone Number), and verifying the form accepts valid
input and rejects invalid input — entirely independent of the document upload step.

**Acceptance Scenarios**:

1. **Given** an authenticated retail customer is on the KYC form page, **When** they enter
   valid data in all 4 required fields and proceed, **Then** the system accepts the input and
   advances them to the document upload step (regardless of whether Phone Number was provided).
2. **Given** a customer has left one or more required fields empty, **When** they attempt to
   proceed, **Then** the system displays field-level validation errors and prevents advancement.
3. **Given** a customer enters an SSN that does not match the format XXX-XX-XXXX, **When**
   they attempt to proceed, **Then** the system highlights the SSN field with a descriptive
   format error.
4. **Given** a customer enters a date of birth that indicates they are under 18 years of age,
   **When** they attempt to proceed, **Then** the system rejects the entry with a clear
   eligibility message stating the minimum age requirement.
5. **Given** a customer enters a US Phone Number that does not conform to the 10-digit NANP
   format, **When** they attempt to proceed, **Then** the system highlights the Phone Number
   field with a format error; an empty Phone Number field MUST NOT block advancement.
6. **Given** an authenticated customer who already has an active (pending or under-review) KYC
   application attempts to access the KYC form, **When** the page loads, **Then** the system
   blocks entry to the form and displays a message showing the existing reference number and
   current application status — no new submission form is presented.
7. **Given** an authenticated customer who previously left the form mid-completion returns to
   the KYC form on a subsequent login, **When** the page loads, **Then** the system detects
   the saved draft and presents two options: resume the saved draft (with fields pre-populated)
   or discard the draft and start a new application.
8. **Given** a customer chooses to resume their saved draft, **When** the form loads, **Then**
   all previously entered field values are restored exactly as the customer last entered them.

---

### User Story 2 - Upload Identity Document (Priority: P2)

After providing personal information, the customer selects their document type from a
pre-populated selector (US Driver's License or US Passport) and then uploads the
corresponding file. Because only valid document types are selectable, invalid type selection
is prevented by design. The customer cannot receive KYC approval without a successfully
uploaded accepted document.

**Why this priority**: Document upload is the second mandatory step for KYC approval — both
personal information and a valid government ID are required together. Personal information
alone is insufficient for bank KYC approval.

**Independent Test**: Can be tested by presenting the document upload component in isolation:
select Driver's License, upload valid file → accepted; select Passport, upload valid file →
accepted; attempt invalid file format or oversized file → rejected with clear error.

**Acceptance Scenarios**:

1. **Given** a customer is on the document upload step, **When** they view the document type
   selector, **Then** exactly two options are displayed: "US Driver's License" and
   "US Passport" — no other options are available.
2. **Given** a customer selects "US Driver's License" and uploads a JPEG, PNG, or PDF file
   within the size limit, **When** they submit, **Then** the system accepts the file and
   displays a success confirmation with the document type label and filename.
3. **Given** a customer selects "US Passport" and uploads a JPEG, PNG, or PDF file within
   the size limit, **When** they submit, **Then** the system accepts the file and displays a
   success confirmation.
4. **Given** a customer uploads a file that exceeds 5 MB, **When** they attempt to upload,
   **Then** the system rejects the file and displays an error stating the 5 MB size limit.
5. **Given** a customer uploads a file in an unsupported format (e.g., .docx, .xls, .heic),
   **When** they attempt to upload, **Then** the system rejects the file and lists the
   accepted formats (JPEG, PNG, PDF).
6. **Given** the document upload service or S3 is temporarily unavailable, **When** a
   customer attempts to upload a file, **Then** the system silently retries the upload up to
   3 times; if all retries fail, the system displays an inline error message and a "Try
   Again" button without losing any previously entered personal information.

---

### User Story 3 - Confirm KYC Application Submission (Priority: P3)

After completing both the personal information form and document upload, the customer submits
the complete KYC application and receives a confirmation that their application has been
received and is under review.

**Why this priority**: Confirmation provides the customer with closure, sets expectations for
the review timeline, and gives them a reference number to track or query their application.

**Independent Test**: Can be tested by mocking a completed application state and verifying the
confirmation screen renders correctly with a reference number and appropriate messaging.

**Acceptance Scenarios**:

1. **Given** a customer has completed all required personal information fields and uploaded a
   valid document, **When** they submit the application, **Then** the system displays a
   confirmation screen with a unique application reference number and an estimated review
   timeline.
2. **Given** a customer successfully submits a KYC application, **When** submission completes,
   **Then** the customer receives a confirmation email to the email address the bank has on
   file. SMS notification is out of scope for this feature.
3. **Given** a system error occurs during submission, **When** the customer attempts to
   submit, **Then** the system displays a user-friendly error message with a visible "Try
   Again" button and preserves all customer-entered form data so the customer does not need
   to re-enter information and can re-attempt submission without navigating away.

---

### Edge Cases

- What happens when a customer uploads a driver's license image that is blurry or unreadable?
- When a customer closes the browser mid-form, the system saves their progress as a server-side
  draft. On their next login, they are offered the option to resume the draft or start over
  (see FR-016). Draft data is not considered an active application and does not trigger
  the duplicate-application block (FR-015).
- If the same SSN is submitted under two different customer accounts: system MUST reject the
  second submission at submission time, returning a generic error without exposing the
  conflicting customer's identity (see FR-017). The check is SSN-hash-based and applies only
  to active (pending/under_review) applications, not to drafts or rejected applications.
- When a customer with an existing pending or under-review KYC application attempts to access
  the form, the system blocks entry and presents the existing reference number and status
  (see FR-015). Re-application after rejection is out of scope.
- If the document upload service or S3 is temporarily unavailable: system silently retries
  up to 3 times, then displays an inline error with a "Try Again" button; all previously
  entered personal information is preserved (see US2 AC-6).

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST present a KYC application form with 4 required input fields — Full
  Legal Name, Date of Birth, Social Security Number (SSN), and Home Address — and 1 optional
  input field: US Phone Number. Total form fields: 5.
- **FR-002**: System MUST validate all 4 required fields as complete and correctly formatted
  before allowing the customer to proceed to the document upload step.
- **FR-003**: System MUST validate that the applicant's date of birth indicates they are at
  least 18 years of age at the time of submission.
- **FR-004**: System MUST validate SSN format (9-digit pattern: XXX-XX-XXXX).
- **FR-005**: System MUST display clear, field-level validation error messages adjacent to
  each invalid or empty required field.
- **FR-006**: System MUST allow customers to upload exactly one identity document per
  KYC application.
- **FR-007**: System MUST present a document type selector offering exactly two options:
  "US Driver's License" and "US Passport". No other document type option MUST be available.
  The customer MUST select a document type before the file upload control is enabled.
- **FR-008**: System MUST record the customer-declared document type (from FR-007 selector)
  alongside the uploaded file. The declared type is set exclusively via the selector and
  cannot be left unselected before upload proceeds.
- **FR-009**: System MUST enforce a maximum file size of 5 MB on uploaded documents. Files
  exceeding 5 MB MUST be rejected with an error message stating the 5 MB limit.
- **FR-010**: System MUST accept uploaded documents in JPEG, PNG, and PDF formats only;
  all other file formats MUST be rejected.
- **FR-011**: System MUST generate a unique application reference number upon successful
  application submission.
- **FR-012**: System MUST display a confirmation screen containing the reference number and
  an estimated review timeline of 3 business days after successful submission.
- **FR-013**: System MUST preserve all customer-entered form data if a submission error
  occurs, so the customer does not need to re-enter information. The system MUST display
  a visible "Try Again" button inline with the error message so the customer can re-attempt
  submission without navigating away from the form.
- **FR-014**: If the customer provides a US Phone Number, the system MUST validate it as a
  valid 10-digit US phone number in NANP format (e.g., XXX-XXX-XXXX). An empty US Phone
  Number field MUST NOT prevent form advancement.
- **FR-015**: System MUST detect when an authenticated customer already has an active KYC
  application in pending or under-review status. When detected, the system MUST block access
  to the new application form and display the existing application's reference number and
  current status. The customer MUST NOT be able to submit a new application while one is
  active. Draft-status applications MUST NOT trigger this block.
- **FR-016**: System MUST auto-save the customer's form progress as a server-side draft
  whenever they navigate away from or close the KYC form before submission. On the
  customer's next authenticated visit to the KYC form, the system MUST detect the saved
  draft and offer two options: (a) resume — restore all previously entered field values
  exactly as saved, or (b) discard — clear the draft and present a fresh empty form.
  A draft application MUST NOT be treated as a submitted application for any purpose.
- **FR-017**: At application submission time, the system MUST compute the SHA-256 hash of
  the submitted SSN and check it against the `ssn_hash` values of all active applications
  (status: pending or under_review) belonging to different customer accounts. If a match is
  found, the system MUST reject the submission with a clear error — "An application using
  this identity is already under review" — without revealing the conflicting customer's
  identity or reference number. The submitted application reverts to draft status.

### Key Entities

- **KYC Application**: Represents a single customer's identity verification request. Links
  the 5 personal information fields (4 required, 1 optional) to the uploaded identity
  document and tracks the application through its lifecycle:
  draft → pending → under-review → approved | rejected.
  Only one non-draft application may be active per customer at any time.
- **Identity Document**: The file uploaded by the customer. Carries a customer-declared
  document type (one of: US Driver's License, US Passport — selected before upload), file
  format, file size, and upload status.
- **Applicant**: The authenticated retail bank customer submitting the KYC form. Identified
  by the personal information entered across the 4 required fields.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A customer with valid information can complete the full KYC application — all
  required fields plus document upload — in under 5 minutes from form load to confirmation
  screen.
- **SC-002**: 100% of uploaded documents that are not a US driver's license or US passport
  are rejected before the application reaches the review queue.
- **SC-003**: All 4 required fields are validated before submission, preventing any incomplete
  or incorrectly formatted personal information from reaching the review system.
- **SC-004**: Customers receive a confirmation screen with a unique reference number within
  3 seconds of successful application submission under normal load conditions.
- **SC-005**: 95% of customers who begin the KYC form successfully complete and submit the
  application on their first attempt without requiring external support.
- **SC-006**: Zero loss of customer-entered form data occurs in the event of a submission
  error — all entered data persists and remains editable after an error.

## Assumptions

- The KYC form is accessed only by authenticated retail bank customers — the customer is
  logged into their bank account before the KYC flow begins; authentication is out of scope.
- Full Legal Name is captured as a single input field (not split into First Name / Last Name
  separately) to stay within the 5-field maximum.
- Home Address is captured as a single structured input; multi-step address wizards are out
  of scope.
- SSN format validation is performed by the form (pattern: XXX-XX-XXXX); real-time
  verification against external government databases is out of scope for this form.
- Document authenticity verification (e.g., fraud detection, liveness checks, OCR) is
  handled by a downstream review process, not by the form itself.
- Only one active KYC application per customer is permitted at a time; re-application and
  amendment flows are out of scope.
- The form must be usable on both desktop browsers and mobile smartphone browsers.
- Notification upon submission is email only, sent to the email address already on file
  with the bank. The form does not collect a new email address. SMS notification is
  explicitly out of scope for this feature.
- "US driver's license" refers to a driver's license issued by any US state or territory.
- US Phone Number, when provided, is stored as entered and used for potential future contact
  by the bank; it does not replace the existing contact on file.
- Draft KYC applications (auto-saved mid-form) store partial PII server-side before
  submission. Data retention policy and expiry rules for abandoned drafts are out of scope
  for this feature and will be determined by a separate data governance process.

## Clarifications

### Session 2026-06-17

- Q: Should US phone number be included in the KYC form as a required or optional field?
  → A: Optional. US Phone Number is an optional 5th field; the 4 CIP-required fields
  (Full Legal Name, Date of Birth, SSN, Home Address) remain mandatory. When provided,
  the phone number must conform to 10-digit NANP format.
- Q: How does the customer declare which document type they are uploading (driver's license
  vs. US passport)?
  → A: Option A — customer selects document type from a pre-populated selector (US Driver's
  License / US Passport) before the file upload control is enabled. Only these two options
  are presented; invalid type selection is prevented by design.
- Q: What happens when a customer who already has an active (pending/under-review) KYC
  application attempts to submit a new one?
  → A: Option A — block and inform. System detects the active application, blocks access to
  the form, and displays the existing reference number and current status (FR-015).
- Q: What is the maximum permitted file size for uploaded identity documents?
  → A: 5 MB. Files exceeding 5 MB are rejected with a clear error message stating the limit
  (FR-009).
- Q: How does the system handle a partially completed session (customer closes browser
  mid-form)?
  → A: Option C — server-side draft. System auto-saves progress; on next authenticated visit
  the customer is offered resume or discard. Draft status is distinct from active/pending
  application status and does not trigger the duplicate-application block (FR-016).
- Q: What if the same SSN is submitted in two different KYC applications simultaneously
  (different customer accounts)?
  → A: Option B — in scope. System checks ssn_hash at submission time against all active
  (pending/under_review) applications across all customer accounts. Duplicate SSN triggers
  rejection with a generic error; the conflicting customer's identity is never revealed
  (FR-017).
- Q: What is the estimated review timeline shown on the confirmation screen (FR-012)?
  → A: 3 business days.
- Q: What notification channel is used for submission confirmation (US3 AC-2)?
  → A: Email only — sent to the email address the bank has on file. SMS is out of scope
  for this feature.
- Q: How does the system respond if the document upload service is temporarily unavailable?
  → A: Option C — silent retry up to 3 times; if all fail, show an inline error message
  and "Try Again" button; all personal information fields are preserved (US2 AC-6).
- Q: Does FR-013 require a visible retry action after submission error, or data persistence
  only?
  → A: Option B — data persistence plus a visible "Try Again" button displayed inline with
  the error message; submit button re-enabled so customer can re-attempt without navigating
  away (FR-013, US3 AC-3).
