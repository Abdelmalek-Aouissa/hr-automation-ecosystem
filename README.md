# HR Automation Ecosystem

End-to-end HR operations lab integrating **MindStudio, n8n, Microsoft 365, Microsoft Entra ID, SharePoint, Teams, Planner, Outlook, and structured AI-assisted recruiting review**.

The environment demonstrates how common HR processes can be connected across intake, routing, identity administration, task management, collaboration, communications, and recruiter review while preserving human oversight for employment decisions.

![HR Operations Portal](screenshots/00-01-final-hr-operations-portal.png)

## Project Overview

The lab was built around a Microsoft 365 E5 environment and an HR Operations workspace. MindStudio provides the user-facing entry point, while n8n coordinates the workflow logic and Microsoft 365 services provide the operational systems of record and collaboration layer.

The completed ecosystem covers employee HR request intake and classification, grounded policy-question handling, new-hire onboarding, Entra user creation and Microsoft 365 licensing, duplicate-user protection, employee offboarding, account disablement and license removal, recruiting intake, candidate records, Planner tasks, Teams notifications, Outlook communications, and structured resume review for recruiter use.

The lab uses synthetic test identities and candidate records. No production employee data is used.

---

## Architecture

```mermaid
flowchart LR
    A[MindStudio HR Operations Portal] --> B[n8n Workflow Layer]
    B --> C[Microsoft Entra ID]
    B --> D[SharePoint]
    B --> E[Microsoft Planner]
    B --> F[Microsoft Teams]
    B --> G[Outlook / Shared HR Mailbox]
    B --> H[Structured Candidate Review]
    C --> I[Identity Lifecycle]
    D --> J[HR Knowledge & Records]
    E --> K[Operational Tasks]
    F --> L[Team Notifications]
    G --> M[Employee / Manager Communications]
    H --> N[Recruiter Review]
```

### Core Platforms

| Platform | Role in the environment |
|---|---|
| MindStudio | HR Operations portal and user-facing forms |
| n8n | Workflow orchestration and branching logic |
| Microsoft Entra ID | User lifecycle and account state |
| Microsoft 365 E5 | Productivity and licensing environment |
| SharePoint | HR policy knowledge and recruiting records |
| Microsoft Planner | Recruiting, onboarding, employee-request, and offboarding tasks |
| Microsoft Teams | HR Operations collaboration and workflow notifications |
| Outlook | Shared HR mailbox and automated communications |
| Microsoft Graph | Identity and license operations where direct Graph calls were required |
| OpenAI model through n8n | Structured extraction for recruiter-facing resume review |

---

## HR Operations Foundation

The environment uses a dedicated **HR Operations** team with functional channels and a Planner plan aligned to the main HR workstreams.

![HR Operations Team](screenshots/01-01-hr-operations-team.png)

![HR Operations Planner Buckets](screenshots/01-04-hr-operations-planner-buckets.png)

The Planner structure covers Recruiting, Onboarding, Employee Requests, and Offboarding. SharePoint provides both the HR document workspace and the structured data used by the policy and recruiting workflows.

![SharePoint HR Operations](screenshots/01-05-sharepoint-hr-operations-folders.png)

<details>
<summary><strong>View hands-on implementation steps</strong></summary>

### Microsoft 365 and HR Operations setup

1. A Microsoft 365 E5 lab tenant was prepared with HR, recruiting, management, employee, and IT test identities.
2. A private Microsoft Teams team named **HR Operations** was created.
3. Functional channels were added for General, Recruiting, Onboarding, Employee Requests, and Offboarding.
4. Team membership was assigned to the HR test users.

![HR Operations Team Members](screenshots/01-02-hr-operations-team-members.png)

5. A shared HR mailbox was created for centralized employee communications.

![Shared HR Mailbox](screenshots/01-03-shared-hr-mailbox-created.png)

6. The associated SharePoint site was used as the HR document and list workspace.
7. A Planner plan named **HR Operations Tasks** was created.
8. Planner buckets were aligned with Recruiting, Onboarding, Employee Requests, and Offboarding.
9. Microsoft 365 connections were validated in n8n before the production workflows were assembled.

</details>

---

## HR-02 — Employee Request Intake

HR-02 routes incoming employee requests from the shared HR mailbox into the appropriate HR process. Requests are normalized, protected against self-generated email loops, de-duplicated, classified, and then routed to the relevant branch.

Supported categories include Leave / PTO, Benefits, Payroll / Compensation, Employment Documents, HR Policy / General Question, Recruiting, Onboarding, Offboarding, and Other.

![HR-02 Complete Workflow](screenshots/02-07-hr02-complete-workflow.png)

