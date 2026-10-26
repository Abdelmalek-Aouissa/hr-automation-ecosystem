# HR Automation Ecosystem — Implementation Guide

This document is the technical reconstruction reference for the **HR Automation Ecosystem** project.

It complements the main repository `README.md` by preserving the implementation details required to rebuild, validate, or revise the lab without turning the portfolio landing page into a configuration manual.

> **Environment note:** All users, candidate records, email addresses, and business data shown in this lab are synthetic test data. Secrets, passwords, access tokens, client secrets, webhook secrets, and private authentication material are intentionally excluded.

---

## 1. Environment Overview

The lab uses the following components:

| Component | Purpose |
|---|---|
| Microsoft 365 E5 | Core productivity and licensing environment |
| Microsoft Entra ID | User lifecycle and identity administration |
| Microsoft Teams | HR Operations collaboration and notifications |
| Microsoft Planner | Operational HR tasks |
| SharePoint | HR policy knowledge and recruiting records |
| Outlook | Shared HR mailbox and automated communications |
| Microsoft Graph | User lookup, licensing, and account-state operations |
| n8n Cloud | Workflow orchestration |
| MindStudio | HR Operations portal and forms |
| OpenAI model through n8n | Structured resume-information extraction |

The primary tenant used in the lab is:

```text
aiproservices.onmicrosoft.com
```

The principal functional workspace is:

```text
HR Operations
```

---

# 2. Microsoft 365 Foundation

## 2.1 Test Users

The lab includes representative HR and business roles:

- Helen Carter — HR Manager
- Rebecca Moore — Recruiter
- Daniel Brooks — Hiring Manager / Operations
- Sophia Turner — HR Coordinator
- James Wilson — Employee / Operations
- Alex Morgan — IT Support Specialist

Additional test users were created during workflow validation.

The identities are laboratory accounts and are not production employee records.

---

## 2.2 Microsoft Teams

Create a private Microsoft Teams team:

```text
HR Operations
```

Create the following channels:

```text
General
Recruiting
Onboarding
Employee Requests
Offboarding
```

Assign the HR test users to the team.

Reference:

![HR Operations Team](../screenshots/01-01-hr-operations-team.png)

![HR Operations Team Members](../screenshots/01-02-hr-operations-team-members.png)

---

## 2.3 Shared HR Mailbox

Create a shared mailbox:

```text
hr@aiproservices.onmicrosoft.com
```

Grant the appropriate HR test users permission to send through the shared mailbox.

In the lab, two HR users were granted **Send on Behalf** permission.

Reference:

![Shared HR Mailbox](../screenshots/01-03-shared-hr-mailbox-created.png)

---

## 2.4 Planner

Create a Planner plan:

```text
HR Operations Tasks
```

Create these buckets:

```text
Recruiting
Onboarding
Employee Requests
Offboarding
```

Reference:

![Planner Buckets](../screenshots/01-04-hr-operations-planner-buckets.png)

---

## 2.5 SharePoint

Use the SharePoint site associated with the **HR Operations** team.

The site supports:

- HR document storage
- the HR Policy Knowledge list
- the Recruiting Candidates list

Reference:

![SharePoint HR Operations](../screenshots/01-05-sharepoint-hr-operations-folders.png)

---

# 3. Microsoft and n8n Credentials

The project deliberately uses separate credentials for different Microsoft 365 operations.

## 3.1 Outlook Credential

Used for:

- shared HR mailbox sends
- manager notifications
- employee replies
- welcome emails

The shared mailbox was validated with a simple connection-test workflow before the main HR automations were built.

---

## 3.2 Microsoft Teams / Planner Credential

Use the existing working Microsoft Teams credential for:

- Planner task creation
- Teams channel messages

Do not replace this credential with the SharePoint generic Microsoft credential.

A recurring source of confusion during the lab was a Teams node being configured with the SharePoint credential. The result was a `403 Missing scope permissions` response when loading the Team list.

When this occurs, compare the credential used by the failing node with a known-good Teams or Planner node from HR-03 or HR-04.

---

## 3.3 SharePoint Generic OAuth Credential

The generic n8n credential appears as:

```text
Microsoft account
```

The associated Entra application was:

```text
n8n HR SharePoint Lab
```

Redirect URI:

```text
https://oauth.n8n.cloud/oauth2/callback
```

Final delegated scope configuration:

```text
openid offline_access Sites.ReadWrite.All
```

The Entra application was granted Microsoft Graph delegated permission:

```text
Sites.ReadWrite.All
```

Admin consent was granted.

This credential is used for SharePoint list read/write operations.

Do not extend this credential unnecessarily to Teams operations when an existing Teams credential already works.

---

