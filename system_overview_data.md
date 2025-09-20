🚀 Migrant Health Records System Overview (Non‑Technical)

Focus: Real users, journeys, and outcomes.
Audience: Judges, program managers, health officials.
Scope: Frontend UX, user flows, admin oversight, safety, and interoperability.

👥 Personas & Roles

Migrant (Patient) – Your health, your control

Doctor (Provider) – Consultation, follow-up, care management

Hospital & Lab – Diagnostics and medical support

Admins

Super Admin (Full Control)

State Admin → District → Subdivision → Block Admin

🌐 High-Level System Context
flowchart LR
  User[(People: Migrants, Doctors, Labs, Admins)] -->|Web App| FE[HealthLink Frontend]
  FE -->|Secure APIs| BE[HealthLink Backend]
  BE --> DB[(Health Records Database)]
  BE --> S3[(Secure File Storage)]
  BE --> Email[Email/OTP]
  FE -->|Public QR| PublicProfile[Public Profile View]

🔑 Registration & Login (OTP Supported)

Secure Sign-Up: Email/password for all roles

OTP Login: Quick, safe, reliable

Password Reset: Built-in protection against abuse

sequenceDiagram
  participant U as User
  participant FE as Web App
  participant API as Auth API
  participant SMTP as Email

  U->>FE: Enter email (request OTP)
  FE->>API: Request OTP
  API->>SMTP: Send OTP email
  U->>FE: Enter OTP
  FE->>API: Verify OTP
  API-->>FE: Logged in (session)

🏛 Admin Hierarchy (Super → Block)

Super Admin: Ultimate control, can manage the entire hierarchy

Scoped Admins: State, District, Subdivision, Block

Full Control vs Limited:

Full Control → Invite/manage next level

Limited → View & operate only within scope

flowchart TD
  SA[Super Admin] -->|Invite/Assign| STA[State Admin]
  STA -->|Invite/Assign (Full Control)| DA[District Admin]
  DA -->|Invite/Assign (Full Control)| SDA[Subdivision Admin]
  SDA -->|Invite/Assign (Full Control)| BA[Block Admin]

  classDef fullC fill:#d1fae5,stroke:#10b981,color:#065f46
  STA:::fullC
  DA:::fullC
  SDA:::fullC
  BA:::fullC

🏗 Creating New Jurisdiction Units
sequenceDiagram
  participant A as Scoped Admin
  participant FE as Admin UI
  participant API as Admin GEO API
  participant DB as Database

  A->>FE: Create new unit (e.g., District)
  FE->>API: Submit details with parent scope
  API->>DB: Create/validate hierarchy
  DB-->>API: Success
  API-->>FE: New unit appears in directories

✉️ Invitations & Admin Sign-Up
flowchart LR
  Inviter[Admin with Full Control] -->|Generate Invitation| InviteCode[Invitation Code]
  InviteCode -->|Share Securely| Invitee[New Admin]
  Invitee -->|Complete Registration| AdminPortal[Admin Register]

🆔 Migrant Registration & QR/ID Card

Register & Manage Profile

Download Printable ID Card (CR80 size)

QR Code: Quick access to public profile

flowchart TD
  Migrant -->|Register| Profile[My Profile]
  Profile -->|Download| IDCard[Printable ID Card]
  Profile -->|QR| PublicView[Public Profile Page]

🩺 Doctor–Patient Collaboration

Search & view patients (permissions applied)

Create Consultation Reports (summary + notes)

Upload supporting files (prescriptions, scans)

Request lab reports & manage allergies, medications, vaccinations

sequenceDiagram
  participant D as Doctor
  participant FE as Clinician UI
  participant API as Patients/Reports API
  participant S3 as Storage

  D->>FE: Open patient
  D->>FE: Create Consultation Report (summary + notes)
  FE->>API: Save report
  D->>FE: Upload supporting file (e.g., prescription, scan)
  FE->>API: Presign upload
  FE->>S3: Upload file
  FE->>API: Mark available

✅ Consent (Patient-Centric)

Providers/hospitals need patient consent to view or update records

One-time code sent to patient

Patients can revoke consent at any time

flowchart TD
  P[Provider/Hospital] -->|Request Consent| System
  System -->|Send Code via Email| Migrant
  Migrant -->|Shares Code| P
  P -->|Enter Code| System
  System -->|Grant Time-Bound Consent| P
  Migrant -->|Revoke| System

⚡ Emergency Protocol

Time-limited read-only access in urgent situations

Fully audited for safety and transparency

