# Propose New Business Term (with Approval) – Collibra Workflow

| | |
|---|---|
| **Process ID** | `intakeBusinessTermApproval` (separate from `intakeBusinessTerm`, so both can be deployed side by side) |
| **Process name** | Propose New Business Term (with Approval) |
| **File** | `intakeBusinessTermApproval.bpmn` |
| **Start type** | Global |
| **Approver** | **Business Steward** of the Glossary domain the requester selects |
| **Status** | Not yet deployed or tested. See [Section 8](#8-items-to-verify-on-first-deployment) |

---

## 1. What it does

1. A requester proposes a Business Term or Acronym and picks the target **Glossary** from a drop-down.
2. The asset is created right away with status **Candidate**. This lets the steward review the real asset in Collibra.
3. A review task goes to the **Business Steward of that Glossary**, with reviewer guidance included.
4. The steward either:
   - **Approves**: status changes to **Accepted**, and the proposer is notified (this notification can be turned off); or
   - **Rejects**: a reason is **required**. Status changes to **Rejected**, and the reason is **emailed to the proposer**.
5. Every action is **audited** (see [Section 5](#5-audit-trail)).

---

## 2. Process flow

```
REQUESTER
 (Start) ─► [Get Glossary Domains] ─► [Propose Business Term] ─► [Create Candidate Asset + Resolve Steward]
                                                                              │
BUSINESS STEWARD                                                              ▼
                               ┌──────────────────────────────► [Review Proposed Glossary Asset]
                               │                                              │
                               │                                              ▼
                               │                                     [Validate Decision]
                               │                                              │
                               └──── invalid (e.g. no reason) ─────── <Decision?>
                                                                        │         │
                                                                 approve│         │reject
                                                                        ▼         ▼
                                                 [Set Accepted, Audit, Notify]   [Set Rejected, Audit, Email reason]
                                                                        │         │
                                                                   (Approved)  (Rejected)
```

| Step | ID | Type | Purpose |
|---|---|---|---|
| Get Glossary Domains | `getGlossaryDomains` | Script | Builds the Glossary drop-down (domain type `…010001`). |
| Propose Business Term | `proposeTerm` | User task (initiator) | Proposal form. Also holds the hidden configuration. |
| Create Candidate Asset + Resolve Business Steward | `createCandidateAsset` | Script | Finds the approvers, creates the asset as Candidate, adds attributes and relations, writes the **PROPOSED** audit entry, and sets the variables the review task needs. |
| Review Proposed Glossary Asset | `reviewTerm` | User task (Business Steward) | Shows the guidance, checklist and proposed values as readable text at the top, then collects the decision and comments. |
| Validate Decision | `validateDecision` | Script | Records who reviewed. Enforces "reason required on reject" and, optionally, blocks self-approval. |
| Decision? | `decisionGateway` | Exclusive gateway | `approve` / `reject` / `invalid` (invalid loops back to the review task with a message). |
| Approve | `approveAsset` | Script | Sets status to Accepted, writes the **APPROVED** audit entry, and notifies the proposer. |
| Reject | `rejectAsset` | Script | Sets status to Rejected, writes the **REJECTED** audit entry, and emails the reason to the proposer. |

---

## 3. How the approver is found

The approver comes from a Collibra user expression built at runtime from the selected Glossary:

```
{role(Business Steward;<Community name>;<Glossary domain name>)}
```

- Collibra documents this syntax for roles on a specific domain.
- If **no one** holds the role, the task goes to `fallbackApproverExpression`. The default is `{group(Data Governance Council)}`. **Change this to a group that exists in your environment.**
- If the fallback also finds no users, the proposal is **not** created. The requester sees an error, and the log explains why.
- Collibra's standard task notification emails the steward(s) about the new task, subject to each user's notification settings.
- Any one of the candidate stewards can claim and complete the task.

> **Note for bailine-fed:** a sample Glossary asset (`OPM Business Glossary`) showed **no responsibilities assigned**. Before go-live, check that the Glossaries you plan to use actually have a Business Steward. Otherwise every proposal will go to the fallback group.

---

## 4. Forms

### 4.1 Proposal form (`proposeTerm`)

This is the same as the version without approval: **Glossary** (drop-down), **Name**, **Type** (Business Term / Acronym), **Proposed Definition**, **Related Assets**, **Reason for proposal**, and a **Propose** button.

### 4.2 Review form (`reviewTerm`)

The form reads from top to bottom: guidance, then the proposed values, then the decision. The guidance and proposed values are both in the **task description** at the top of the form. There they display as normal, readable text and can't be edited, so no grayed-out fields are needed.

**Layout of the task description**

```
<Proposer> has proposed the <Type> '<Name>' for the glossary '<Glossary>'.

Before deciding, please review the proposed values below and:
1. Verify the name is correct, correctly spelled and follows naming standards.
2. Verify the definition is clear, accurate, complete and not circular.
3. Confirm it is not a duplicate of an existing term or acronym in the glossary.
4. Confirm it belongs in this glossary and the type is correct.
5. Check that any related assets are appropriate.

Approve sets the status to Accepted.

Reject sets the status to Rejected, requires a reason, and emails that reason to the proposer.

PROPOSED VALUES (read-only):
Name: <name>
Type: Business Term | Acronym
Glossary: <glossary name>
Proposed by: <full name> (<username>)
Definition: <definition, plain text, or "(none provided)">
Reason for proposal: <reason, or "(none provided)">
Related assets: <asset names, or "(none)">
```

- **Glossary name only:** only the glossary's name is shown, not its parent community.
- **Formatting:** with the default `reviewDescriptionFormat=html`, paragraphs are separated by a blank line, each checklist item and value is on its own line, and labels (Approve, Reject, PROPOSED VALUES and the value names) are **bold**. If raw `<b>` or `<br/>` tags appear instead, set it to `text`, which uses plain line breaks.
- **"ACTION REQUIRED" message:** if the steward submits something invalid (for example, rejects without a reason), the task comes back with **ACTION REQUIRED: ...** as its own paragraph at the top.
- **Length limits:** Collibra stores the task description in a 4,000-character field, so long values are shortened. Definition is capped at 1,000 characters, Reason at 700 and Related assets at 400. A shortened value ends with *"... (truncated - full text is on the asset)"*.

**3. Decision**

| Field | Required | Content |
|---|---|---|
| **Decision** | Yes | Approve / Reject |
| **Comments (required to reject)** | Only when rejecting | Emailed to the proposer on rejection and recorded in the audit trail. |
| **Submit decision** | – | Button |

**To change the guidance**, edit the description text in the `createCandidateAsset` script (the `checklist` list and the `description` string). The whole description (guidance, checklist and proposed values) is built in the `createCandidateAsset` script and stored in the `reviewDescription` variable. The `reviewTerm` documentation is just `${reviewValidationMessage}${reviewDescription}`.

> **Note:** the Candidate asset is still created before review, so it exists in the glossary and appears in searches. The clickable link was removed because it didn't lead to the asset in testing.

---

## 5. Audit trail

| Layer | What's recorded | Where to see it |
|---|---|---|
| **Asset comments** | One entry per action, prefixed `[Workflow audit]`: **PROPOSED** (who, when, glossary, type, who it was routed to), **APPROVED** or **REJECTED** (who, when, status change, reason or comments, notification result). | Comments on the asset |
| **Asset history** | Creation, attribute and relation changes, and status changes Candidate → Accepted / Rejected. | Asset > History |
| **Workflow history** | Task assignment, the user who completed each task, and process variables such as `reviewerUserName`, `reviewDecision`, `reviewComment` and `decisionDate`. | Workflow / task history (admin) |
| **Application log** | Every step, tagged `[intakeBusinessTermApproval]`, including `AUDIT` lines and any failures. | Collibra Console > Logs > `dgc.log` |
| **Notifications** | Approval and rejection messages linked to the asset. | Notification Center / email |

**Rejected assets are kept, not deleted**, so the audit history is preserved.

**If comments can't be written** (for example, if the comment API differs in your version), the audit entry is written as a **Note** attribute on the asset instead. You can change this with `auditFallbackAttributeTypeUuid`.

---

## 6. Configuration

All options are **hidden fields on the `proposeTerm` user task**. Edit their `default` values in the BPMN or in Workflow Designer.

| Field ID | Default | Purpose |
|---|---|---|
| `businessStewardRoleName` | `Business Steward` | Resource role that approves. Must match the role name exactly. |
| `fallbackApproverExpression` | `{group(Data Governance Council)}` | **Must be changed** to a real group or user expression, e.g. `{group(Glossary Approvers)}` or `{user(jdoe)}`. |
| `candidateStatusUuid` | `…5008` | Status when the asset is created (Candidate). Verified in bailine-fed. |
| `acceptedStatusUuid` | `…5009` | Status on approval (Accepted). Verified in bailine-fed. |
| `rejectedStatusUuid` | `…5010` | Status on rejection (Rejected). Verified in bailine-fed. |
| `preventSelfApproval` | `false` | Set to `true` to stop a steward approving their own proposal. They'd be asked to reassign the task instead. |
| `notifyProposerOnApproval` | `true` | Send an approval notification. Rejection emails are always sent. |
| `reviewDescriptionFormat` | `html` | How the review guidance and proposed values are formatted. `html` gives paragraph spacing, line breaks and bold labels. `text` uses plain line breaks, for versions that show raw HTML tags. |
| `auditFallbackAttributeTypeUuid` | `…3116` (Note) | Attribute used for audit entries if comments fail. |
| `usesRelationTypeUuid` / `definitionAttributeTypeUuid` / `noteAttributeTypeUuid` | Same as the non-approval version | Relation and attribute types. |

The Glossary domain type (`…010001`) and the fallback domain (`…6013`) are set at the top of the `getGlossaryDomains` script.

---

## 7. Deployment steps

1. **Before deploying, set the fallback approver.** Change `fallbackApproverExpression` to a group that exists.
2. **Upload the file.** In **Settings > Workflows > Definitions**, click **Upload a file** and choose `intakeBusinessTermApproval.bpmn`. Its process ID is new, so the existing workflow isn't affected.
   - The file is **744 lines** and ends at `</definitions>` with no trailing blank line.
3. **Configure the definition:**

   | Setting | Value | Notes |
   |---|---|---|
   | **Enabled** | **On** | **Required.** A newly uploaded definition is **disabled** by default and can't be started, even for testing, until you enable it. |
   | Applies to | Global | |
   | **Applies to > Show in Global Create** | **Checked** | **Required.** Without it, the workflow doesn't appear in the global Create menu, and users can't start it. |
   | Start events | User | |
   | **Start roles** | **Remove *Sysadmin*** | By default only Sysadmins can start the workflow. |
   | **Any user can start the workflow** | **Checked** | Lets every user start it. If only certain users should propose terms, leave this unchecked and add their role(s) under Start roles instead. |
   | Stop roles | Admin / Sysadmin | |

   **Optional:** disable the non-approval version so users don't pick the wrong one.
4. **Assign stewards.** Give each target Glossary (or its community) a **Business Steward**.
5. **Check email and notifications.** Make sure outgoing email is configured for the environment, and that proposers haven't turned off email notifications.
6. **Test** using the scenarios in the next section.

---

## 8. Items to verify on first deployment

Some Java API calls used here aren't fully confirmed for your Collibra version. Each is either wrapped so a failure doesn't stop the workflow, or will show up clearly in `dgc.log`:

| Item | Used for | If it fails |
|---|---|---|
| `notificationApi.send { … }` | Rejection and approval emails | Logged. The audit entry records `Rejection email … FAILED`. The workflow still completes. On older versions without `notificationApi`, email needs the legacy `mail.sendMails` bean plus a Velocity template. |
| `commentApi.addComment(AddCommentRequest…baseResourceId / baseResourceDiscriminator)` | Audit comments | Falls back to a Note attribute and logs the error. If the **import** of `AddCommentRequest` doesn't exist in your version, the script won't compile. Remove the import and the `try` block in that case. |
| `flowable:taskCompleterVariableName="reviewerUserName"` | Records who completed the review | Falls back to `users.getCurrent()`. |
| `userApi.getUserByUsername` | Friendly names in audit entries and emails | Falls back to the username. |
| `{role(Role;Community;Domain)}` | Finding the Business Steward | Glossary or community names containing `;`, `(` or `)` may break the expression. Rename them or switch to a responsibility lookup. |
| Required attributes | bailine-fed marks **Definition** and **Descriptive Example** as required for Business Term | API creation may bypass this check, but stewards should confirm these are filled before approving. Consider adding a *Descriptive Example* field to the form. |

---

## 9. Test scenarios

| # | Scenario | Expected result |
|---|---|---|
| 1 | Propose into a Glossary **with** a Business Steward | Candidate asset created; PROPOSED comment added; task appears for the steward. |
| 2 | Propose into a Glossary **without** a Business Steward | Task goes to the fallback group; the PROPOSED comment names the fallback. |
| 3 | Fallback group empty or invalid | Proposal fails with a clear log message; no asset created. |
| 4 | Steward chooses **Reject** with an empty reason | Task comes back with the "reason is required" message; nothing changes. |
| 5 | Steward **rejects** with a reason | Status Rejected; REJECTED comment includes the reason; proposer gets an email with the reason. |
| 6 | Steward **approves** | Status Accepted; APPROVED comment added; proposer notified. |
| 7 | `preventSelfApproval=true`, and the steward is also the proposer | Approval blocked with a message; the task must be reassigned. |
| 8 | Run 1–6 as a **non-admin** proposer and a **non-admin** steward | Same results. Confirm permissions to start the workflow, complete the task and change the asset's status. |

---

## 10. Known limitations and options

- **The asset exists while under review**, as Candidate. If you'd rather create it only after approval, the create step can move after the Approve branch. The steward would then review the form data instead of a real asset.
- **No reminders or escalation** if the steward doesn't act. A timer boundary event on `reviewTerm` could add reminders or escalate to the fallback group.
- **One approver.** Any one Business Steward can decide. Voting or multi-approver setups would need Collibra's voting subprocess.
- **A rejected proposal can't be resubmitted** in the same run. The requester starts a new proposal.

---

## 11. Change log

| Date | Change |
|---|---|
| 2026-09-29 | First version: Business Steward approval, required rejection reason emailed to the proposer, audit trail. |
| 2026-09-29 | Removed the trailing blank line from the BPMN (now 695 lines). Deployment steps updated to match the tested setup: Enabled, Show in Global Create, Sysadmin removed from Start roles, Any user can start the workflow. |
| 2026-09-29 | Simplified the review form: removed the read-only *Action required*, *Reviewer guidance*, *Proposed asset*, *Name*, *Type*, *Glossary*, *Proposed by*, *Proposed definition* and *Reason for proposal* fields. The summary, checklist and any validation message are now in the task description. Added a clickable asset link to the Comments help text. BPMN is now 674 lines. |
| 2026-09-29 | Review form reworked after testing. Removed the clickable asset link, which didn't lead to the asset. Brought back read-only (grayed) fields for the proposed values: Name, Type, Glossary, Proposed By, Definition, Reason, Related Assets. They appear under the guidance, which ends with "THESE ARE THE PROPOSED VALUES (read-only):", and before the Decision. Definition and reason are now single-line fields so they display grayed out. Removed the asset link from the approval and rejection emails. BPMN is now 701 lines. |
| 2026-09-29 | Proposed values moved from grayed-out fields into the review task description, so they're easier to read and still read-only. Long values are shortened to stay within the 4,000-character description limit. Added the `reviewDescriptionFormat` option (text or html). BPMN is now 723 lines. |
| 2026-09-29 | Review guidance formatting: paragraph breaks before "Before deciding", "Approve", "Reject" and the proposed values; checklist items and proposed values each on their own line; bold labels. The glossary is shown by name only, without its parent community. The description is now built in the script, and `reviewDescriptionFormat` defaults to `html`. BPMN is now 744 lines. |