## 3.4 Microsoft Graph / Entra Credential

Graph operations are used where the standard n8n nodes do not expose the required action directly.

The n8n credential must allow HTTP requests to:

```text
graph.microsoft.com
```

If the credential includes an **Allowed HTTP Request Domains** restriction, include:

```text
graph.microsoft.com
```

---

# 4. HR-01 — Microsoft 365 Connection Test

Workflow:

```text
Manual Trigger
    ↓
Outlook Send Message
```

Purpose:

- validate the Outlook connection
- validate shared-mailbox sending
- confirm the n8n Microsoft 365 connection before building larger workflows

This workflow can remain inactive after validation and serve as a diagnostic utility.

---

# 5. HR-02 — Employee Request Intake

## 5.1 Workflow Architecture

The common intake path is:

```text
Outlook Trigger
    ↓
Edit Fields
    ↓
Ignore HR Self-Messages
    ↓
Remove Duplicates
    ↓
Text Classifier
```

The classifier routes requests into:

```text
Leave / PTO
Benefits
Payroll / Compensation
Employment Documents
HR Policy / General Question
Recruiting
Onboarding
Offboarding
Other
```

Reference:

![HR-02 Workflow](../screenshots/02-07-hr02-complete-workflow.png)

---

## 5.2 Outlook Trigger

The workflow monitors the shared HR mailbox for incoming employee requests.

Normalize the message into consistent fields such as:

- sender name
- sender email
- subject
- body
- message ID

---

## 5.3 Self-Message Loop Protection

Automated HR replies can re-enter an Outlook polling trigger and cause a message loop.

Add an IF node:

```text
Ignore HR Self-Messages
```

Condition:

**Left — EXPRESSION**

```text
{{ $json.sender_email.toLowerCase() }}
```

**Operator**

```text
is not equal to
```

**Right — FIXED**

```text
hr@aiproservices.onmicrosoft.com
```

Only messages not sent by the HR mailbox continue.

---

## 5.4 Duplicate Protection

Use the incoming Microsoft message ID in a duplicate-removal node.

Purpose:

- prevent repeated processing
- protect against trigger polling behavior
- prevent duplicate tasks and replies

---

## 5.5 Text Classifier

Configure these categories:

```text
Leave / PTO
Benefits
Payroll / Compensation
Employment Documents
HR Policy / General Question
Recruiting
Onboarding
Offboarding
Other
```

Reference:

![Text Classifier](../screenshots/02-02-text-classifier-categories.png)

---

# 6. HR-02 Branches

## 6.1 Leave / PTO

The Leave/PTO branch:

1. receives the classified employee request
2. extracts relevant request information
3. creates an operational HR task
4. posts a Teams notification
5. preserves the original request context for HR follow-up

Reference:

![PTO Request](../screenshots/02-01-pto-request-email.png)

![Employee Request Planner](../screenshots/02-04-employee-request-planner-board.png)

![Employee Request Teams](../screenshots/02-05-employee-request-teams-notifications.png)

---

## 6.2 Employment Documents

Typical request:

```text
Employment verification
```

Reference:

![Employment Verification Request](../screenshots/02-06-employment-verification-request-email.png)

The branch separates employment-document requests from general HR questions so they can be handled as a dedicated operational activity.

---

## 6.3 Benefits

Benefits requests are routed to their own branch.

Reference:

![Benefits Request](../screenshots/02-08-benefits-request-email.png)

---

# 7. HR Policy / General Question Branch

## 7.1 SharePoint List

Create:

```text
HR Policy Knowledge
```

Columns:

| Column | Type |
|---|---|
| Title | Single line of text |
| Category | Single line of text |
| PolicyText | Multiple lines of text |
| EffectiveDate | Date |

Example records used in the lab:

- Remote Work
- PTO
- Health Benefits Enrollment

Reference:

![HR Policy Knowledge](../screenshots/02-09-hr-policy-knowledge-list.png)

---

## 7.2 Policy Branch Sequence

```text
Policy Question Extractor
    ↓
SharePoint Get many items
    ↓
Generate Policy Answer
    ↓
Outlook Reply
    ↓
Update Message Is Read=True
```

---

## 7.3 Information Extractor

The policy branch uses structured extraction to isolate the employee's actual question before retrieving policy content.

Reference:

![Information Extractor](../screenshots/02-03-information-extractor-configuration.png)

---

## 7.4 SharePoint Retrieval

Use the SharePoint generic Microsoft credential.

Ensure the SharePoint node returns the list item's `fields` object so the policy text and metadata are available downstream.

---

## 7.5 Grounding Rule

The policy answer must be based only on retrieved SharePoint policy records.

The design goal is:

```text
Employee question
    ↓
Question extraction
    ↓
SharePoint policy retrieval
    ↓
Answer grounded in retrieved policy
```

Do not use unrestricted generation as the policy source.

Reference:

![Remote Work Policy Request](../screenshots/02-10-remote-work-policy-request-email.png)

---

# 8. MindStudio HR Operations Portal

The MindStudio agent is:

```text
HR Operations Portal
```

Main flow:

```text
Main.flow
```

Start mode:

```text
On-Demand
```

Main menu options:

```text
Recruiting
Start Onboarding
Employee Request
Start Offboarding
Generate HR Document
```

The completed project actively implements the Recruiting, Onboarding, and Offboarding branches used in the final portal.

Reference:

![HR Operations Portal](../screenshots/00-01-final-hr-operations-portal.png)

---

## 8.1 MindStudio Flow-Building Notes

The MindStudio interface used in this lab has several UI behaviors worth preserving.

To create a disconnected branch safely:

```text
Right-click empty canvas
→ New Block
→ View All Blocks
→ User Input
```

Then route the menu item with:

```text
Select Destination
```

The right-side dot often means:

```text
Add Next Block
```

It should not be assumed to be a generic connector.

`On Cancel` is not the normal continuation route.

A Checkpoint block caused:

```text
Checkpoint has no transition
```

The Checkpoint was removed from the onboarding branch rather than retained as unnecessary complexity.

For testing:

```text
Preview
→ Open Draft Agent
```

The debugger may show `Waiting for Input`; the actual user-facing form appears through the Draft Agent.

---

# 9. HR-03 — New Hire Onboarding

## 9.1 MindStudio Form

Branch:

```text
Start Onboarding
    ↓
Onboarding Form
    ↓
Send New Hire to n8n
    ↓
End
```

Form variables:

```text
first_name
last_name
personal_email
job_title
department
manager_email
start_date
```

Reference:

![Onboarding Form](../screenshots/03-01-mindstudio-onboarding-form.png)

---

## 9.2 n8n Workflow

```text
Webhook
    ↓
Normalize New Hire
    ↓
Check Existing User
    ↓
User Does Not Exist?
```

TRUE branch:

```text
Generate Temporary Password
    ↓
Create Entra User
    ↓
Assign Office 365 E5
    ↓
Create Onboarding Task
    ↓
Post Onboarding Notification
    ↓
Notify Hiring Manager
    ↓
Send Welcome Email
    ↓
Onboarding Success Response
```

FALSE branch:

```text
Duplicate User Response
```

Reference:

![HR-03 Complete Workflow](../screenshots/03-06-hr03-complete-workflow.png)

---

## 9.3 Webhook

Method:

```text
POST
```

Path:

```text
hr-new-hire-onboarding
```

Authentication:

```text
None
```

Response behavior:

```text
Respond When Last Node Finishes
```

Response data:

```text
First Entry JSON
```

---

## 9.4 Normalize New Hire

A Code node is used because MindStudio can send the payload in more than one shape.

```javascript
const body = $json.body ?? {};
let data = body;

if (typeof body.first_name === 'undefined') {
  const key = Object.keys(body)[0];
  const raw =
    key !== undefined &&
    typeof body[key] === 'string' &&
    body[key].trim()
      ? body[key]
      : key;

  data = JSON.parse(raw);
}

return [
  {
    json: {
      first_name: data.first_name ?? null,
      last_name: data.last_name ?? null,
      personal_email: (data.personal_email ?? '')
        .replace(/^Email:\s*/i, ''),
      job_title: data.job_title ?? null,
      department: data.department ?? null,
      manager_email: data.manager_email ?? null,
      start_date: data.start_date ?? null,
      source: 'MindStudio'
    }
  }
];
```

---

## 9.5 User Principal Name

Use:

```text
{{ first_name.toLowerCase() + '.' + last_name.toLowerCase() + '@aiproservices.onmicrosoft.com' }}
```

---

## 9.6 Existing User Check

Graph endpoint:

```text
GET https://graph.microsoft.com/v1.0/users
```

Query parameter:

**`$filter` — EXPRESSION**

```text
{{ "userPrincipalName eq '" + first_name.toLowerCase() + "." + last_name.toLowerCase() + "@aiproservices.onmicrosoft.com'" }}
```

**`$select` — FIXED**

```text
id,displayName,userPrincipalName
```

---

## 9.7 User Does Not Exist?

IF condition:

**Left — EXPRESSION**

```text
{{ $json.value.length }}
```

Type:

```text
Number
```

Operator:

```text
is equal to
```

Right:

```text
0
```

A previous configuration used the String type and produced incorrect behavior. The length comparison must be numeric.

Reference:

