# invoice-tracker-automation
# Building an Invoice Approval & Two-Stage Payment System with Power Automate (No Code, Just Logic)

*How I replaced a manual, multi-team invoice process with four connected flows, a relational SharePoint structure, and zero new software spend.*

---

## The Problem

At MPL Ghana, part of the Tolaram Group, our logistics operation runs on third-party transporters delivering goods to multiple distributors. Here's the wrinkle that makes this harder than it sounds: **one external truck can deliver to several distributors in a single trip.** One vendor invoice. One vehicle. Multiple Delivery Orders, multiple distributors, one payment.

Before this project, that whole chain — submission, tracking, approval, payment — ran on email threads, WhatsApp follow-ups, and physical paperwork. Four different roles touched every invoice: a Transport Manager, a Tracking Manager, a Finance Controller, and a Payment Office. Nobody had a single source of truth for where an invoice stood. Payments were sometimes made twice, sometimes too early, sometimes nobody could find the original PDF.

I set out to fix this using only what we already had: Microsoft 365.

---

## The Constraint That Shaped Everything

No budget for new software. No developers on standby. Just Power Automate, SharePoint, Microsoft Forms, and Teams — tools already licensed and already familiar to the team.

That constraint turned out to be a feature, not a limitation. It forced the design to stay legible to non-developers, which matters a lot when the people maintaining it day to day are business users, not engineers.

---

## Architecture, Version 1: Get It Working

The first version was deliberately simple — one form per stakeholder, one flow connecting two steps, one SharePoint list as the single source of truth.

```
Transport Manager  →  Form 1  →  Flow 1  →  SharePoint row created
                                              ↓
Tracking Manager   →  Form 2  →  Flow 2  →  SharePoint row updated
                                              ↓
                              Finance Controller approves/rejects (email buttons)
                                              ↓
                         Payment Office notified  →  SharePoint row finalised
```

The primary key was the Delivery Order Number. Every invoice mapped to exactly one DO. Clean, simple, working.

Then reality intervened.

---

## The Restructure: When One Truck Serves Many Distributors

Once the first version was live, the actual shape of the business surfaced: **a single Freight Order (FO) can cover multiple Delivery Orders going to multiple distributors.** The Delivery Order Number couldn't be the primary key anymore — it wasn't unique enough up the chain. The FO Number, generated *after* the Transport Manager's submission by the Tracking Manager, was the real anchor.

This wasn't a tweak. It meant:

- **A new primary key**, generated downstream of the first form, not at the point of entry
- **A one-to-many data relationship** — one FO, many DOs, many distributors
- **A flat SharePoint list could no longer represent the data correctly**

### The relational fix

I split the single list into two:

| List | Purpose | Key |
|---|---|---|
| **FO Master** | One row per Freight Order — vendor, transporter, payment, approval data | FO Number (primary key, indexed, unique) |
| **DO Details** | One row per individual Delivery Order | FO Number (foreign key) |

To populate `DO Details`, the Transport Manager enters all DO numbers and distributor names as comma-separated text in Form 1 (`DO-001, DO-002, DO-003`). Flow 1 then does the unglamorous but essential work:

```
split(DO_Numbers_field, ',')        →  array of DO numbers
split(Distributor_Names_field, ',') →  array of distributor names

Apply to each DO in the array:
    Create a row in DO Details
    DO Number      = trim(current item)
    Distributor     = trim(array[LoopCounter])
    Increment LoopCounter
```

A simple integer counter, incremented on each loop iteration, kept the two arrays correctly paired by index. It's not glamorous engineering, but it's the kind of pragmatic solution that ships — and it scales to any number of DOs per FO without touching the flow again.

---

## The Two-Stage Payment Problem

The second major change came from how the vendor actually gets paid: **an initial payment before delivery, and a final payment after delivery is confirmed.** The first version of the system had no concept of payment staging at all.

I didn't want the Payment Office to have to remember to check back in days later. So the final payment trigger needed to be **event-driven**, not a manual follow-up.

### The design: a self-triggering link

When the Payment Office confirms the initial payment, Flow 3 sends the Transport Manager a confirmation email — and embeds a link to a fourth form, **Delivery Confirmation & Final Payment Request**. The Transport Manager doesn't touch this link until the delivery is actually done. When they do, they:

1. Confirm delivery occurred
2. Enter the final payment amount
3. Flag any deductions

Flow 4 picks this up, checks a condition (`Delivery Confirmation contains "Yes"`), and only then notifies the Payment Office for the final payment — copying in the Finance Controller and posting to a dedicated Teams channel.

