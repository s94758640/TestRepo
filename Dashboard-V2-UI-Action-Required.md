# Dashboard v2 — action required from the UI team

## Status update — 18 Nov

You raised three drill-down issues against the 11–17 Nov range. **Two were ours and are
fixed; one is still open and blocks a click you should not wire up yet.**

| | Issue | Status |
|---|---|---|
| D4 | `dataTypes` returned 0 for POLICY, short for PII and PCI | **Fixed, API side. No UI change — your bodies were correct.** |
| D5 | `apps:["Claude"]` returned 13 against a tile of 48 | **Fixed, API side. No UI change.** |
| D6 | `{"apps":["Claude"]}` with no `areas` returns 500 | **Open.** Do not wire or retest Overview → Top Incidents by Apps clicks yet. |
| D1 | drill-down not scoped to the tab | **Still yours.** The rest of this document. |

**On D4 and D5 — you were right and nothing on your side needs to change.** The cards
resolve values through a catalogue before displaying them and the filters did not, so the
`code` we handed you could not be matched back. Both filters now expand each value to every
name the column holds it under.

**One thing to expect when you retest D4.** Sensitive Data Detected counts *identifier
mentions*; the panel counts *events*. One prompt carrying an email address and an Aadhaar
number adds 2 to the card's PII row and 1 to the panel. So the tile can still read slightly
higher — that is correct. What it must never do again is return 0 or a small fraction.

**Two API-side number changes shipped in the same build**, not your work, please do not
chase them:

- Security Score → Compliance now matches the headline Policy Compliance tile. "N policies
  need review" will also rise where web policies are active.
- Top Blocked Domains is now **one row per domain** instead of one per domain + category.
  If your row component assumed that pair was unique, it no longer needs to.

### What we need back from you

1. **Which widgets now send `areas`, and which are still to come?** The D4 and D5 retests
   depend on correct bodies, so we need to know what is already converted.
2. **For the 500: the exact failing request and roughly when.** The exception is swallowed
   into the generic message; with a timestamp we can pull the log line that names it.
3. **Confirm the user-chip second hop carries `areas` forward** — see "Two more things to
   get right" below. Dropping it there reintroduces the leak one level down.

---

**The original ask — one change: every drill-down click that happens on a tab must send
that tab's `areas`.**

Nothing else about the integration changes. No new endpoint, no new field, no response
shape change. One property, added to the request body you already send.

---

## What's wrong today

Open **Dashboard → File Sharing → Data Classification → PII**. The panel shows more events
than the card you clicked, and the list contains rows like:

```
PFI, PII, FIN_RECORDS, PCI
Selva S · Prompt · www.google.com                    Blocked
```

**A Prompt is a Content Sharing event.** It should not be in a panel opened from the File
Sharing tab. The same leak happens on every tab-local widget: Top Destinations, Top Device
Types, Risk by Channel, Largest Transfers, Top AI Tools, Sensitive Data Detected, and the
Web Filtering lists.

## Why

`drill-down` is **one endpoint over all three populations** — file, content and web. There
is no tab in the URL, no tab in the path, and no session state. **The request body is the
only thing that tells the API which tab you are on.**

The body being sent is:

```json
{ "fromDate": "…", "toDate": "…", "dataTypes": ["PII"] }
```

With no `areas`, that means *"PII in file **and** content"* — and the API answers it
correctly. It is not a server bug; it is the question that was asked.

It should be:

```json
{ "fromDate": "…", "toDate": "…", "areas": ["file"], "dataTypes": ["PII"] }
```

> **This was our documentation's fault.** The old Click → filter map listed un-scoped
> filters for tab-local widgets. It has been rewritten — see
> `Dashboard-V2-API-Documentation.html → Click → filter map`, now split into four per-tab
> tables with the exact body for every widget.

---

## The rule

| Tab | Every drill-down body must include |
|---|---|
| Overview | *nothing* — Overview widgets are genuinely cross-area |
| File Sharing | `"areas": ["file"]` |
| Content Sharing | `"areas": ["content"]` |
| Web Filtering | `"areas": ["web"]` |

Where the widget's own filter already implies the area — `channels:["upload"]` can only be
file, `channels:["prompt"]` can only be content — sending `areas` as well is redundant but
harmless. **Send it everywhere.** It costs less than remembering which cases are which.

The cleanest implementation is to hold the active tab's area in the drill-down helper and
merge it into every body, rather than adding it at each of the ~30 call sites.

---

## Exact bodies — File Sharing tab

| Clicked | Body |
|---|---|
| Total File Events tile | `{"areas":["file"]}` |
| Upload / Download / Device Write / Device Read tile | `{"areas":["file"],"channels":["upload"]}` |
| File Activity Split segment or legend row | `{"areas":["file"],"channels":["device_write"]}` |
| Top Destinations row | `{"areas":["file"],"apps":["Google Chrome"]}` |
| Data Classification category | `{"areas":["file"],"dataTypes":["PII"]}` |
| Top Device Types row | `{"areas":["file"],"devices":["USB"]}` |
| Risk by Channel — channel name | `{"areas":["file"],"channels":["upload"]}` |
| Risk by Channel — a severity segment | `{"areas":["file"],"channels":["upload"],"severities":["Critical"]}` |
| Risk by Channel — the "Stopped" % | `{"areas":["file"],"channels":["upload"],"actions":["Blocked"]}` |
| Largest Transfers row | `{"areas":["file"],"userIds":[11],"search":"upload.bin"}` |