![Duplicate User Protection](../screenshots/03-05-duplicate-user-protection-branch.png)

---

## 9.8 Duplicate Response

Fields:

**status — FIXED**

```text
duplicate
```

**message — FIXED**

```text
User already exists — onboarding stopped safely.
```

**user_principal_name — EXPRESSION**

Use the calculated UPN.

Keep only the set response fields.

---

## 9.9 Temporary Password

Use a Crypto node:

```text
Generate Temporary Password
```

Generate:

```text
HEX
```

Output field:

```text
temp_password_seed
```

Length:

```text
20
```

Password expression:

```text
{{ 'Aa1!' + $('Generate Temporary Password').first().json.temp_password_seed }}
```

Enforce:

```text
Force password change at next sign-in = True
```

Do not expose generated temporary passwords in repository screenshots or documentation.

---

## 9.10 Entra User Creation

Create the user through the Microsoft Entra ID node.

Use:

- generated UPN
- display name from first/last name
- temporary password
- force password change
- UsageLocation = `US` when available

Reference:

![New Hire Entra User](../screenshots/03-02-new-hire-entra-user-with-license.png)

---

## 9.11 Office 365 E5 License

SKU:

```text
ENTERPRISEPREMIUM
```

SKU ID:

```text
c7df2760-2c81-4ef7-b578-5b5392b571df
```

Graph endpoint:

```text
POST /users/{id}/assignLicense
```

Request body:

```json
{
  "addLicenses": [
    {
      "skuId": "c7df2760-2c81-4ef7-b578-5b5392b571df"
    }
  ],
  "removeLicenses": []
}
```

---

## 9.12 Planner Task

Create a task in:

```text
Plan: HR Operations Tasks
Bucket: Onboarding
```

Use the known-good Teams/Planner credential.

---

## 9.13 Teams Notification

Post to:

```text
Team: HR Operations
Channel: Onboarding
```

---

## 9.14 Hiring Manager Notification

Send through Outlook to the manager email supplied in the form.

Reference:

![Hiring Manager Notification](../screenshots/03-08-hiring-manager-notification-email.png)

---

## 9.15 Welcome Email

Send to the new hire's personal email.

Do not include the generated temporary password in the GitHub documentation.

Reference:

![Welcome Email](../screenshots/03-07-new-hire-welcome-email.png)

---

## 9.16 Validation

Confirm:

- Entra user exists
- account is enabled
- Office 365 E5 is assigned
- Planner onboarding task exists
- Teams onboarding message exists
- manager notification was delivered
- welcome email was delivered
- success response was returned

Reference:

![Successful HR-03 Execution](../screenshots/03-03-hr03-successful-execution.png)

---

# 10. HR-04 — Employee Offboarding

## 10.1 MindStudio Form

Branch:

```text
Start Offboarding
    ↓
Offboarding Form
    ↓
Send Offboarding to n8n
    ↓
Offboarding End
```

Form variables:

```text
employee_name
user_principal_name
manager_email
last_working_date
reason
notes
```

Reference:

![Offboarding Flow](../screenshots/04-01-mindstudio-offboarding-flow.png)

---

## 10.2 n8n Workflow

```text
Webhook
    ↓
Normalize Offboarding Request
    ↓
Find Employee in Entra
    ↓
Employee Exists?
```

TRUE:

```text
Disable Entra Account
    ↓
Remove Office 365 E5
    ↓
Create Offboarding Task
    ↓
Post Offboarding Notification
    ↓
Notify Manager
    ↓
Offboarding Success Response
```

FALSE:

```text
Employee Not Found Response
```

Reference:

![HR-04 Complete Workflow](../screenshots/04-04-hr04-complete-successful-workflow.png)

---

## 10.3 Webhook

Method:

```text
POST
```

Path:

```text
hr-employee-offboarding
```

---

## 10.4 Normalize Offboarding Request

```javascript
const body = $json.body ?? {};
let data = body;

if (typeof body.user_principal_name === 'undefined') {
  const key = Object.keys(body)[0];

  if (key === undefined) {
    throw new Error('No offboarding data received');
  }

  const raw =
    typeof body[key] === 'string' && body[key].trim()
      ? body[key]
      : key;

  data = JSON.parse(raw);
}

return [
  {
    json: {
      employee_name: data.employee_name ?? '',
      user_principal_name: (data.user_principal_name ?? '')
        .replace(/^Email:\s*/i, '')
        .trim()
        .toLowerCase(),
      manager_email: (data.manager_email ?? '')
        .replace(/^Email:\s*/i, '')
        .trim()
        .toLowerCase(),
      last_working_date: data.last_working_date ?? '',
      reason: data.reason ?? '',
      notes: data.notes ?? '',
      source: 'MindStudio'
    }
  }
];
```