sequenceDiagram
  participant P as Provider/Hospital
  participant API as Emergency Access API

  P->>API: Request emergency access (reason)
  API-->>P: Time-limited read-only grant
  Note over P,API: All emergency access is fully audited

🧪 Lab Workflow

Doctors request lab reports; optional lab assignment

Labs manage jobs: accept, deny with reason, upload reports

CSV export for offline analysis

flowchart LR
  Doctor -->|Request Report| System
  System -->|Assign (optional)| Lab
  Lab -->|Accept/Claim| Worklist
  Lab -->|Upload Report| Storage
  System -->|Mark Completed| Doctor & Migrant
  Lab -->|CSV Export| Local

🤖 Migrant Chatbot (Personal Health Assistant)

Ask plain-language questions

Summarizes personal records & reports

Multi-language: Indian languages + English

Disclaimers: Informative only, not medical advice

flowchart TD
  Migrant -->|Question| ChatUI[Chat]
  ChatUI --> Context[Personal Summary + Recent Report Snippets]
  Context --> Answer[Helpful, safe explanation]
  Answer --> Migrant

🔗 FHIR-Compatible (Read-Only Interoperability)
flowchart LR
  ExternalApp[External Health App] -- FHIR (read-only) --> API[/FHIR API/]

📜 Audit Logs

Logs every important action: reports, uploads, consent changes, emergency grants

Patients can see who did what and when

flowchart TD
  Actions --> Audit[Audit Trail]
  Admin/Doctor -->|Review| Audit
  Audit -->|Print/Download| Records

📊 Admin Dashboard & Heatmap

KPIs: users, patients, reports, attachments, active consents

Time-series charts: growth trends

Choropleth heatmap: geography-aware insights

Quick actions & one-click PDF download

flowchart TD
  Admin --> Dashboard
  Dashboard --> KPIs
  Dashboard --> Charts
  Dashboard --> Heatmap
  Dashboard --> QuickActions
  Dashboard --> Download[Download PDF]

🔍 Admin Directories & Search

Browse Doctors, Hospitals, Labs

Powerful search & filters

Clickable profile cards

flowchart LR
  Admin --> Directories
  Directories -->|Search/Filter| Results
  Results -->|Open| ProfileCard[Profile]

🌍 Accessibility & Multilingual Support

Accessibility widget: readability toggles

Switch UI languages: English + Indian languages

flowchart LR
  User --> Accessibility[Accessibility Widget]
  User --> Language[Language Switcher]

🔐 Password Reset (Forgot Password)
sequenceDiagram
  participant U as User
  participant API as Password Reset
  participant SMTP as Email

  U->>API: Enter Health ID (request reset)
  API->>SMTP: Send OTP
  U->>API: Enter OTP + New Password
  API-->>U: Password updated

🔄 End-to-End Flow Example
sequenceDiagram
  participant M as Migrant
  participant D as Doctor
  participant L as Lab
  participant A as Admin

  M->>System: Registers, gets Health ID & QR
  D->>System: Requests consent from M
  M-->>System: Approves consent (code)
  D->>System: Creates Consultation Report, uploads files
  D->>System: Requests Lab Report (assign lab optional)
  L->>System: Accepts job, uploads result (CSV export available)
  M->>Chatbot: Asks a question about results
  Chatbot-->>M: Safe, plain-language explanation
  A->>System: Reviews KPIs & heatmap; downloads dashboard PDF

🎯 Feature Checklist by Persona

Migrant

Profile, ID card, QR to public profile

Chatbot (personal guidance)

Consent control (approve/revoke)

Multilingual UI & accessibility

Doctor / Hospital

Search/open patients, create consultation reports

Upload supporting files

Manage medications, allergies, vaccinations

Request lab reports & assign labs

View audit logs

Lab

Jobs board: assigned/accepted/denied

Accept, deny, upload reports

CSV export

Admins

Hierarchical management

Full Control → invite/manage next level

Directories, search, quick actions

Review jurisdiction changes

Dashboard with KPIs, charts, heatmap, PDF download

📝 Glossary

Consultation Report: Doctor’s visit summary

Attachment: Supporting document (scan, prescription)

Consent: Patient permission for access

Emergency Access: Time-limited read-only access

Public Profile: Limited QR-based profile

ID Card: Printable ID with QR (CR80 size)

🛡 Safety & Privacy Principles

Least privilege & consent-first access

Full audit trails for every action

Patient-friendly language with multi-language guidance

Explanations only, never medical advice