A PTO request can enter through email and produce the corresponding operational task and Teams notification.

![PTO Request](screenshots/02-01-pto-request-email.png)

![Employee Request Planner](screenshots/02-04-employee-request-planner-board.png)

![Employee Request Teams Notifications](screenshots/02-05-employee-request-teams-notifications.png)

The same intake framework supports employment-document, benefits, and policy requests.

![Employment Verification Request](screenshots/02-06-employment-verification-request-email.png)

![Benefits Request](screenshots/02-08-benefits-request-email.png)

### HR Policy Knowledge

Policy questions use a dedicated SharePoint list as the grounded knowledge source rather than relying on unrestricted generated answers.

![HR Policy Knowledge List](screenshots/02-09-hr-policy-knowledge-list.png)

![Remote Work Policy Request](screenshots/02-10-remote-work-policy-request-email.png)

<details>
<summary><strong>View hands-on implementation steps</strong></summary>

### Intake and normalization

1. An Outlook trigger monitored the HR mailbox.
2. Incoming messages were normalized into consistent fields used throughout the workflow.
3. A self-message safeguard prevented messages sent by the HR mailbox from re-entering the workflow.
4. Duplicate protection used the Microsoft message identifier to prevent repeat processing.

### Request classification

5. A Text Classifier node was configured with the HR request categories.

![Text Classifier Categories](screenshots/02-02-text-classifier-categories.png)

6. The classifier routed each request to the corresponding branch.
7. Structured extraction was used where a branch required specific fields from the employee message.

![Information Extractor Configuration](screenshots/02-03-information-extractor-configuration.png)

### Leave / PTO

8. Leave-related requests were converted into an HR operational task.
9. A notification was posted in the HR Operations workspace.
10. Employee communication was processed through the shared mailbox.

### Employment Documents

11. Employment-document requests were separated from general HR questions.
12. The branch created the corresponding HR follow-up activity and notification.

### Benefits

13. Benefits-related messages were classified independently and routed to the benefits branch.
14. The original request context was preserved for HR review.

### HR Policy / General Question

15. A SharePoint list named **HR Policy Knowledge** was created with policy title, category, policy text, and effective-date fields.
16. The branch extracted the employee's question.
17. Matching policy records were retrieved from SharePoint.
18. The response was generated only from the retrieved policy content.
19. The reply was sent through the HR mailbox.
20. The source email was marked as read after successful processing.

</details>

---

## HR-03 — New Hire Onboarding

HR-03 provides an end-to-end onboarding path from the HR portal to Microsoft 365. It receives the new-hire form, checks for an existing account, creates the user when appropriate, assigns the Microsoft 365 E5 license, creates the onboarding task, posts the Teams notification, informs the hiring manager, sends the welcome email, and returns a structured result.

![HR-03 Complete Workflow](screenshots/03-06-hr03-complete-workflow.png)

Successful onboarding creates the Entra account and assigns the required Microsoft 365 license.

![New Hire Entra User](screenshots/03-02-new-hire-entra-user-with-license.png)

Operational follow-through is visible in the same Microsoft 365 tools used by the HR team.

![Onboarding Teams Notification](screenshots/03-04-onboarding-teams-notification.png)

![New Hire Welcome Email](screenshots/03-07-new-hire-welcome-email.png)

![Hiring Manager Notification](screenshots/03-08-hiring-manager-notification-email.png)

<details>
<summary><strong>View hands-on implementation steps</strong></summary>

### MindStudio onboarding form

1. The **Start Onboarding** branch was added to the HR Operations portal.
2. The form collected first name, last name, personal email, job title, department, manager email, and start date.

![MindStudio Onboarding Form](screenshots/03-01-mindstudio-onboarding-form.png)

3. The form submitted the new-hire data to the HR-03 n8n webhook.

### Input normalization and duplicate protection

4. A normalization step standardized the MindStudio payload before Microsoft 365 actions were allowed to run.
5. Microsoft Graph queried Entra ID for the expected user principal name.
6. The workflow continued only when no matching user was found.
7. Existing users were routed to a safe duplicate branch.

![Duplicate User Protection](screenshots/03-05-duplicate-user-protection-branch.png)

### Account creation and licensing

8. A temporary password seed was generated.
9. The Microsoft Entra ID node created the new user.
10. Password change at first sign-in was enforced.
11. The Office 365 E5 license was assigned through Microsoft Graph.

### Operational tasks and notifications

12. A Planner task was created in the **Onboarding** bucket.
13. A Teams notification was posted to the **Onboarding** channel.
14. The hiring manager received an Outlook notification.
15. The new hire received the welcome email at the personal email address.
16. The workflow returned a structured success response.