---

## 10.5 Employee Lookup

Graph endpoint:

```text
GET https://graph.microsoft.com/v1.0/users
```

Filter on:

```text
userPrincipalName
```

Select:

```text
id,displayName,userPrincipalName,accountEnabled
```

---

## 10.6 Employee Exists?

Condition:

**Left — EXPRESSION**

```text
{{ $json.value.length }}
```

Type:

```text
Number
```

Operator:

```text
is greater than
```

Right:

```text
0
```

---

## 10.7 Employee Not Found

Return:

**status — FIXED**

```text
not_found
```

**message — FIXED**

```text
Employee was not found in Microsoft Entra ID. Offboarding stopped safely.
```

**user_principal_name — EXPRESSION**

Use the normalized UPN.

Reference:

![Employee Not Found](../screenshots/04-02-employee-not-found-branch.png)

---

## 10.8 Disable Entra Account

Graph request:

```text
PATCH /users/{id}
```

JSON body:

```json
{
  "accountEnabled": false
}
```

Graph returns a successful `204 No Content`.

Enable the n8n node setting:

```text
Always Output Data = ON
```

This allows the downstream workflow to continue after a successful empty response.

---

## 10.9 Remove Office 365 E5

Graph endpoint:

```text
POST /users/{id}/assignLicense
```

Body:

```json
{
  "addLicenses": [],
  "removeLicenses": [
    "c7df2760-2c81-4ef7-b578-5b5392b571df"
  ]
}
```

The account is disabled but not deleted.

Reference:

![License Removed](../screenshots/04-03-license-removed-after-offboarding.png)

---

## 10.10 Planner and Teams

Planner:

```text
Plan: HR Operations Tasks
Bucket: Offboarding
```

Teams:

```text
Team: HR Operations
Channel: Offboarding
```

Reference:

![Offboarding Planner](../screenshots/04-05-offboarding-planner-tasks.png)

![Offboarding Teams](../screenshots/04-06-offboarding-teams-notification.png)

---

## 10.11 Manager Notification

Send the offboarding notification through Outlook.

Reference:

![Offboarding Manager Email](../screenshots/04-07-offboarding-manager-email.png)

---

# 11. HR-05 — Recruiting Intake

## 11.1 SharePoint List

Create:

```text
Recruiting Candidates
```

Core columns:

| Column | Type |
|---|---|
| Title | Default SharePoint text field |
| CandidateEmail | Single line of text |
| RoleApplied | Single line of text |
| CandidateSource | Single line of text |
| Status | Single line of text |
| RecruiterNotes | Multiple lines of text |
| ApplicationDate | Date |

---

## 11.2 MindStudio Recruiting Form

Create the branch:

```text
Recruiting
    ↓
Recruiting Form
    ↓
Send Candidate to n8n
    ↓
Recruiting End
```

Form variables:

```text
candidate_name
candidate_email
role_applied
candidate_source
recruiter_notes
application_date
```

Reference:

![Recruiting Form](../screenshots/05-01-recruiting-form-lena-ortiz.png)

---

## 11.3 N8N Run Node

Use separate key/value rows.

Do not send the entire payload as a single key unless the normalization layer is intentionally designed for it.

Rows:

| Key | Value |
|---|---|
| candidate_name | `{{ candidate_name }}` |
| candidate_email | `{{ candidate_email }}` |
| role_applied | `{{ role_applied }}` |
| candidate_source | `{{ candidate_source }}` |
| recruiter_notes | `{{ recruiter_notes }}` |
| application_date | `{{ application_date }}` |

HTTP method:

```text
POST
```

Authentication:

```text
None
```

Output variable:

```text
recruiting_response
```

---

## 11.4 n8n Workflow

Core flow:

```text
Webhook
    ↓
Normalize Candidate Intake
    ↓
Create SharePoint Candidate Record
    ↓
Create Recruiting Task
    ↓
Post Recruiting Notification
    ↓
Recruiting Success Response
```

Reference:

![HR-05 Core Workflow](../screenshots/05-02-hr05-core-workflow.png)

---

## 11.5 Webhook

Method:

```text
POST
```

Path:

```text
hr-recruiting-intake
```

Authentication:

```text
None
```

Response:

```text
Respond When Last Node Finishes
First Entry JSON
```

---

## 11.6 Normalize Candidate Intake

