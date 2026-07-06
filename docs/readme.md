Below is the **technical proof-of-concept setup** references validated for automation in Egnyte.

Future reference: This document assumes the Egnyte tenant has **Workflow Templates / multi-step workflows** enabled.

---

# POC Goal

Prove this lifecycle:

```text
User changes metadata/status tag
        ↓
Egnyte workflow starts
        ↓
Approver is notified
        ↓
Approver approves
        ↓
Egnyte moves file to the correct lifecycle folder
        ↓
Egnyte stamps/updates lifecycle metadata
        ↓
Distribution owner/test user is notified
```

For the test:

* Contributor / distribution tester: `testuser01`
* Approver: `testapprover01`

---

# 1. Create the POC folder structure

Create this folder tree in Egnyte:

```text
/Shared/Marketing Asset Lifecycle POC
├── 01_Working Drafts
├── 02_Review Queue
├── 03_Approved Pending Distribution
├── 04_Current Distributed Assets
└── 05_Archived Previous Versions
```

For the POC, start with simple permissions:

| Folder                             |      testuser01 |  testapprover01 |
| ---------------------------------- | --------------: | --------------: |
| `01_Working Drafts`                |   Editor / Full | Viewer / Editor |
| `02_Review Queue`                  | Viewer / Editor |   Editor / Full |
| `03_Approved Pending Distribution` |          Viewer |   Editor / Full |
| `04_Current Distributed Assets`    |          Viewer |   Editor / Full |
| `05_Archived Previous Versions`    |          Viewer |   Editor / Full |

The important governance idea: `testuser01` can create and submit, but should not own final approval. `testapprover01` is the approval gate.

---

# 2. Create POC users or groups

For a cleaner proof, use groups even if each group only has one test user.

Create these groups:

```text
MKTG-POC-Contributors
MKTG-POC-Approvers
MKTG-POC-Distribution-Owners
```

Membership:

```text
MKTG-POC-Contributors
└── testuser01

MKTG-POC-Approvers
└── testapprover01

MKTG-POC-Distribution-Owners
└── testuser01
```

Why put `testuser01` in Distribution Owners? Because you specifically want to test notifications to `testuser01`. In a real setup, this might be a marketing operations person or campaign owner.

---

# 3. Create custom metadata section

In Egnyte, go to:

```text
Settings / Configuration
→ Metadata Tags / Custom Metadata
→ Add New Section
```

Exact menu wording may vary slightly by Egnyte plan/UI version, but Egnyte’s metadata setup is managed from the Web UI and administrators can create metadata sections and properties for files/folders. ([Egnyte Helpdesk][2])

Create metadata section:

```text
Marketing Asset Lifecycle
```

Add these fields:

| Field              | Type                            | Values                                                                                                              |
| ------------------ | ------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| `Lifecycle Status` | Dropdown / single select        | `In Progress`, `Ready for Review`, `Approved - Pending Distribution`, `Live / Distributed`, `Archived / Superseded` |
| `Asset Type`       | Dropdown / single select        | `Slide Deck`, `One-Pager`, `Flyer`, `Case Study`, `Campaign Asset`, `Sales Collateral`                              |
| `Owner`            | Text or user field if available | Freeform for POC                                                                                                    |
| `Approver`         | Text or user field if available | Freeform for POC                                                                                                    |
| `Version`          | Text                            | Example: `v0.1`, `v1.0`                                                                                             |
| `Distributed Date` | Date                            | blank until live                                                                                                    |
| `Review Date`      | Date                            | optional                                                                                                            |

For the POC, the **critical field** is:

```text
Lifecycle Status
```

That is the source-of-truth field.

---

# 4. Create test asset

Upload a test file to:

```text
/Shared/Marketing Asset Lifecycle POC/01_Working Drafts
```

Example file:

```text
POC_Service_Overview_Deck_v0.1.pptx
```

Set metadata:

```text
Lifecycle Status: In Progress
Asset Type: Slide Deck
Owner: testuser01
Approver: testapprover01
Version: v0.1
```

This represents the starting point.

---

# 5. Create Workflow Template 1 — Ready for Review

This workflow proves:

```text
Status = Ready for Review
→ move file to Review Queue
→ notify approver
```

Go to:

```text
Settings
→ Configuration
→ Workflows
→ Templates
→ Add Template
→ File Workflow Template
```

Egnyte’s Workflow Templates are accessed through the Workflows section in Configuration settings, and template access can be controlled by folder location, user group, and file extension. ([Egnyte Helpdesk][1])

Name the template:

```text
MKTG POC - Ready for Review Routing
```

## Template access / scope

Limit it to:

```text
Folder location:
 /Shared/Marketing Asset Lifecycle POC/01_Working Drafts

Allowed users/groups:
 MKTG-POC-Contributors

File types:
 pptx, pdf, docx
```

## Trigger

Configure workflow trigger:

```text
When specific metadata is added/updated on a file
Metadata section: Marketing Asset Lifecycle
Field: Lifecycle Status
Condition: equals
Value: Ready for Review
```

Egnyte supports workflow triggers where a workflow starts when specific metadata is added or updated and matches a configured condition. ([Egnyte Helpdesk][3])

## Steps

### Step 1 — Automated Action

Type:

```text
Automated Action
```

Action:

```text
Move file to a folder
```

Destination:

```text
/Shared/Marketing Asset Lifecycle POC/02_Review Queue
```

Egnyte supports a **Move file to a folder** automated action inside Workflow Templates. ([Egnyte Helpdesk][4])

Failure handling:

```text
If step fails:
Mark step as Failed and cancel workflow
```

For a POC, I would cancel on failure so the test failure is obvious.

### Step 2 — Review Task

Type:

```text
Review
```

Assignee:

```text
testapprover01
```

Task name:

```text
Review marketing asset
```

Instructions:

```text
Please review this marketing asset. If changes are needed, request revisions. If acceptable, approve it for pending distribution.
```

Notification:

```text
Email notification enabled
Recipient: testapprover01
```

Egnyte workflows support customizable workflow notifications, and task assignees receive email notifications for new workflow tasks. ([Egnyte Helpdesk][1])

---

# 6. Create Workflow Template 2 — Approval Gate

This workflow proves:

```text
Approver approval
→ add Approved - Pending Distribution metadata
→ move file to approved pending distribution folder
→ notify testuser01
```

Name:

```text
MKTG POC - Approval to Pending Distribution
```

## Scope

```text
Folder location:
 /Shared/Marketing Asset Lifecycle POC/02_Review Queue

Allowed users/groups:
 MKTG-POC-Approvers

File types:
 pptx, pdf, docx
```

## Recommended trigger

For the cleanest POC, use the workflow completion/approval pattern:

```text
Trigger:
Workflow from Ready for Review template is completed / approved
```

Egnyte’s workflow triggers include the ability to create automated workflows when a workflow from a specific template is completed. ([Egnyte Helpdesk][5])

## Steps

### Step 1 — Approval

Type:

```text
Approval
```

Assignee:

```text
testapprover01
```

Task name:

```text
Final approval gate
```

Instructions:

```text
Approve this file only if it is ready to be released into Approved Pending Distribution.
```

Approval result:

```text
Approved
```

### Step 2 — Automated Action: Add metadata

Type:

```text
Automated Action
```

Action:

```text
Add metadata to file
```

Metadata:

```text
Marketing Asset Lifecycle
Lifecycle Status = Approved - Pending Distribution
Approver = testapprover01
```

Egnyte supports an **Add metadata to file** automated action in workflow templates. ([Egnyte Helpdesk][6])

### Step 3 — Automated Action: Move file

Type:

```text
Automated Action
```

Action:

```text
Move file to a folder
```

Destination:

```text
/Shared/Marketing Asset Lifecycle POC/03_Approved Pending Distribution
```

### Step 4 — Notification / To-Do

Type:

```text
To Do
```

Assignee:

```text
testuser01
```

Task name:

```text
Asset approved - prepare distribution
```

Instructions:

```text
This asset has been approved and moved to Approved Pending Distribution. Please prepare it for release and then set Lifecycle Status to Live / Distributed when it is actually distributed.
```

Notification:

```text
Email notification enabled
Recipient: testuser01
```

This is your first direct notification test to `testuser01`.

---

# 7. Create Workflow Template 3 — Live / Distributed

This workflow proves:

```text
Status = Live / Distributed
→ move file to Current Distributed Assets
→ stamp distributed metadata
→ notify testuser01 and/or testapprover01
```

Name:

```text
MKTG POC - Distribution to Live
```

## Scope

```text
Folder location:
 /Shared/Marketing Asset Lifecycle POC/03_Approved Pending Distribution

Allowed users/groups:
 MKTG-POC-Distribution-Owners

File types:
 pptx, pdf, docx
```

## Trigger

```text
When specific metadata is added/updated on a file
Metadata section: Marketing Asset Lifecycle
Field: Lifecycle Status
Condition: equals
Value: Live / Distributed
```

## Steps

### Step 1 — Automated Action: Move file

Type:

```text
Automated Action
```

Action:

```text
Move file to a folder
```

Destination:

```text
/Shared/Marketing Asset Lifecycle POC/04_Current Distributed Assets
```

### Step 2 — Automated Action: Add metadata

