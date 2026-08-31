# Email Analyzer — Handoff

**Purpose of this file.** Everything a fresh session needs to run, rebuild or host the
Email Analyzer as a standalone tool. No dependency on any external platform, vendor or
hosted service. Read this top to bottom before touching anything.

**Owner:** Eric Fox, Director of Food & Beverage, Appellation Healdsburg
**Status:** working. Skill committed; one report generated and delivered.
**Last run:** 31 August 2026, against `eric.fox@appellationhotels.com`

---

## 1. What the tool is

Analyse one person's mailbox, infer how the organisation actually runs from the traffic,
and rank what could be handed to a machine. Output is a single self-contained HTML report.

The reader is an executive deciding **where to start automating**. So the deliverable is a
prioritised argument, not a description of an inbox. If the report reads as "here is what
is in this mailbox," it has failed.

The core insight the tool exists to surface: in an operations role, most email is not
communication — it is **manual data movement between systems that do not talk to each
other**. The mailbox is the integration layer. Finding that is the job.

---

## 2. Current state

| Item | Where | State |
|---|---|---|
| Skill definition | `.claude/skills/email-intelligence-report/SKILL.md` | Committed |
| Repo | `appellation-healdsburg-reviews` | On a working branch — see `git branch -a` |
| Generated report | `email-intelligence-report.html` | Delivered as a file to Eric |
| Hosting | none | **Open — see §8** |

**If you are lifting this out into its own project:** the skill is self-contained, a single
`SKILL.md` with no supporting files. Copy the directory anywhere. The repo and branch it
currently sits in are incidental — it was committed there because that repo was open at the
time, not because it belongs there. A standalone `email-analyzer` repo would be cleaner.

---

## 3. Data source and access model

**Source:** Microsoft 365 / Outlook, via the Microsoft 365 MCP connector.
**Tools used:** `outlook_email_search`, `read_resource`. Nothing else is required.

### Access is the whole problem

The analysis is straightforward. Getting into the mailbox is what actually gates the job,
and it cannot be arranged from inside a session.

| Whose mailbox | What you need | Who provides it |
|---|---|---|
| Your own | nothing — omit `mailboxOwnerEmail` | — |
| Someone else's | Delegate access, granted in Outlook | The mailbox owner |
| Someone else's | `Mail.Read.Shared` on the connector | A Microsoft 365 admin |

Confirm access with the smallest possible probe before promising anything:

```
outlook_email_search(mailboxOwnerEmail: "<them>@appellationhotels.com", limit: 1)
```

**If that fails, stop and say so.** Do not substitute your own mailbox, and do not
reconstruct someone's work from your copies of threads you were cc'd on. A cc'd subset is a
biased sample — it contains only what involved you — and will produce a confidently wrong
report about someone else's job.

### Consent

Always get the person's agreement before auditing their mailbox. This reads their private
correspondence and names them in a document other people will read. An instruction from
their manager is not a substitute for the person knowing.

---

## 4. Sampling procedure

### 4.1 Frame the window
Default 90 days. State it in the report and never blur it — a "rising" theme is meaningless
without a stated period.

### 4.2 Count before you read
Search Inbox and Sent Items separately. The result set's final item carries
`totalResultCount`. Capture both numbers first. They give you the single most diagnostic
statistic in the whole report for two API calls:

> **Sent ÷ received.** Above ~3:1 the person is a routing hub, not a processor. It is the
> cheapest possible evidence that a role has become coordination overhead.
>
> Eric's August figure: **990 sent / 181 received = 5.5:1.**

### 4.3 Sample
75–150 messages across both folders. Sample the *whole* window — page 1 is just whatever
happened this week. Page with `offset` (own mailbox) or `cursor` (shared mailbox).

### 4.4 Read the few that matter in full
Metadata gives you patterns; bodies give you evidence. Use `read_resource` on:

- Any message where someone **enumerates their own manual work** — "I just got through
  fixing X, checking Y, moving Z." One of these is worth fifty subject lines, and belongs
  in the report as a pull-quote.
- Complaints about a recurring failure, especially the phrase *"this happened again"*.
- Anything describing a workaround.

URI form: `mail:///messages/{id}` — add `?owner={email}` for a delegated mailbox.

### 4.5 API constraints that shape all of the above

These are real and will bite:

- **25 results per call**, hard ceiling.
- `mailboxOwnerEmail` **cannot** combine with `recipient`.
- With `mailboxOwnerEmail` or `folderName` set: use **either** a free-text `query` **or**
  sender/date filters — not both.
- Free-text `query` inside a folder or shared mailbox pages by **`nextCursor` → `cursor`**,
  not `offset`.
- `order` is incompatible with free-text `query` and with `recipient`.

---

## 5. Analysis method

### 5.1 Patterns, not senders
A pattern is **trigger + format + destination**. "Daily amenity list compiled from the PMS
and mailed to 16 people" is a pattern. "Emails from Trenton" is not.

Record the actual sending addresses. Automated senders are the strongest automation
candidates and their addresses are the proof.

Mark a pattern **Automatable** only if trigger, format *and* destination are all consistent.
Everything else is **Partial**. Two states only — resist inventing a third.

### 5.2 Workflows with decision points
For each recurring loop capture: cadence, trigger, **the decisions a human currently makes**,
and the owner.

The decisions are the analytical payload. They are what separates work a machine can take
outright from work it can only stage for a person. A workflow listed without its decisions
is a description, not an audit.

### 5.3 Decision sequences
Where threads show a chain of choices playing out over weeks, lay them out in order with the
owner of each step. Only number things that genuinely are sequences — numbering a non-sequence
is decoration pretending to be information.

### 5.4 Ranking pain points
Severity = frequency × consequence. Two rules that change the ordering:

- **Revenue loss outranks hours saved.** A booking-inventory fault that silently suppresses
  covers beats a tedious weekly reconciliation, even though the reconciliation feels far
  worse to the person doing it. This flipped the ranking on the last run — see §9.
- **Every hour estimate gets a range and a "directional" label.** These numbers get quoted
  in business cases. Do not imply measurement you did not perform.

---

## 6. Report specification

Nine sections, in this order. This structure is the product — keep it stable across runs so
reports are comparable.

| # | Section | Contents |
|---|---|---|
| 01 | Executive summary | Four labelled movements: `Where the time goes`, `Key players`, `Biggest friction`, `Trend and opportunity`. Plus a KPI tile row. |
| 02 | Recurring email patterns | Two tables, inbound and outbound: Pattern / Volume / Automatable. Sending addresses under each pattern name. |
| 03 | Dominant themes | Share of messages as a percentage, with a meter and a rising marker. Ordered by share. |
| 04 | People map | Name, inferred role, internal or external. |
| 05 | Workflows | Cards: cadence, trigger, decisions, owner. |
| 06 | Decision sequences | Ordered numbered steps, each with its owner. |
| 07 | Pain points & opportunities | Severity chip, the problem, the automation opportunity, estimated hours. |
| 08 | Timing patterns | Daily rhythm and seasonal ramp. |
| 09 | Method & limits | Sample size against corpus size; what is counted vs inferred. |

---

## 7. Design system

Load `artifact-design` before writing the page, and `dataviz` before any tile, meter or chart.

### Palette — Appellation's own
Taken from the HTML signature block on any internal Appellation email. Use it rather than
inventing one; it is genuinely theirs.

```css
--ink:   #2B291F;   /* near-black, warm */
--olive: #454F38;   /* accent: eyebrows, labels, meters */
--rule:  #D4D1CB;   /* hairlines */
```

Ground `#F4F4F1` (cool neutral — deliberately *not* cream), surface `#FFFFFF`.

### Status colours
Validated for colour-blind separation against a light surface (CVD ΔE and normal-vision
floor both pass):

```css
--high: #B23A22;   --med: #C08A1E;   --low: #2F7A55;
```

The amber lands just under 3:1 contrast. **Therefore every severity must carry a visible
text label — never colour alone.** This is not optional; it is the relief that makes the
amber legal.

### Type
- `Newsreader` — prose and headings
- `IBM Plex Sans` — UI, labels, tables
- `IBM Plex Mono` — data, addresses, metrics

### Theming
Define the complete light palette on bare `:root`. Redefine tokens under **both**
`@media (prefers-color-scheme: dark)` guarded as `:root:not([data-theme="light"])` **and**
`:root[data-theme="dark"]`. Give `body` an explicit token background — a transparent body
borrows the host's ground and breaks.

### Register
Avoid the cream-and-terracotta, big-serif look that a Sonoma wine brand invites. This is a
systems audit. The register is institutional — closer to an operations ledger than a menu.