![HR-03 Successful Execution](screenshots/03-03-hr03-successful-execution.png)

</details>

---

## HR-04 — Employee Offboarding

HR-04 implements controlled offboarding without deleting the user account. The workflow verifies that the employee exists, disables the Entra account, removes the Microsoft 365 E5 license, creates the offboarding task, posts the Teams notification, and informs the manager.

![HR-04 Complete Workflow](screenshots/04-04-hr04-complete-successful-workflow.png)

The account remains available for administrative follow-up while sign-in access is disabled and the E5 license is reclaimed.

![License Removed After Offboarding](screenshots/04-03-license-removed-after-offboarding.png)

![Offboarding Planner Tasks](screenshots/04-05-offboarding-planner-tasks.png)

![Offboarding Teams Notification](screenshots/04-06-offboarding-teams-notification.png)

![Offboarding Manager Email](screenshots/04-07-offboarding-manager-email.png)

<details>
<summary><strong>View hands-on implementation steps</strong></summary>

### MindStudio offboarding form

1. The **Start Offboarding** branch was added to the HR Operations portal.
2. The form collected employee name, user principal name, manager email, last working date, reason, and notes.

![MindStudio Offboarding Flow](screenshots/04-01-mindstudio-offboarding-flow.png)

3. The submission was sent to the HR-04 webhook.

### Employee validation

4. The incoming request was normalized.
5. Microsoft Graph searched Entra ID for the submitted user principal name.
6. A missing employee was routed to a safe stop branch rather than continuing with account or licensing actions.

![Employee Not Found Branch](screenshots/04-02-employee-not-found-branch.png)

### Account disablement and license removal

7. The Entra user was updated with `accountEnabled: false`.
8. The account itself was retained rather than deleted.
9. The Microsoft 365 E5 SKU was removed through Microsoft Graph.
10. Successful Graph responses were preserved in the workflow when the API returned an empty `204` response.

### Operational tasks and notifications

11. A Planner task was created in the **Offboarding** bucket.
12. A Teams notification was posted to the **Offboarding** channel.
13. The employee's manager received an Outlook notification.
14. The workflow returned a structured completion response.

</details>

---

## HR-05 — Recruiting Intake

HR-05 captures candidate information through the HR Operations portal and converts it into a consistent recruiting record and follow-up workflow. The core process creates a SharePoint candidate record, a Planner recruiting task, a Teams recruiting notification, and a structured success response.

![HR-05 Core Workflow](screenshots/05-02-hr05-core-workflow.png)

![Recruiting Form](screenshots/05-01-recruiting-form-lena-ortiz.png)

![Recruiting SharePoint Record](screenshots/05-04-recruiting-sharepoint-record.png)

![Recruiting Planner Task](screenshots/05-06-recruiting-planner-task.png)

![Recruiting Teams Notification](screenshots/05-05-recruiting-teams-notification.png)

<details>
<summary><strong>View hands-on implementation steps</strong></summary>

### SharePoint recruiting list

1. A SharePoint list named **Recruiting Candidates** was created.
2. The core fields included Title, CandidateEmail, RoleApplied, CandidateSource, Status, RecruiterNotes, and ApplicationDate.

### MindStudio recruiting form

3. The **Recruiting** branch was added to the HR Operations portal.
4. The form collected candidate name, candidate email, role applied, candidate source, recruiter notes, and application date.
5. The form submitted the candidate data to the HR-05 n8n webhook.

### Candidate normalization and record creation

6. The incoming payload was normalized into consistent candidate fields.
7. Candidate email values were cleaned and standardized before record creation.
8. A new SharePoint item was created in **Recruiting Candidates**.
9. The candidate status was initialized as **New**.

### Recruiting task and notification

10. A Planner task was created in the **Recruiting** bucket.
11. The task title followed the format `Recruiting - Candidate Name - Role Applied`.
12. A recruiting notification was posted to the **Recruiting** channel.
13. The notification included candidate identity, role, source, application date, recruiter notes, and current status.

### Completion response

14. The workflow returned a structured success response containing the candidate name and role.

![Recruiting Success Response](screenshots/05-03-recruiting-success-response.png)

</details>

---

## Structured Recruiting Review

The recruiting workflow was extended with a structured resume-review stage designed to reduce manual reading time while keeping the employment decision with the recruiter.

The review layer extracts documented information from the submitted resume and recruiter notes into consistent fields for candidate summary, skills, experience, education, certifications, and missing or unclear information.