```javascript
const body = $json.body ?? {};
let data = body;

if (typeof body.candidate_name === 'undefined') {
  const key = Object.keys(body)[0];

  if (key === undefined) {
    throw new Error('No recruiting data received');
  }

  const raw =
    typeof body[key] === 'string' && body[key].trim()
      ? body[key]
      : key;

  data = JSON.parse(raw);
}

return [
  {
    json: {
      candidate_name: data.candidate_name ?? '',
      candidate_email: (data.candidate_email ?? '')
        .replace(/^Email:\s*/i, '')
        .trim()
        .toLowerCase(),
      role_applied: data.role_applied ?? '',
      candidate_source: data.candidate_source ?? '',
      recruiter_notes: data.recruiter_notes ?? '',
      application_date: data.application_date ?? '',
      source_system: 'MindStudio'
    }
  }
];
```

---

## 11.7 Create SharePoint Candidate Record

Select:

```text
Microsoft SharePoint
→ Create an item
```

Node name:

```text
Create SharePoint Candidate Record
```

Credential:

```text
Microsoft account
```

Resource:

```text
Item
```

Operation:

```text
Create
```

Site:

```text
HR Operations
```

List:

```text
Recruiting Candidates
```

Mappings:

**Title — EXPRESSION**

```text
{{ $('Normalize Candidate Intake').first().json.candidate_name }}
```

**CandidateEmail — EXPRESSION**

```text
{{ $('Normalize Candidate Intake').first().json.candidate_email }}
```

**RoleApplied — EXPRESSION**

```text
{{ $('Normalize Candidate Intake').first().json.role_applied }}
```

**CandidateSource — EXPRESSION**

```text
{{ $('Normalize Candidate Intake').first().json.candidate_source }}
```

**Status — FIXED**

```text
New
```

**RecruiterNotes — EXPRESSION**

```text
{{ $('Normalize Candidate Intake').first().json.recruiter_notes }}
```

**ApplicationDate — EXPRESSION**

```text
{{ $('Normalize Candidate Intake').first().json.application_date }}
```

Reference:

![Recruiting SharePoint Record](../screenshots/05-04-recruiting-sharepoint-record.png)

---

## 11.8 Create Recruiting Task

Use the same known-good Microsoft Teams / Planner credential used by HR-03 and HR-04.

Configure:

```text
Resource: Task
Operation: Create
Team: HR Operations
Plan: HR Operations Tasks
Bucket: Recruiting
```

Task title — **EXPRESSION**:

```text
{{ `Recruiting - ${$('Normalize Candidate Intake').first().json.candidate_name} - ${$('Normalize Candidate Intake').first().json.role_applied}` }}
```

Reference:

![Recruiting Planner Task](../screenshots/05-06-recruiting-planner-task.png)

---

## 11.9 Post Recruiting Notification

When searching Microsoft Teams actions, select the channel-message creation action.

Configure:

```text
Resource: Channel Message
Operation: Create
Team: HR Operations
Channel: Recruiting
```

Message — **EXPRESSION**:

```text
{{
`New recruiting candidate received

Candidate: ${$('Normalize Candidate Intake').first().json.candidate_name}
Email: ${$('Normalize Candidate Intake').first().json.candidate_email}
Role Applied For: ${$('Normalize Candidate Intake').first().json.role_applied}
Candidate Source: ${$('Normalize Candidate Intake').first().json.candidate_source}
Application Date: ${$('Normalize Candidate Intake').first().json.application_date}

Recruiter Notes:
${$('Normalize Candidate Intake').first().json.recruiter_notes || 'None provided'}

Status: New

Candidate record added to SharePoint and recruiting task created for HR review.`
}}
```

Reference:

![Recruiting Teams Notification](../screenshots/05-05-recruiting-teams-notification.png)

---

## 11.10 Recruiting Success Response

Use an Edit Fields node.

Fields:

**status — FIXED**

```text
success
```

**message — FIXED**

```text
Candidate recruiting intake completed successfully.
```

**candidate — EXPRESSION**

```text
{{ $('Normalize Candidate Intake').first().json.candidate_name }}
```

**role_applied — EXPRESSION**

```text
{{ $('Normalize Candidate Intake').first().json.role_applied }}
```

Keep only the configured response fields.

Reference:

![Recruiting Success Response](../screenshots/05-03-recruiting-success-response.png)

---

# 12. Structured Recruiting Review

The final enhancement adds structured resume-information extraction while leaving all hiring decisions with the recruiter.

The review layer must not:

- score candidates
- rank candidates
- recommend hiring
- reject applicants
- infer unsupported qualifications

It only organizes factual information submitted in the resume and recruiter notes.

---

## 12.1 MindStudio Resume Input

Add a file input to the Recruiting Form.

Variable:

```text
resume_text
```

Label:

```text
Resume / CV
```

Processing:

```text
Extract Text
```

The extracted text is sent to n8n.

Add a seventh Data row in the N8N Run Node:

**Key — FIXED**

