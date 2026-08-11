# Inbound Lead Enrichment & AI Scoring Pipeline

**Author:** Adilsha S Rehman
**Stack:** Make.com · Apollo.io API · Anthropic Claude API · Google Forms · Google Sheets · Slack

An inbound lead pipeline that enriches trial signups with firmographic data, scores them against ICP criteria, and routes only high-fit accounts to the sales channel — with the cheap checks placed before the expensive ones.

---

## The problem

Inbound trial signups arrive with self-reported data that is incomplete, occasionally wrong, and sometimes deliberately understated. Account Executives either research every lead manually or work the queue in arrival order. Both waste time on accounts that were never going to close.

This pipeline enriches each signup against Apollo, scores it, and alerts the sales channel only for leads worth a call.

---

## Architecture

```
Google Forms (polling trigger)
    │
    └── Set Variable — extract company domain from email
            │
            └── ROUTER
                ├── Route 1 · business email
                │     └── HTTP — Apollo organisation enrichment
                │           └── [Resume error handler]
                │                 └── Claude — ICP scoring (schema-enforced JSON)
                │                       └── Sheets — log every lead
                │                             └── ROUTER
                │                                 └── fit_score ≥ 8 → Slack alert
                │
                └── Route 2 · consumer email (fallback)
                      └── Sheets — logged, not enriched
```

### Design decisions

**Free-email domains are filtered before the API call.** Consumer signups can't be enriched, so spending an Apollo credit and an LLM call on them is waste. They're routed to logging instead of being dropped, so the record stays complete.

**Consumer rows still keep the lead's own data.** Name, email, phone, and self-reported company size are all recorded on the fallback route. Only the Apollo-derived fields are marked unenriched. The distinction matters: "we didn't look this up" is not the same as "we don't know."

**Every lead is logged, not just the winners.** The Sheets module sits before the scoring router. Scoring quality can't be reviewed if only high scorers are kept.

**Facts come from the source, judgement comes from the model.** Headcount, industry, revenue, and company name map straight from the Apollo response into Sheets and Slack. Claude outputs only what it generates: the score, the hidden-enterprise flag, and the reasoning. Passing factual data through an LLM to get it back adds cost and a transcription risk for nothing.

**Apollo failures don't kill the run.** A Resume error handler on the HTTP module lets the scenario continue with an empty bundle. Every Apollo field in the prompt is wrapped in `ifempty(...; "Unknown")`, so the model receives an explicit "Unknown" rather than a blank, and is instructed to cap the score accordingly.

**Output format is enforced at the API level.** The Claude module uses a JSON schema rather than prompt instructions plus post-hoc cleanup, which removes an entire class of parsing failure.

### The hidden-enterprise rule

The most valuable signal in inbound isn't the self-reported size — it's the gap between what a lead claims and what they are.

`is_hidden_enterprise` is set true when Apollo's headcount is at least **3× the maximum of the self-reported range**. Explicitly numeric, because an earlier version asked the model to judge a "significant margin" and the boundary was therefore the model's to set, not mine.

One exception: if the lead selected the highest tier (`1000+`), the flag stays false regardless of actual size. They picked the largest option available — there was nothing more accurate for them to choose.

---

## Testing

Nine cases across three rounds, covering large discrepancies, the top-tier exception, mid-market, genuine SMBs, consumer emails, uppercase domains, unknown domains, and reproducibility.

| Case | Self-reported | Apollo | Score | Flagged | Outcome |
|---|---|---|---|---|---|
| Siemens | 0–99 | 313,000 | 9 | Yes | Correct — 3,161× gap |
| Shopify (`Shopify.COM`) | 100–499 | 7,600 | 9 | Yes | Correct at 15.2×; uppercase domain matched |
| Stripe | 1000+ | 17,000 | 9 | No | Correct — top-tier exception applied |
| Apple | 500–999 | 166,000 | 9 | Yes | Correct |
| Buffer | 100–499 | 330 | 3 | No | Correct — below ICP threshold |
| Basecamp | 1–99 | 3 | 2 | No | Scored correctly on wrong data (see below) |
| Unknown domain | 500–999 | Unknown | 5 | No | Survived enrichment failure |
| Consumer email (Yahoo) | 500–999 | Not called | — | — | Routed to fallback, logged unenriched |
| Consumer email (Gmail) | 500–999 | 200, empty body | 5 | No | Pre-router behaviour; now routed to fallback |