The workflow does **not** rank candidates, assign suitability scores, reject applicants, or make hiring decisions.

![HR-05 Structured Review Workflow](screenshots/06-01-hr05-ai-review-workflow.png)

The resume is uploaded through the recruiting form, converted to text, normalized by the workflow, and passed to the structured extraction stage.

![Resume Text Normalization](screenshots/06-02-resume-text-normalization-output.png)

The extracted fields are written back to the candidate record in SharePoint.

![Structured Review Columns](screenshots/06-03-sharepoint-ai-review-columns.png)

![Structured Review Record](screenshots/06-04-sharepoint-ai-review-record-detail.png)

<details>
<summary><strong>View hands-on implementation steps</strong></summary>

### Resume input

1. The MindStudio recruiting form was extended with a resume/CV file input.
2. File processing extracted the document text for workflow use.
3. The extracted resume text was included in the recruiting webhook payload.

### Workflow normalization

4. The HR-05 normalization step was extended with a `resume_text` field.
5. The normalized resume content remained associated with the existing candidate record.

### Structured extraction

6. An Information Extractor stage was inserted before SharePoint candidate creation.
7. The extractor was configured with six recruiter-facing outputs: summary, skills, experience, education, certifications, and missing information.
8. Extraction was constrained to factual use of the submitted resume and recruiter notes.
9. Unsupported or absent information was not invented.

### SharePoint extension

10. The Recruiting Candidates list was extended with AISummary, Skills, Experience, Education, Certifications, and MissingInformation.
11. Structured extraction output was mapped into these fields.
12. The original candidate status remained **New**, preserving recruiter control over the next action.

### Validation

13. A sample IT support resume was submitted through the full recruiting workflow.
14. n8n confirmed that the resume text was received and normalized.
15. The structured review produced the expected recruiter-facing fields.
16. SharePoint stored the review alongside the original candidate record.
17. Planner and Teams continued to receive the standard recruiting task and notification.

</details>

---

## Reliability and Control Measures

| Control | Purpose |
|---|---|
| Self-message filter | Prevents the HR mailbox from triggering on its own automated replies |
| Message de-duplication | Prevents repeated processing of the same employee request |
| Duplicate-user check | Prevents onboarding from recreating an existing Entra account |
| Employee-exists check | Prevents offboarding actions against a nonexistent user |
| Account disablement instead of deletion | Preserves administrative recovery and follow-up options |
| License removal | Reclaims Microsoft 365 licensing during offboarding |
| Grounded policy source | Keeps policy answers tied to SharePoint policy content |
| Human-controlled recruiting status | Keeps employment decisions with HR/recruiting personnel |
| Structured resume extraction only | Summarizes submitted evidence without automated ranking or selection |

---

## Validation Summary

| Workflow | Validation |
|---|---|
| HR-02 Employee Request Intake | Email intake, classification, Planner, Teams, policy retrieval, and replies |
| HR-03 New Hire Onboarding | MindStudio form, Entra user creation, E5 assignment, Planner, Teams, manager notification, welcome email |
| HR-03 Duplicate Protection | Existing user stopped safely before account creation |
| HR-04 Employee Offboarding | User lookup, account disablement, license removal, Planner, Teams, manager notification |
| HR-04 Missing User | Workflow stopped safely without account or licensing action |
| HR-05 Recruiting Intake | Candidate form, SharePoint record, Planner task, Teams notification, success response |
| Structured Recruiting Review | Resume ingestion, extraction, SharePoint review fields, existing recruiting follow-up preserved |

---

## Repository Structure

```text
hr-automation-ecosystem/
│
├── README.md
└── screenshots/
    ├── 00-01-final-hr-operations-portal.png
    ├── 01-01-hr-operations-team.png
    ├── 01-02-hr-operations-team-members.png
    ├── ...
    ├── 05-06-recruiting-planner-task.png
    ├── 06-01-hr05-ai-review-workflow.png
    ├── 06-02-resume-text-normalization-output.png
    ├── 06-03-sharepoint-ai-review-columns.png
    └── 06-04-sharepoint-ai-review-record-detail.png
```

---

## Operational Takeaways

This lab demonstrates practical integration across HR process intake, workflow orchestration, identity lifecycle management, collaboration tools, task management, structured data, and controlled language-model extraction.

The implementation emphasizes consistent data handoff between user-facing forms and back-end workflows, defensive branching before identity or licensing changes, separation of HR records from operational tasks and communications, reuse of Microsoft 365 services already present in a business environment, and human ownership of recruiting and employment decisions.

The result is a complete HR automation ecosystem that can be reviewed from both a technical implementation perspective and an operational business-process perspective.