```text
resume_text
```

**Value — VARIABLE**

```text
{{ resume_text }}
```

---

## 12.2 Update Normalize Candidate Intake

Extend the normalized output with:

```javascript
resume_text: data.resume_text ?? '',
```

The relevant output object becomes:

```javascript
json: {
  candidate_name: data.candidate_name ?? '',
  candidate_email: (data.candidate_email ?? '')
    .replace(/^Email:\s*/i, '')
    .trim()
    .toLowerCase(),
  role_applied: data.role_applied ?? '',
  candidate_source: data.candidate_source ?? '',
  recruiter_notes: data.recruiter_notes ?? '',
  application_date: data.application_date ?? '',
  resume_text: data.resume_text ?? '',
  source_system: 'MindStudio'
}
```

Reference:

![Resume Text Normalization](../screenshots/06-02-resume-text-normalization-output.png)

---

## 12.3 SharePoint Review Columns

Add these columns to `Recruiting Candidates`:

| Column | Type |
|---|---|
| AISummary | Multiple lines of text |
| Skills | Multiple lines of text |
| Experience | Multiple lines of text |
| Education | Multiple lines of text |
| Certifications | Multiple lines of text |
| MissingInformation | Multiple lines of text |

Reference:

![Review Columns](../screenshots/06-03-sharepoint-ai-review-columns.png)

---

## 12.4 Insert Structured Extraction

Final workflow:

```text
Webhook
    ↓
Normalize Candidate Intake
    ↓
AI Candidate Review
    ↓
Create SharePoint Candidate Record
    ↓
Create Recruiting Task
    ↓
Post Recruiting Notification
    ↓
Recruiting Success Response
```

Reference:

![Structured Review Workflow](../screenshots/06-01-hr05-ai-review-workflow.png)

Use an n8n:

```text
Information Extractor
```

Rename:

```text
AI Candidate Review
```

---

## 12.5 Extractor Input

Main text — **EXPRESSION**:

```text
{{
`Candidate: ${$('Normalize Candidate Intake').first().json.candidate_name}

Role Applied For:
${$('Normalize Candidate Intake').first().json.role_applied}

Resume / CV:
${$('Normalize Candidate Intake').first().json.resume_text}

Recruiter Notes:
${$('Normalize Candidate Intake').first().json.recruiter_notes}`
}}
```

---

## 12.6 Structured Fields

### summary

Type:

```text
String
```

Description:

```text
Write a concise factual recruiter-facing summary of the candidate's documented background. Use only information present in the resume or recruiter notes. Do not evaluate whether the candidate should be hired.
```

### skills

Type:

```text
String
```

Description:

```text
List the candidate's explicitly documented technical and professional skills. Do not infer skills that are not stated.
```

### experience

Type:

```text
String
```

Description:

```text
Summarize documented employment experience, roles, responsibilities, and years or dates when provided. Do not invent missing dates or experience.
```

### education

Type:

```text
String
```

Description:

```text
Summarize documented education, degrees, institutions, and fields of study. If none is provided, return "Not provided".
```

### certifications

Type:

```text
String
```

Description:

```text
List documented professional certifications. If none are provided, return "Not provided".
```

### missing_information

Type:

```text
String
```

Description:

```text
List important factual information that is absent or unclear and that a recruiter may want to verify. Do not judge the candidate.
```

---

## 12.7 Chat Model

Attach an OpenAI Chat Model to the Information Extractor.

Use the existing working OpenAI credential and model configuration.

No autonomous agent is required.

---

## 12.8 SharePoint Review Mapping

Add the following mappings to `Create SharePoint Candidate Record`.

**AISummary — EXPRESSION**

```text
{{ $('AI Candidate Review').first().json.output.summary }}
```

**Skills — EXPRESSION**

```text
{{ $('AI Candidate Review').first().json.output.skills }}
```

**Experience — EXPRESSION**

```text
{{ $('AI Candidate Review').first().json.output.experience }}
```

**Education — EXPRESSION**

```text
{{ $('AI Candidate Review').first().json.output.education }}
```

**Certifications — EXPRESSION**

```text
{{ $('AI Candidate Review').first().json.output.certifications }}
```

**MissingInformation — EXPRESSION**

```text
{{ $('AI Candidate Review').first().json.output.missing_information }}
```

Keep:

```text
Status = New
```

This preserves recruiter ownership of the next action.

Reference:

![Structured Candidate Record](../screenshots/06-04-sharepoint-ai-review-record-detail.png)

---

# 13. Final End-to-End Validation

## 13.1 HR-02

Validate:

- incoming HR email is detected
- HR self-message is filtered
- duplicate email is not processed twice
- classifier selects the correct category
- branch-specific task or response is created
- policy questions use SharePoint source content