Channel values: `upload` · `download` · `device_read` · `device_write`

## Exact bodies — Content Sharing tab

| Clicked | Body |
|---|---|
| Total Content Events tile | `{"areas":["content"]}` |
| Copy / Paste, Typed Content, Prompt tile | `{"areas":["content"],"channels":["prompt"]}` |
| Blocked tile | `{"areas":["content"],"actions":["Blocked"]}` |
| Content Activity Split segment or legend row | `{"areas":["content"],"channels":["copy_paste"]}` |
| Top AI Tools by Prompts row | `{"areas":["content"],"apps":["Claude"]}` |
| Sensitive Data Detected tile | `{"areas":["content"],"dataTypes":["PII"]}` |
| Response Outcome segment | `{"areas":["content"],"actions":["Blocked"]}` |
| Risk by Channel — channel / severity / Stopped | as File Sharing, with `"areas":["content"]` |
| Top Users Sharing Sensitive Content row | `{"areas":["content"],"userIds":[11]}` |

Channel values: `copy_paste` · `typed` · `prompt`
Action values: `Blocked` · `Warned` · `Allowed` · `Override` — **Override exists on this tab only.**

## Exact bodies — Web Filtering tab

| Clicked | Body |
|---|---|
| Policy-Matched Requests tile | `{"areas":["web"]}` |
| Blocked / Warned / Allowed tile | `{"areas":["web"],"actions":["Blocked"]}` |
| Top Categories segment or legend row | `{"areas":["web"],"categories":["GAMBLING"]}` |
| Top Blocked Domains row | `{"areas":["web"],"domains":["you.com"],"actions":["Blocked"]}` |
| Top Users by Blocks row | `{"areas":["web"],"userIds":[11],"actions":["Blocked"]}` |
| Risk by Category row | `{"areas":["web"],"categories":["GAMBLING"]}` |

> **Do not send `channels`, `severities`, `dataTypes`, `devices` or `minSizeBytes` with
> `"areas":["web"]`.** A web request has no channel, no severity and no scanned content,
> so those fields cannot match one. The API skips the web read entirely rather than
> returning rows that don't satisfy the filter, and your panel comes back empty. If a Web
> Filtering widget seems to need one of those, it's the wrong filter for that widget.

## Overview tab — unchanged

Overview widgets are cross-area by design and may leave `areas` out.

| Clicked | Body |
|---|---|
| Total Incidents · Policy Compliance | `{"actions":["Blocked","Warned","Override"]}` |
| High-Risk Users tile or list row | `{"userIds":[11],"severities":["Critical","High"]}` |
| Off-Hour Activity | `{"offHoursOnly":true}` |
| A File / Content / Web area card | `{"areas":["file"]}` |
| A channel inside an area card | `{"areas":["file"],"channels":["upload"]}` |
| An Incidents-by-Area line | `{"areas":["content"]}` |
| A heat-map cell | `{"daysOfWeek":[1],"hours":[9]}` |
| Top Incidents by Apps row | `{"apps":["Teams"]}` |
| Data Classification category | `{"dataTypes":["PII"]}` |

---

## Two more things to get right

**1. The user chip inside the panel must keep `areas`.**
Clicking a top-user chip re-opens the panel as *the original filter AND that user*. Carry
the whole previous body forward and add `userIds` — including its `areas`. Dropping it on
the second hop reintroduces the same leak one level down.

```json
{"areas":["file"],"dataTypes":["PII"],"userIds":[237]}
```

**2. `Total Incidents` needs all three actions.**
`totalIncidents` = Blocked + Warned + **Override**. The matching drill-down filter is
`{"actions":["Blocked","Warned","Override"]}`. With only two values you're asking a
narrower question and the panel will correctly show a smaller number than the tile.

---

## How to verify you've got it

For each of the four tabs, click every clickable widget and check:

1. The request body carries that tab's `areas`.
2. **The panel's Events count equals the number on the widget you clicked.** This is the
   real test — if the panel reads higher than the card, a filter is still missing.
3. On **File Sharing**, no listed row has a `Prompt`, `Copy / Paste` or `Typed Content`
   channel.
4. On **Content Sharing**, no listed row is an `Upload`, `Download` or device event.
5. On **Web Filtering**, every listed row is a domain.

---

## Heads-up — two API-side changes in the same build

Not your work, but you'll see the numbers move. Please don't chase them:

- **Security Score → Compliance now matches the headline Policy Compliance tile.** It was
  counting file + content only while the tile counted those plus web, so the Overview
  showed two different percentages (15% and 11%) under the same word. Both now count all
  three areas. **"N policies need review" will also go up** on tenants with active web
  policies — a web policy's noise is now counted too.
- **Top Blocked Domains is now one row per domain.** `chatgpt.com` used to appear twice
  (146 Uncategorised + 40 Computers And Technology), splitting its rank. Each domain now
  has a single row with its full block count, and the sub-line shows the category *most*
  of those blocks carried. If your row component assumed domain+category uniqueness, it
  no longer needs to.

---

## References

- `Dashboard-V2-API-Documentation.html` — the contract. The **Click → filter map** section
  is the one to work from; it's now four per-tab tables.
- `Dashboard-V2-QA-KT.html` — the QA handover. Useful for understanding what each number
  means before you wire a click to it.
