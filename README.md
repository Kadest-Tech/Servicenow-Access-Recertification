![Access Recertification banner](assets/Banner.png)

# Access Recertification

A ServiceNow scoped application that identifies access people should no longer have and sends that access to a real person for review and sign-off.

## Project Overview

| Component | Count |
|---|---:|
| Custom tables | 3 |
| Automations | 2 |
| Access Control Lists | 23 |
| Reports | 4 |

## Contents

1. [The problem this solves](#01-the-problem-this-solves)
2. [How it works in real life](#02-how-it-works-in-real-life)
3. [How the whole thing flows](#03-how-the-whole-thing-flows)
4. [The three tables](#04-the-three-tables)
5. [What was built, and how](#05-what-was-built-and-how)
6. [How the security works](#06-how-the-security-works)
7. [What testing actually found](#07-what-testing-actually-found)
8. [What was deliberately left out](#08-what-was-deliberately-left-out)
9. [What would change at real company scale](#09-what-would-change-at-real-company-scale)
10. [Install it yourself](#10-install-it-yourself)

---

## 01. The problem this solves

Companies hand out access to software every single day. Someone joins the billing team, so they get into the billing system. That part works fine.

The part that does not work is taking access away.

People switch teams. Projects end. Contractors leave. But the access they were given usually just stays there. Nobody removes it, because removing it is nobody's job.

Do that for a few years across a few hundred people and you get a real mess:

> [!WARNING]
> A paralegal moves from lawsuits to compliance—and still has full editing rights to old case files. A contractor's admin account is still live eight months after the contract ended. Nobody knows either of these things, because nobody ever checks.

This is the exact thing auditors ask about. Under rules like **SOX**, **ISO 27001**, and **NIST 800-53**, a company has to answer two questions:

1. **Who can get into what?**
2. **When did a real person last confirm that was still OK?**

Most tools handle giving access well. They handle *checking* it badly. But the checking is where the actual safety comes from.

**This app is the checking.**

---

## 02. How it works in real life

The demo data is set up like a law firm. The firm runs five programs:

| Program | Used for |
|---|---|
| iManage | Storing case documents |
| NetDocuments | Storing case documents |
| Aderant | Tracking time and billing |
| Elite | Tracking time and billing |
| Active Directory | Logging into everything else |

Three kinds of people use the app, and each one can do different things:

| Who | What they can do | What they cannot do |
|---|---|---|
| **Requester** | Ask for access. See their own access. | See anyone else's access. |
| **Reviewer** | Approve or remove access assigned to them. | Touch reviews assigned to someone else. |
| **Access Manager** | See everything. Track progress. | Make any approve/remove decision. |

> [!IMPORTANT]
> The last row is the point. The person **watching** the review cannot make the decision. If one person did both, they could approve their own work and call it done. Splitting the two responsibilities is called **separation of duties**, and it is a core audit rule.

---

## 03. How the whole thing flows

Read this as two separate stories running side by side.

| Story 1 — Someone asks for access | Story 2 — Time to check old access |
|---|---|
| 1. A user fills out a short form in the portal. | 1. A scheduled job wakes up on the first of the month. |
| 2. The app looks up their manager automatically—nothing is hardcoded. | 2. It checks whether a review round is already running. |
| 3. The manager gets an approval request. | 3. If yes, it stops. If no, it starts a new round. |
| 4. If approved, access turns on and the date is stamped. | 4. For every active access record, it creates a review task and sends it to that person's manager. |
| 5. If rejected, access is marked revoked and no date is stamped. | 5. The manager chooses to keep or remove it. If removed, the original access record automatically changes to revoked. |

---

## 04. The three tables

| Table | Plain-English meaning | Example |
|---|---|---|
| **Access Item** | One row for one person's access to one program. | Fred can edit in iManage. |
| **Review Campaign** | One round of checking, like a folder holding a batch of reviews. | Contains the reviews for that round. |
| **Access Review** | One decision about one access record. | Should Fred still be able to edit in iManage? |

> [!NOTE]
> **Access Review is built on top of ServiceNow's built-in Task table.** Every review automatically receives an owner, status, work notes, and an approval section without rebuilding those features. Reviews are real work items someone must finish, not log entries sitting in a database.

---

## 05. What was built, and how

| What it does | How it was built |
|---|---|
| Users request access themselves | Service Catalog record producer with 4 required fields |
| Finds the right approver | Flow looks up the requester's manager live |
| Handles yes and no | Flow branches—approval stamps the date; rejection does not |
| Starts review rounds on schedule | Scheduled flow with a safety check built in |
| Assigns each review | Looks up each person's manager one at a time |
| Users see only their own records | ACL script compares the record to the logged-in user |
| Locks one specific field | Separate ACL on the “why did you remove it” field |
| Watchers cannot be deciders | Managers receive read access but no write access |
| Form reacts as it is used | UI Policy shows a required field only when needed |
| Data stays correct | Business rule updates the original record automatically |
| Shows the big picture | Four reports on access and review results |

---

## 06. How the security works

Full details are documented in [`docs/ACL-Matrix.md`](docs/ACL-Matrix.md).

Three ideas run through the whole design:

### 1. Anyone can ask. Almost nobody can see.

There is no restriction on submitting a request. Any user can ask for anything.

There **is** a restriction on viewing records. A script checks whether the record belongs to the logged-in user before showing it.

Why? Asking is not dangerous. Manager approval is the actual gate. Locking down the request form would break the form for the same people it was built to serve.

### 2. Reviewers only touch their own work

Both reading and editing a review are checked against the person to whom it is assigned. Beth cannot open Bud's reviews.

The records are not merely hidden—they are actually blocked.

### 3. Roles come from groups, never handed out one by one

Nobody receives a role directly. A user joins a group, and the group supplies the role.

> [!TIP]
> This makes access traceable. You can answer “Why does this person have this?” with “Because they are in that group.” A one-off grant to a single person leaves no clear trail—and untracked grants are exactly what this application exists to catch.

---

## 07. What testing actually found

Real testing found three real problems. All three were fixed. How a bug is found can say more than the bug itself.

### Problem 1 — The security looked broken. It was not.

Logging in as a normal user showed every record in the system, not just that user's records. The security rule was supposed to prevent this.

The rule was fine. The test account had a leftover `admin` grant that nobody remembered assigning. Administrators bypass record-level security by design, so the rule never ran.

Removing that single grant automatically removed **120 other roles** that came with it. The account dropped to two roles, both inherited from group membership. Retesting immediately produced the correct result.

> [!CAUTION]
> This is the application's own point, proved on the application itself: a forgotten permission sat unnoticed until something forced a check. That is precisely what access recertification exists to find.

### Problem 2 — Real users could not use the request form

The first problem concealed a second one. The user's earlier submission worked only because that user was an administrator.

The rules allowed administrators and access managers to create records, but not regular users. The request form built for regular users did not work for them.

**Fix:** Added a create rule for the requester role.

### Problem 3 — The documentation was incomplete

One valid security rule existed in the system but was missing from the written matrix. Reviewers do need to see access details to make a decision, but the rule had not been documented.

**Fix:** Added the rule to the ACL matrix.

---

## 08. What was deliberately left out

Every project makes simplifications. These are documented openly rather than hidden.

### Review rounds are quarterly, but the job runs monthly

ServiceNow's scheduler did not offer an “every three months” option, and the script action needed to calculate that timing was not available on this instance.

The job therefore runs monthly and begins with one question:

> **Is a review round already open?**

If yes, it stops immediately and creates nothing.

The result is the same whether it runs monthly, weekly, or twice in a row: it never creates duplicate work. The actual timing is controlled by when someone closes the current round.

Two execution logs in `/screenshots/` show both outcomes: one run that created the work and one that correctly refused.

### Other simplifications

- The campaign due date is not filled automatically. In production, this would require a small script.
- `granted_by` is set when the request is submitted, so it effectively means “requested by” until approval finishes. The field name could be clearer.
- “Approval for” does not display on the approval screen. This is standard ServiceNow behavior for custom tables and has no functional impact.
- There is no dashboard—only four separate reports. ServiceNow's newer dashboard tool uses a different storage system and would require the reports to be rebuilt.
- Two permission records were created automatically by the platform rather than manually. They are legitimate and included in the update set.

### The application installs empty

ServiceNow update sets carry **configuration, not data**. Tables, fields, flows, and rules travel. Actual records do not.

Importing the update set gives you a working but empty application. Add your own access records or submit them through the catalog form.

---

## 09. What would change at real company scale

| Now | At scale |
|---|---|
| Access records entered by hand | Pulled in automatically from each program |
| Every active record gets reviewed | Only high-risk access gets reviewed each round |
| Removal updates the local record | Removal actually disables the account in the real system |
| Reviews completed one at a time | One screen shows all of a manager's reviews together |
| No reminders | Nudges for late reviews and automatic removal when a round expires |

---

## 10. Install it yourself

1. In ServiceNow, go to **System Update Sets → Retrieved Update Sets → Import Update Set from XML**.
2. Upload `/update-set/access-recertification_v1.xml`.
3. Select **Preview**, resolve anything flagged, and then select **Commit**.
4. Create the three groups and assign the four roles.
5. Add access records or submit one through the catalog form.

---

---

**Built on a ServiceNow Personal Developer Instance — Australia release**
