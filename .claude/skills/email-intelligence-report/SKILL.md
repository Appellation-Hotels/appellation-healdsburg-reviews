---
name: email-intelligence-report
description: Audit a mailbox to find what the org actually does day to day and which of it can be automated. Produces the Appellation Email Intelligence Report — executive summary, recurring inbound/outbound patterns, dominant themes, people map, workflows with decision points, decision sequences, ranked pain points with automation opportunities, and timing patterns. Use when asked to "run the email analyzer", "analyse someone's mailbox", "find automation opportunities in email", "what does X spend their time on", "email intelligence report", or to onboard a new person or department to the AI programme. Rebuild of the original Trellis email analyzer at appellation.th1.ai/email-analyzer.
---

# Email Intelligence Report

Analyse one mailbox, infer how the organisation actually runs, and rank what can be
handed to a machine. The output is a single self-contained HTML report.

The reader is usually an executive or an automation vendor deciding *where to start*.
So the deliverable is a prioritised argument, not a description of an inbox.

## Before you start: whose mailbox, and do you have access?

This is the step that actually gates the job.

**Your own mailbox** — nothing to arrange. Omit `mailboxOwnerEmail`.

**Someone else's** — you need one of these in place first, and neither can be
granted from inside this session:

| Route | What it gives | Who sets it up |
|---|---|---|
| Delegate access to their mailbox | Full read via `mailboxOwnerEmail` | The mailbox owner, in Outlook |
| `Mail.Read.Shared` on the connector | Read across mailboxes you're delegated | A Microsoft 365 admin |

Confirm access before promising a report. Run the smallest possible probe:

```
outlook_email_search(mailboxOwnerEmail: "<them>@appellationhotels.com", limit: 1)
```

If it returns nothing or errors on permissions, **stop and say so**. Do not
substitute your own mailbox and do not infer their work from your copies of
shared threads — a cc'd subset is a biased sample and will produce a confidently
wrong report.

**Always get the person's agreement before auditing their mailbox.** This reads
their private correspondence and names them in a document other people will see.
An instruction from their manager is not a substitute for telling the person.

### Constraints that shape the sampling
- `mailboxOwnerEmail` cannot be combined with `recipient`, and you must use
  *either* a free-text `query` *or* sender/date filters — not both.
- Paging in a shared mailbox uses `nextCursor` → `cursor`, not `offset`.
- 25 results per call is the hard ceiling.

## Procedure

### 1. Frame the window
Default to the last 90 days. Say the window in the report and never blur it —
a theme that is "rising" is only meaningful against a stated period.

### 2. Pull both directions, and count before you read
Search Inbox and Sent Items separately. The result set ends with a
`totalResultCount` — capture it for both. Those two numbers are the single most
diagnostic statistic in the report:

> **Sent ÷ received.** Above about 3:1 the person is a routing hub, not a
> processor. It is the cheapest evidence that a role is coordination overhead,
> and it costs two API calls.

Then sample. Aim for 75–150 messages across both folders, paging by
`offset` (own mailbox) or `cursor` (shared). Sample the *whole* window, not just
the newest page — page 1 is whatever happened this week.

### 3. Read the few messages that are worth reading in full
Metadata gets you patterns; bodies get you evidence. Use `read_resource` on:
- Any message where someone **enumerates their own manual work** ("I just got
  through fixing X, checking Y, moving Z"). One of these is worth fifty subject
  lines and belongs in the report as a pull-quote.
- Complaints about a recurring failure, especially "this happened again".
- Anything describing a workaround.

### 4. Classify into patterns, not senders
A pattern is *trigger + format + destination*. "Daily amenity list compiled from
the PMS and mailed to 16 people" is a pattern; "emails from Trenton" is not.
Note the actual sending addresses — automated senders are the strongest
automation candidates and their addresses are the proof.

Mark each pattern automatable only if the trigger, the format **and** the
destination are all consistent. Everything else is Partial.

### 5. Extract workflows with their decision points
For each recurring loop: trigger, the decisions a human currently makes, and the
owner. **The decisions are the analytical payload** — they separate work a
machine can take outright from work it can only stage for a person. A workflow
listed without its decisions is a description, not an audit.

### 6. Reconstruct decision sequences
Where threads show a chain of choices over weeks, lay them out in order with the
owner of each step. Only number things that genuinely are sequences.

### 7. Rank pain points
Severity = frequency × consequence. Rank by cost, not by annoyance. Two rules:

- **Revenue loss outranks hours saved.** A booking-inventory fault that silently
  suppresses covers beats a tedious weekly reconciliation, even though the
  reconciliation feels worse to the person doing it.
- **Give every hour estimate a range and label it directional.** These get
  quoted in business cases. Do not imply measurement you did not perform.

### 8. Build the report
Sections, in order:

1. Executive summary — with `Where the time goes`, `Key players`,
   `Biggest friction`, `Trend and opportunity`
2. Recurring email patterns — inbound and outbound tables (Pattern / Volume / Automatable)
3. Dominant themes — share of messages, with rising markers
4. People map — name, inferred role, internal or external
5. Workflows — cadence, trigger, decisions, owner
6. Decision sequences — ordered, with owners
7. Pain points and opportunities — severity, the fix, estimated hours
8. Timing patterns — daily rhythm and seasonal ramp
9. Method and limits — sample size against corpus size, and what is inferred

## Design

Load `artifact-design` before writing, and `dataviz` before any tile, meter or
chart. Grounding that is specific to this client rather than generic:

**Appellation's real palette**, lifted from the HTML signature block on any
internal email — use it instead of inventing one:

```
--ink:   #2B291F   /* near-black, warm */
--olive: #454F38   /* accent: eyebrows, labels, meters */
--rule:  #D4D1CB   /* hairlines */
```

Status colours, validated for colour-blind separation against a light surface:

```
--high: #B23A22    --med: #C08A1E    --low: #2F7A55
```

The amber falls just under 3:1 contrast, so **every severity must carry a text
label** — never colour alone. Type: `Newsreader` for prose, `IBM Plex Sans` for
UI, `IBM Plex Mono` for data. Define the full light palette on bare `:root` and
redefine tokens under both `prefers-color-scheme: dark` and `[data-theme="dark"]`.

Avoid the cream-and-terracotta look that a Sonoma wine brand invites; this is a
systems audit, and the register is institutional.

## Delivering it

The report names staff, quotes private mail and may carry guest details. **It
contains personal data.** Publishing it as an artifact may be blocked, and that
block is correct — do not attempt to route around it.

Deliver the file directly, and let the requester decide where it goes. If they
want it hosted, a private Trellis board behind their login is the right home,
not a public URL.

## Honesty rules

These matter more than the formatting.

- **State the sample against the corpus.** "75 of 1,171 messages" — never let a
  sampled finding read as a full-corpus count.
- **Separate counted from inferred.** `totalResultCount` is exact. Monthly
  averages, themes and role titles are inferred. Say which is which.
- **Never invent volume figures.** If you sampled 75 messages you cannot state
  what arrives monthly. Either count it or mark it an estimate.
- **Re-running a prior report?** Reproduce the earlier findings as the earlier
  window, and put new observations in a clearly separated addendum with its own
  date. Never silently merge two windows into one set of numbers.