```
Initial payment confirmed (Flow 3)
        ↓
Email to Transport Manager — includes Form 4 link
        ↓
   [Transport Manager waits until delivery happens]
        ↓
Transport Manager clicks link → confirms delivery → submits final amount
        ↓
Flow 4: Condition — was delivery actually confirmed?
        ↓ Yes                              ↓ No
Notify Payment Office, FC, TM       Notify TM — nothing triggered
(email + Teams)
```

This pattern — **using an email-embedded form link as a manual but gated trigger** — turned out to be a useful general technique. It puts control in the hands of the person who has the real-world information (did the truck actually arrive?), without requiring a polling mechanism or a manual check-in from the Payment Office.

---

## Notification Design: Email + Teams, Not Either/Or

Email is good for detail-rich, formal records. It's bad for "did anyone actually see this." Adding Microsoft Teams notifications to a dedicated `#Payment-Alerts` channel solved that gap — every payment milestone (FC approval, FC rejection, initial payment made, final payment triggered) now posts to a channel the whole finance and logistics team can glance at, without anyone needing to open their inbox.

Both channels carry the same core information; Teams just makes it ambient.

---

## What the Final System Looks Like

```
FLOW 1 — Invoice Submission
Transport Manager submits multi-DO invoice
→ Creates FO Master row + one DO Details row per DO
→ Emails Tracking Manager

FLOW 2 — FO Number & FC Approval
Tracking Manager generates FO Number
→ Writes FO Number back to FO Master AND every linked DO Details row
→ Sends structured approval email to Finance Controller (native Approve/Reject buttons)
→ Branches: Approved → notify Payment Office + Teams | Rejected → notify Transport Manager + Teams

FLOW 3 — Initial Payment
Payment Office confirms initial payment
→ Updates FO Master
→ Emails Finance Controller (confirmation) and Transport Manager (with Form 4 trigger link)
→ Posts to Teams

FLOW 4 — Delivery Confirmation & Final Payment
Transport Manager confirms delivery via the embedded link
→ Validates delivery was actually confirmed
→ Notifies Payment Office, Finance Controller, Transport Manager
→ Posts to Teams
```

Four flows. Four forms. Two SharePoint lists. One Teams channel. Five stakeholder roles, none of whom write a line of code, all of whom now see exactly where every invoice stands.

---

## Lessons That Generalise Beyond This Project

**1. SharePoint's "internal column name" will bite you if you ignore it.**
Display names and internal field names diverge the moment you rename a column after creation. Every filter query I wrote that silently returned zero rows traced back to this. Always check the actual internal name via the column's edit URL before writing an OData filter.

**2. File uploads from Microsoft Forms come back as JSON, not URLs.**
Forms returns an array of objects (`name`, `link`, `id`, `size`...) for any file upload question — even with a single file. The fix is a small but essential expression:
```
first(json(body('Get_response_details')?['FIELD_KEY']))?['link']
```
Skipping this is the single most common point of failure in these builds.

**3. Loops inside loops need explicit scoping, every time.**
A `Condition` or approval action added "near" a `For each` loop in the visual designer doesn't automatically belong to it. If the action references `items('For_each')` but actually lives inside a *different* `For_each_1`, the loop has nothing to iterate over and silently skips. Always check Code View when something inexplicably never fires.

**4. A primary key generated downstream breaks naive designs.**
If the entity that ends up being your unique identifier doesn't exist at the point of first data entry, your data model needs an interim key (we used Invoice Number) to bridge the gap until the real key arrives. Don't force the first form to invent an ID it has no business generating.

**5. Constraints are useful.**
Building this entirely inside the Microsoft 365 tools the team already had — no new procurement, no new training overhead — was the single biggest factor in actual adoption. The "best" tool is the one people will actually use.

---

## Stack

`Microsoft Power Automate` · `Microsoft Forms` · `SharePoint Online` · `Microsoft Teams` · `Office 365 Outlook` · `Approvals connector`

---

## Closing Thought

This project didn't require a single line of traditional code — it required treating Power Automate like a real system: thinking in data models, not just sequences of clicks, and designing for what happens when the business reality doesn't match the first draft of the architecture. The most valuable thing I built wasn't any individual flow — it was the relational structure underneath all of them, the part that made it possible to add complexity (multiple DOs, two-stage payments, Teams) without starting over.

If you're building something similar on Microsoft 365 and hitting walls with file uploads, loop scoping, or multi-entity primary keys — happy to compare notes.

---

*Mega Anokye — Business Analyst & Process Automation Specialist, Tolaram Group, MPL Ghana*
*Connect on [LinkedIn](https://linkedin.com/in/mega-anokye)*