The score distribution separates cleanly: 2–3 for genuine small companies, 5 for unresolvable enrichment, 9 for real enterprises. Nothing lands near the threshold, which is why the exact cut matters less than it might appear at this sample size.

### What broke

**Temperature was left at the default of 1** — maximum output variation — on a task that assigns a number to a routing decision. The same lead could score either side of the threshold across runs. Set to 0. Variation is acceptable when generating prose and unacceptable when generating a score.

**The scoring scale silently drifted to 0–100.** After moving to structured output, a run returned `fit_score: 85`. The schema guaranteed an integer; nothing constrained the range; and the router's `≥ 8` check passed it through as a VIP lead without complaint. The fix went into the prompt rather than the schema, because Make's JSON schema implementation rejects `minimum` and `maximum` on integer fields — a platform constraint that pushes value validation back into either the prompt or a scenario filter. Structure validation and value validation are separate problems, and only one of them was being done.

**Claude wrapped its JSON in a markdown code fence** despite an explicit instruction not to. Caught by a regex parser at the time — the reason the pipeline later moved to schema-enforced output rather than cleaning up after the fact.

**Apollo returned inaccurate data in two of nine cases.** A query for `basecamp.com` came back as "Basecamp Talent" — 3 employees, $12.3M revenue — rather than 37signals. `buffer.com` returned 330 employees against a company generally reported at under 100. The pipeline scored both confidently. Enrichment is evidence, not ground truth, and a model reasoning over bad data produces a wrong answer with the same confidence as a right one. Verifying this properly needs a submitted company name to compare against the returned record; the trial form this mirrors doesn't collect one.

**An API key was exposed in an exported blueprint.** The HTTP module was configured with no-auth and the key pasted into a header, so it travelled with the export. Key rotated, credential moved into Make's keychain, exports checked before committing.

**Google Forms rate-limited the trigger during testing.** The trigger was set to pull up to 1,500 responses per run — fine on a 15-minute production schedule, wasteful when re-running repeatedly. Reduced the limit and switched to running individual modules against cached bundles instead of re-triggering the whole scenario.

---

## Known limitations

- **No deduplication.** The same person submitting twice produces two alerts. A data store keyed on email would fix it, though permanent dedup is wrong for sales — a prospect returning after six months is a real lead, so any implementation needs a time window.
- **The threshold of 8 is a judgement, not a calibration.** In production it would be set against closed-won data rather than intuition.
- **Polling, not real-time.** Make's Google Forms module is a polling trigger, and the plan minimum is 15 minutes. Instant delivery would need a Google Apps Script pushing to a Make webhook on submit — at the cost of maintaining a script outside the platform.
- **No enrichment confidence signal.** See the Apollo accuracy finding above.
- **Free-email list is hardcoded.** Four domains cover the common cases; a maintained list or a disposable-domain API would generalise.

---

## Tool selection

Make over Zapier for this build: the flow branches twice, needs an error handler on an external API call, and benefits from per-module data inspection when debugging an unpredictable LLM response. Zapier's paths and filters could express the branching, but its error handling is thinner and its per-task pricing is worse at volume.

The visual canvas is a genuine trade-off rather than a pure win — branching and inline data inspection come at the cost of spatial management that a linear builder doesn't have.

---

## What this demonstrates

Ordering work by cost: filter before you enrich, enrich before you infer. Treating enrichment data as evidence rather than fact. And validating model output on two axes — that it has the right shape, and that its values are in range — because the second failure is the one that routes quietly.