---

## 13.2 HR-03

Validate:

- onboarding form submits successfully
- payload normalizes correctly
- duplicate user is blocked
- new user is created
- temporary password is generated
- password change is required at first sign-in
- Office 365 E5 is assigned
- Planner task exists
- Teams notification exists
- manager email is delivered
- welcome email is delivered
- success response is returned

---

## 13.3 HR-04

Validate:

- offboarding form submits
- target user is found
- missing user stops safely
- existing user is disabled
- Office 365 E5 is removed
- Planner task exists
- Teams notification exists
- manager email is delivered
- user account is retained rather than deleted

---

## 13.4 HR-05

Validate:

- recruiting form submits
- candidate data normalizes
- SharePoint candidate record is created
- Planner recruiting task is created
- Teams recruiting notification is posted
- success response is returned

---

## 13.5 Structured Recruiting Review

Validate:

- resume text reaches n8n
- normalized `resume_text` is not blank
- extractor returns:
  - summary
  - skills
  - experience
  - education
  - certifications
  - missing information
- SharePoint stores all review fields
- Planner and Teams behavior remains unchanged
- output contains no ranking, hire recommendation, rejection decision, or score

---

# 14. Important Troubleshooting Notes

## 14.1 n8n Editor vs Executions

The editor can display stale or absent data after a webhook run.

When verifying a live workflow, use:

```text
Executions
```

as the source of truth.

Confirm that the execution belongs to the correct workflow before diagnosing a failure.

---

## 14.2 Teams 403 Scope Error

Symptom:

```text
403 - Missing scope permissions
```

when loading Teams or Planner data.

Typical cause:

The Teams node is using the SharePoint generic `Microsoft account` credential.

Resolution:

Use the same working Microsoft Teams credential already validated in HR-03 or HR-04.

---

## 14.3 Graph PATCH Returns No JSON

A successful Graph PATCH may return:

```text
204 No Content
```

If downstream n8n nodes require output:

```text
Always Output Data = ON
```

---

## 14.4 MindStudio Payload Shape

MindStudio may send standard key/value fields or a JSON string embedded in an unexpected request-body shape.

The normalization Code nodes in HR-03, HR-04, and HR-05 intentionally handle both formats.

---

## 14.5 MindStudio Checkpoint Error

Observed error:

```text
Checkpoint has no transition
```

The Checkpoint was not required for the workflow and was removed.

---

## 14.6 MindStudio Browser Testing

Use:

```text
Preview
→ Open Draft Agent
```

A debugger state of:

```text
Waiting for Input
```

does not mean the form is broken.

---

## 14.7 Shared Mailbox Sent Items

Messages sent through the shared mailbox may not immediately appear in the expected Sent Items view.

Use Microsoft message trace or recipient delivery as the authoritative validation when needed.

---

# 15. Security and Repository Hygiene

Do not commit:

- API keys
- OAuth secrets
- access tokens
- refresh tokens
- temporary user passwords
- webhook secrets
- production webhook URLs
- authentication cookies
- exported credential files

The project screenshots were curated before publication. Screens exposing credential configuration, consent details, a production webhook URL, debugging-only states, or unrelated desktop content were excluded.

The published screenshots preserve the full-screen context where possible.

---

# 16. Completed Workflow Inventory

```text
HR-01 - Microsoft 365 Connection Test

HR-02 - Employee Request Intake
├── Leave / PTO
├── Employment Documents
├── Benefits
└── HR Policy / General Question

HR-03 - New Hire Onboarding
├── Duplicate-user protection
├── Entra user creation
├── Microsoft 365 E5 assignment
├── Planner task
├── Teams notification
├── Hiring-manager notification
└── Welcome email

HR-04 - Employee Offboarding
├── Employee-exists validation
├── Account disablement
├── Microsoft 365 E5 removal
├── Planner task
├── Teams notification
└── Manager notification

HR-05 - Recruiting Intake
├── MindStudio candidate intake
├── SharePoint candidate record
├── Planner recruiting task
├── Teams recruiting notification
├── Structured resume-information extraction
└── Recruiter-facing SharePoint review fields
```

---

# 17. Final State

The HR Automation Ecosystem is complete.

The environment demonstrates:

- human-facing HR intake through MindStudio
- workflow orchestration through n8n
- Microsoft 365 integration
- identity provisioning and deprovisioning
- automated task creation
- cross-platform notifications
- structured SharePoint records
- grounded HR policy responses
- duplicate and missing-user safeguards
- controlled recruiter-facing resume review

The repository's main `README.md` presents the project as a portfolio piece. This implementation guide preserves the technical detail required to reproduce or extend the lab later.