Type:

```text
Automated Action
```

Action:

```text
Add metadata to file
```

Metadata:

```text
Distributed Date = today’s date
Owner = testuser01
Version = v1.0
```

Whether “today’s date” can be dynamically stamped depends on your Egnyte workflow field options. If dynamic date stamping is not available in the UI, use a manual `Distributed Date` field for the POC, or handle the dynamic stamp through Power Automate/API later.

### Step 3 — Notification / To-Do

Type:

```text
To Do
```

Assignees:

```text
testuser01
testapprover01
```

Task name:

```text
Asset is now live
```

Instructions:

```text
This asset has been moved to Current Distributed Assets and is now the active version for use.
```

This proves both notification paths.

---

# 8. Create Workflow Template 4 — Archive / Superseded

This proves the archive behavior.

Name:

```text
MKTG POC - Archive Superseded Asset
```

## Scope

```text
Folder location:
 /Shared/Marketing Asset Lifecycle POC/04_Current Distributed Assets

Allowed users/groups:
 MKTG-POC-Approvers
 MKTG-POC-Distribution-Owners
```

## Trigger

```text
When specific metadata is added/updated on a file
Metadata section: Marketing Asset Lifecycle
Field: Lifecycle Status
Condition: equals
Value: Archived / Superseded
```

## Steps

### Step 1 — Automated Action: Move file

Action:

```text
Move file to a folder
```

Destination:

```text
/Shared/Marketing Asset Lifecycle POC/05_Archived Previous Versions
```

### Step 2 — Notification

Assignees:

```text
testuser01
testapprover01
```

Task name:

```text
Asset archived
```

Instructions:

```text
This asset has been archived and should no longer be used as the current live version.
```

---

# 9. Configure folder notifications as an extra verification layer

This is optional, but good for testing.

For each lifecycle folder, enable folder notifications for `testuser01` and/or `testapprover01`.

At minimum:

```text
02_Review Queue
Notify: testapprover01
Events: file additions, updates, moves

03_Approved Pending Distribution
Notify: testuser01
Events: file additions, updates, moves

04_Current Distributed Assets
Notify: testuser01, testapprover01
Events: file additions, updates, moves

05_Archived Previous Versions
Notify: testuser01, testapprover01
Events: file additions, moves
```

Egnyte folder notifications can alert users about actions such as uploads, file updates, deletions, moves, copies, downloads, and previews. ([Egnyte Helpdesk][7])

This gives you a backup signal even if workflow-task notifications are configured incorrectly.

---

# 10. Test run script

Use this exact test flow.

## Test 1 — Draft to Review

As `testuser01`:

1. Upload:

```text
POC_Service_Overview_Deck_v0.1.pptx
```

to:

```text
01_Working Drafts
```

2. Set metadata:

```text
Lifecycle Status = In Progress
Asset Type = Slide Deck
Owner = testuser01
Approver = testapprover01
Version = v0.1
```

3. Change:

```text
Lifecycle Status = Ready for Review
```

Expected result:

```text
File moves to 02_Review Queue
testapprover01 receives notification/task
Workflow history shows movement action completed
```

---

## Test 2 — Approval to Pending Distribution

As `testapprover01`:

1. Open the workflow task.
2. Review the file.
3. Approve it.

Expected result:

```text
Lifecycle Status becomes Approved - Pending Distribution
File moves to 03_Approved Pending Distribution
testuser01 receives notification/task
Workflow history shows approval and move action
```

---

## Test 3 — Pending Distribution to Live

As `testuser01` or distribution owner:

1. Open the file in:

```text
03_Approved Pending Distribution
```

2. Change metadata:

```text
Lifecycle Status = Live / Distributed
Version = v1.0
Distributed Date = current test date
```

Expected result:

```text
File moves to 04_Current Distributed Assets
testuser01 receives live confirmation
testapprover01 receives live confirmation
Metadata shows Live / Distributed
```

---

## Test 4 — Archive old version

Upload a second test file:

```text
POC_Service_Overview_Deck_v1.1.pptx
```

Run it through the same lifecycle until it becomes live.

Then manually change the old file:

```text
Lifecycle Status = Archived / Superseded
```

Expected result:

```text
Old file moves to 05_Archived Previous Versions
Old file is no longer in Current Distributed Assets
New file is the only current version in 04_Current Distributed Assets
testuser01 and testapprover01 receive archive notification
```

For a first POC, I would **manually trigger archive** by changing the old file’s status. For a more advanced version, you would automate “when newer version becomes live, find older live version with same Asset ID and archive it,” but that usually requires stronger metadata logic or Power Automate/API.

---

# 11. POC validation checklist

The POC is successful if all of these are true:

```text
[ ] testuser01 can upload a file to Working Drafts
[ ] testuser01 can set Lifecycle Status = Ready for Review
[ ] File automatically moves to 02_Review Queue
[ ] testapprover01 receives notification/task
[ ] testapprover01 can approve the asset
[ ] Approval updates metadata to Approved - Pending Distribution
[ ] File automatically moves to 03_Approved Pending Distribution
[ ] testuser01 receives pending-distribution notification/task
[ ] testuser01 can mark asset Live / Distributed
[ ] File automatically moves to 04_Current Distributed Assets
[ ] testuser01 and testapprover01 receive live notification
[ ] Archived / Superseded status moves file to 05_Archived Previous Versions
[ ] Workflow history shows who approved and when
[ ] Folder history shows file movement
[ ] Metadata search can find assets by Lifecycle Status
```

---

# 12. Important POC caveat: approval tag restriction

The cleanest governance model is:

```text
Only approvers can cause Approved - Pending Distribution to be applied.
```

For the POC, I would **not** rely on regular users manually selecting `Approved - Pending Distribution`.

Instead:

```text
testapprover01 approves workflow task
        ↓
Workflow automatically applies Approved - Pending Distribution metadata
        ↓
Workflow moves file to 03_Approved Pending Distribution
```

That is stronger than “trusting people not to click the wrong tag.”

This is the better technical design because the approval status is produced by the approval workflow, not by casual manual tagging.

---

# 13. Suggested naming convention for the POC

Use these exact names so the test is easy to understand.

## Metadata section

```text
Marketing Asset Lifecycle
```

## Status field

```text
Lifecycle Status
```

## Folders

```text
01_Working Drafts
02_Review Queue
03_Approved Pending Distribution
04_Current Distributed Assets
05_Archived Previous Versions
```

## Workflows

```text
MKTG POC - Ready for Review Routing
MKTG POC - Approval to Pending Distribution
MKTG POC - Distribution to Live
MKTG POC - Archive Superseded Asset
```

## Groups

```text
MKTG-POC-Contributors
MKTG-POC-Approvers
MKTG-POC-Distribution-Owners
```

---

# 14. Fallback if native Egnyte workflow is missing a piece

If your Egnyte plan/UI does not expose one of the required workflow actions, use **Microsoft Power Automate** as the fallback automation layer.

Egnyte has a Power Automate integration that supports workflows around Egnyte files when they are copied, created, deleted, or moved, and Egnyte’s Power Automate admin guide includes metadata operations such as setting metadata by file ID and listing metadata namespaces. ([Egnyte Helpdesk][8])

Use Power Automate for anything Egnyte native workflow cannot handle cleanly, especially:

```text
Dynamic date stamping
Finding previous live version by Asset ID
Archiving the old version automatically
Sending custom email/Teams notifications
Complex conditional logic
```

For the POC, though, I would keep the first test mostly native Egnyte:

```text
Metadata trigger
→ approval task
→ automated metadata update
→ automated file move
→ workflow/folder notifications
```

That proves the operating model without overengineering it.

[1]: https://helpdesk.egnyte.com/hc/en-us/articles/18313846897549-Workflow-Templates-Overview?utm_source=chatgpt.com "Workflow Templates Overview"
[2]: https://helpdesk.egnyte.com/hc/en-us/articles/360035813612-Using-Metadata-in-the-WebUI?utm_source=chatgpt.com "Using Metadata in the WebUI"
[3]: https://helpdesk.egnyte.com/hc/en-us/articles/31789193742733-Workflow-Triggers?utm_source=chatgpt.com "Workflow Triggers"
[4]: https://helpdesk.egnyte.com/hc/en-us/articles/35139103566605-Move-File-to-a-Folder-Using-Workflows?utm_source=chatgpt.com "Move File to a Folder Using Workflows"
[5]: https://helpdesk.egnyte.com/hc/en-us/articles/31785494102669-Web-UI-File-Conversion-and-Workflow-Triggers?utm_source=chatgpt.com "Web UI - File Conversion and Workflow Triggers"
[6]: https://helpdesk.egnyte.com/hc/en-us/articles/35143777415693-Adding-Metadata-to-a-File-Using-Workflows?utm_source=chatgpt.com "Adding Metadata to a File Using Workflows"
[7]: https://helpdesk.egnyte.com/hc/en-us/articles/201637744-Folder-Notifications?utm_source=chatgpt.com "Folder Notifications"
[8]: https://helpdesk.egnyte.com/hc/en-us/articles/360037571231-Egnyte-s-Microsoft-Power-Automate-Integration?utm_source=chatgpt.com "Egnyte's Microsoft Power Automate Integration"