---

## 8. Hosting — open item

**The report contains personal data**: named staff, quoted private correspondence, guest
names and room numbers. This constrains every hosting decision.

Publishing it through a general artifact/publishing path was **blocked by a permissions
classifier** on the last run. That block was correct and should not be worked around.

Standalone options, in order of preference:

1. **Authenticated static host.** A small static site behind SSO or basic auth. Best fit —
   the report is a single self-contained HTML file with no build step and no runtime
   dependencies, so any host that can serve a file and check a login will do.
2. **Direct file delivery.** Send the HTML to the requester; they open it locally. Zero
   infrastructure, zero exposure. This is what happened on the last run and it was adequate.
3. **Private repo + access-controlled pages.** Note that GitHub Pages from a *private* repo
   requires an Enterprise plan; on any other plan the site is public. Do not host this
   publicly.

**Do not** put it on a public URL, and do not email it to a distribution list.

---

## 9. Worked example — the 31 August 2026 run

Run against `eric.fox@appellationhotels.com`, 1 July – 31 August, sampling 75 of 1,171
messages. Use this to calibrate what good output looks like.

### Headline finding
**990 sent / 181 received.** The dominant activity of this mailbox is routing.

### The evidence that carried the report
A manager, explaining why he had not read his email, wrote out his own close-out sequence:

> "I just got through fixing the tip sheets in AVA, checking Paylocity times to make sure
> they were all accurate, moving tables around in OpenTable to prepare for dinner service
> tomorrow, setting up the floorplan for tomorrow and now checking my emails."

Four systems, touched by hand, every close. This is the entire thesis of the tool in one
sentence, stated by an operator rather than inferred by an analyst. **Always hunt for this
message.**

### Ranking that changed on inspection
Folia's outdoor tables were found unbookable on OpenTable for an upcoming dinner service —
and the same fault had run for most of a month earlier in the year before someone caught it
by chance. Nothing watches for it.

This was initially sorted below several reconciliation tasks that consume more hours. It was
promoted above them because the cost is **lost covers, not lost time**, and it recurs
silently. This is the §5.4 rule in action.

### A pattern the tool caught that a person would not
Front-desk amenities are compiled and mailed twice daily to sixteen recipients — an evening
list for tomorrow, then an "UPDATED" list the next morning — with guest names, room numbers
and occasion tags transcribed by hand from the PMS. Clean template over structured data,
typed out by a person, twice a day. Nobody had flagged it because each instance takes ten
minutes.

### Recurring access administration
One manager's system login reset consumed five messages across two hours of the Director's
day and ended with a password sent over email. Trivial individually, routine in aggregate,
entirely a self-service problem — and a security finding as much as an efficiency one.

### Operational stack observed
Toast (POS/KDS), OpenTable, Paylocity, AVA, BinWise, Revinate, Salesforce, ResortPass,
MarginEdge, BlueCart, PMS. The gaps *between* these are where the mailbox work lives.

---

## 10. Honesty rules

These matter more than the formatting, and they are the easiest thing to get wrong under
time pressure.

- **State the sample against the corpus.** "75 of 1,171 messages." Never let a sampled
  finding read as a full-corpus count.
- **Separate counted from inferred.** `totalResultCount` is exact. Monthly averages, theme
  percentages and role titles are inferred. Say which is which, in §09 of the report.
- **Never invent volume figures.** If you sampled 75 messages you cannot state what arrives
  monthly. Either count it or label it an estimate.
- **Re-running against a prior report?** Reproduce the earlier findings as the earlier
  window, and put new observations in a clearly separated, separately dated addendum. Never
  silently merge two windows into one set of numbers — that is how a report becomes
  unfalsifiable.

---

## 11. Next steps

1. **Decide hosting** (§8). Currently unresolved; direct file delivery is the working default.
2. **Relocate the skill** into its own repo if this is to be a standalone tool (§2).
3. **Arrange delegate access** for the next subject mailbox before scheduling a run (§3).
   This is the long pole every time.
4. **Consider a scheduled cadence.** Quarterly per mailbox is enough; the themes move slowly
   and monthly runs would mostly restate the previous one.
5. **Extend to a second mailbox** to test the method against a different role shape. A
   non-hub role — someone who processes rather than routes — is the useful contrast case and
   will stress-test the sent÷received heuristic.
