# Zoho CRM quote approval gate based on the effective discount rate (Deluge sample)

> **Prototype, not client work.** This is a prototype built and tested in a Zoho CRM trial org (JPY, one user). It is not a delivered client project.
> **All test numbers are fictitious.** The amounts and rates used in the examples and test fixtures are made-up values.

## Purpose

This sample is based on the official Zoho Kaizen post
[Kaizen #230 "Smart Discount-Based Quote Approvals Using CRM Functions and Approval Process"](https://help.zoho.com/portal/en/community/topic/kaizen-230-smart-discount-based-quote-approvals-using-crm-functions-and-approval-process).
The prototype changes one thing in the Kaizen #230 formula (the version used here is called "candidate D"). The 10% / 20% thresholds are the example values from Kaizen #230, and they were tested in the trial org.

The problem it addresses: discounts compound by multiplication, not addition. Two 10% discounts give an effective discount of 19%, not 20% (10% + 10% ≠ 20%). If you look only at the header Discount field, you can put a quote in the wrong approval tier. This sample computes one effective rate per quote and sets the approval tier from that rate.

## Formula (candidate D) and tiers

```
effective rate = (Discount − min(Adjustment, 0)) / (Sub_Total − Tax) × 100
```

- `Tax` is **not** the header field. It is the sum of the `Tax` value on each row of `Product_Details` (the quote's line items).
- The 10% and 20% thresholds are constants inside the function.
- The rate is rounded to 2 decimal places (`round(2)`). The same rounded value is used both for the tier decision and for the saved field.

| Effective rate | Approval_Status | Approval_Level |
|---|---|---|
| rate < 10% | Auto Approved | None - Auto Approved |
| 10% ≤ rate < 20% | Pending Manager Approval | Sales Manager |
| rate ≥ 20% | Pending VP Approval | VP Sales |
| cannot be evaluated (see fail-closed cases) | Needs Manual Review | Manual Review |

**Scope of candidate D:** The rate does not include line-item discounts (Sub_Total is already net of line discounts, and the numerator uses only the header Discount and a negative Adjustment). The current layout in this org has no line-item discount field, so orgs with line-item discounts are unverified in this prototype.

## Fields to add to the Quotes module

| API name | Type | Values |
|---|---|---|
| `Discount_Percentage` | Percent | Effective rate, 2 decimal places |
| `Approval_Status` | Picklist (7 values) | Not Evaluated (default) / Auto Approved / Pending Manager Approval / Pending VP Approval / Needs Manual Review / Approved / Rejected |
| `Approval_Level` | Single Line | "None - Auto Approved" / "Sales Manager" / "VP Sales" / "Manual Review" |

## Files

| File | What it is |
|---|---|
| `set_quote_approval_level.deluge` | Main function (standalone, v7). Computes the rate and writes the three fields. |
| `set_quote_approval_level_wf.deluge` | Automation wrapper (one line) so a workflow rule can call the main function. |
| `quote_approval_review_digest.deluge` | Daily digest function (standalone). Emails the admin a list of quotes that became Needs Manual Review. |
| `quote_approval_review_digest_sched.deluge` | Schedule wrapper (one line) so a daily schedule can run the digest. |

Each `.deluge` file holds only the function body. Add the function declaration in the Zoho CRM function editor. The code comments in the two main functions are in Japanese.

## How to register the functions (overview)

1. Add the three fields above to the Quotes module.
2. Create a **standalone** function `set_quote_approval_level` with one argument `id` (String), and paste in `set_quote_approval_level.deluge`. Use a lowercase snake_case name for the declaration; with mixed-case names, other functions could not find it ("Not able to find").
3. Create an **automation** function with one argument `quoteId` (String), map `quoteId` to the quote's record ID, and paste in `set_quote_approval_level_wf.deluge`. A workflow rule cannot call a standalone function directly, so this wrapper is needed.
4. Create a **workflow rule** on Quotes that calls the automation function. While testing, add a condition that limits the rule to test quotes (the prototype used a Subject prefix that matches only test records).
5. For the digest, create a **standalone** function `quote_approval_review_digest` (no arguments) and paste in `quote_approval_review_digest.deluge`. Set `ADMIN_EMAIL` to your admin address (the sample uses `admin@example.com`).
6. Create a **schedule** function, paste in `quote_approval_review_digest_sched.deluge`, and schedule it to run once a day.

The digest assumes the org time zone is Asia/Tokyo. If your org uses another time zone, change `"Asia/Tokyo"` and `TZ_OFFSET` in the digest function.

## Fail-closed cases

When the function cannot compute a trustworthy rate, it does not compute or guess. It writes Approval_Status = **Needs Manual Review** and Approval_Level = **Manual Review**, and does not write `Discount_Percentage`. This happens when:

- the denominator (Sub_Total − Tax) is 0 or less
- the rate is outside the range [0, 100]
- a required value is missing (`Sub_Total` is not in the response, `Product_Details` is missing or null, or a line item has no `Tax` value). A missing tax is never treated as 0.
- **the quote has zero line items** (no rate is computed)
- the function's own steps fail: the record update returns an error, the read-back after the write does not match, or a re-computation from the read-back values gives a different rate (conflict)
- exception (v7): if the write succeeds and the written status is a Pending status (**Pending Manager Approval** or **Pending VP Approval**), the record is submitted to the approval process and locked, so the read-back returns no record. This specific read-back failure is treated as "submitted", not as fail-closed, and Needs Manual Review is not written. If the read-back succeeds and the values do not match, it is still treated as a mismatch.

If the quote record itself cannot be fetched, there is no record to write to, so the function stops and records the reason only in the execution log.

State guard:

- If the rate has not changed and the quote is already **Approved**, the function does not overwrite it.
- After a **Rejected** result, the quote is re-evaluated only if the rate or the tier has changed.
- Entry guard (v7): if the initial fetch at the start returns no fields at all because the quote is in approval processing (approval lock), the function exits without writing anything. A record in approval is never overwritten.

## Daily digest instead of per-record emails

The main function sends no email. Quotes that end in Needs Manual Review are collected by `quote_approval_review_digest`, which runs daily at 09:00 (Asia/Tokyo) and sends the admin at most one email per day. If there are no quotes to review, it sends nothing.

Why: on 2026-11-10, the first version (v1) failed on every record because the header `Tax` key was not in the response, and it sent 12 failure emails between 21:25 and 21:38. Notifications were then moved into this daily digest.

## Approval process (designed separately, verified in the prototype org)

The approval process must be set up separately in Zoho CRM; it is **not** included in the Deluge code of this sample. On 2026-11-13, the process "Quote Discount Approval" was created and activated in the prototype org, and Rule 1 and Rule 2 were verified end to end through function v7 (Rule 3 is configured, but a standalone submission was not measured; see below). The design is:

- Rule 1: Approval_Status = Pending Manager Approval → the submitter's direct manager (Reporting Hierarchy)
- Rule 2: Approval_Status = Pending VP Approval → two levels up
- Rule 3: Approval_Status = Needs Manual Review → Record Owner's Manager (same approver as Rule 1)
- On approval, a field update action writes **Approved** back to Approval_Status. On rejection, it writes **Rejected**.

In the prototype org, a specific user stands in as the approver. In a production org, the approver is resolved through the reporting hierarchy.

Measured on 2026-11-13: a process with this design ("Quote Discount Approval", Quotes, Create or Edit; Rule 1 → Record Owner's Manager, Rule 2 → a specific user, Rule 3 → Record Owner's Manager) was created and activated in the prototype org. An earlier run on the same day did not submit edited records; v7 fixed this. That evening the flow was re-verified with function v7, which passes `trigger: ["approval"]` explicitly to `updateRecord`, treats a read-back that fails because of the approval lock as "submitted" instead of an error, and adds a state guard that skips records it cannot read while they are in approval. The Manager tier was auto-submitted, approved, and ended as **Approved**; the VP tier was auto-submitted, rejected, and ended as **Rejected**. In both cases Approval_Level stayed in place, and no overwrite occurred when the function ran again after the decision. Rule 3 is configured with the same approver as Rule 1, but a quote that is Needs Manual Review from the start was not submitted and measured on its own, so Rule 3 is not claimed as verified end to end.

## What was verified

All tests ran in a Zoho CRM trial org (JPY, one user). All numbers are fictitious.

- **Rate and tier, including boundaries.** The registered function matched the expected result for every fixture (measured with v5 on 2026-11-11 to 12; the current registered version is v7, with the rate calculation unchanged):
  15.00% → Manager / 0.00% → Auto / 9.00% → Auto / **10.00% → Manager (boundary)** / 11.00% → Manager / 0.00% → Auto / **20.00% → VP (boundary)** / **19.99% → Manager (boundary)** / 15.00% → Manager / multi-line (2 rows, tax total 15,000) 1.00% → Auto / workflow trigger test 0.00% → Auto.
- **Tax summed across multiple line items.** Row tax 10,000 + 5,000 = 15,000; Sub_Total 165,000; Discount 1,500 → rate 1.00%. The earlier v4 returned an incorrect 0.97% for this case: it read the two line items as a single row and counted only the first row's tax.
- **Workflow.** The workflow rule "Set Quote Approval Level" was enabled with a condition that limited it to test records.
- **Digest aggregation.** A manual run combined 8 quotes into 1 email. The first scheduled run (2026-11-12 09:00) was checked in the execution log: Status 200, Duration 300 ms, Credits 1, 0 quotes to review, so no email was sent.
- **Field permissions.** With the three fields set to Read Only for the Standard profile, the function still wrote to them successfully.

## Practical note

Expected impact when published: there is no direct way to measure this repository's reach. Any contribution is indirect only, through references from the blog and LinkedIn.
