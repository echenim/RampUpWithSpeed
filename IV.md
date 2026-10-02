Act as a **Principal Backend & Systems Architect** reviewing concurrency control in the Torbit T1 Proxy.

Use these three completed audit documents as the **primary source of truth**:

- `backend_concurrency_limiter_system_audit.md`
- `fetch_semaphore_system_audit.md`
- `bcl_fetch_semaphore_interaction_audit.md`

Your task is to synthesize them into **one concise Design Review document** that explains the current architecture, the problems found, and what I recommend we do next.

The final document must be:

- straight to the point;
- easy to read;
- written in simple language;
- technically rigorous;
- suitable for a design review with senior engineers;
- written primarily in **first-person narration**, as if I am presenting my findings to the team.

Do not simply concatenate or summarize the three reports.

Instead, extract the most important conclusions, remove duplication, reconcile overlapping findings, and build one coherent technical argument.

---

# Primary Goal

The document should allow me to answer these questions clearly during a design review:

1. What is `BackendConcurrencyLimiter`, and how does it work?
2. What is the Fetch Semaphore, and how does it work?
3. Why do we currently have both?
4. Are they solving the same problem or different problems?
5. Can they operate together safely?
6. Does running both introduce unnecessary waiting, duplicated concurrency control, performance overhead, memory overhead, or operational complexity?
7. What bugs or weaknesses did the audits uncover?
8. Which issues should we fix regardless of the architectural decision?
9. Should we keep both mechanisms, change how they interact, disable one, or remove one?
10. What concrete next steps should the team take?

The document must end with a clear, evidence-backed recommendation.

---

# Source Rules

Treat the three audit documents as the authoritative evidence for this review.

Do not invent:

- new code behavior;
- production behavior;
- benchmark results;
- latency numbers;
- throughput numbers;
- memory numbers;
- configuration values;
- incidents;
- assumptions not supported by the audits.

If the audit documents disagree, do not silently choose one.

Call out the disagreement and explain what evidence would resolve it.

If something could not be proven through static analysis, preserve that limitation.

Use:

**Unverified — `<reason>`**

where appropriate.

---

# Writing Style

Write primarily in **first person**.

Use language such as:

- "I found that..."
- "What I see in the current implementation is..."
- "The important distinction here is..."
- "From the code, I can confirm..."
- "The concern I have with this design is..."
- "My recommendation is..."
- "I would change..."
- "I would keep..."
- "I would not remove this yet because..."

Avoid repeatedly writing:

- "the audit found";
- "the report says";
- "according to the document."

I am presenting the findings, so the document should sound like my technical assessment.

Keep paragraphs short.

Prefer direct statements over academic language.

Avoid unnecessary distributed-systems terminology when simpler language communicates the same point.

However, do not simplify away important distinctions such as:

- concurrency vs rate limiting;
- waiting/backpressure vs fail-fast load shedding;
- request vs backend attempt;
- logical backend vs backend host;
- retry budget vs general concurrency limit;
- static inference vs measured performance.

---

# Document Structure

Create the final Design Review using the following structure.

# Backend Concurrency Control Design Review

## 1. Executive Summary

Keep this very short.

In approximately 5–8 bullets explain:

- why I reviewed these mechanisms;
- what BCL does;
- what Fetch Semaphore does;
- whether they are equivalent;
- the most important interaction issue;
- the most important bugs or weaknesses;
- my recommended direction.

The recommendation should be visible immediately.

Do not make the reader wait until the end to understand my position.

---

## 2. Why We Have Two Mechanisms

Start by answering the basic architectural question.

Explain in simple terms:

### BackendConcurrencyLimiter

Describe:

- what it limits;
- where it acts;
- its scope;
- whether it waits or rejects;
- what happens when capacity is exhausted.

### Fetch Semaphore

Describe:

- what it limits;
- where it acts;
- its scope;
- waiting behavior;
- timeout behavior;
- the role of the retry semaphore.

Keep this section concise.

The purpose is to establish the mental model before discussing problems.

---

## 3. How a Request Moves Through the Controls

Describe the actual request path when both mechanisms are enabled.

Explain:

1. which mechanism is encountered first;
2. when capacity is acquired;
3. whether capacity remains held while waiting on another control;
4. what happens during retries;
5. when capacity is released.

Include one simple Mermaid diagram based strictly on the audit findings.

The diagram should emphasize:

- BCL gate;
- Fetch Semaphore gate;
- retry semaphore;
- backend attempt;
- rejection;
- waiting;
- release.

Do not create an overly detailed diagram.

---

## 4. Are BCL and Fetch Semaphore Doing the Same Thing?

Answer this directly.

Do not use a vague answer such as:

"They both limit concurrency."

Break the answer into:

### What overlaps

Explain the protection both mechanisms provide over similar backend work.

### What is different

Cover, where supported by the audits:

- fail-fast rejection vs waiting;
- BCL limit vs fetch semaphore capacity;
- retry semaphore / retry budget;
- keying differences;
- timeout behavior;
- error behavior;
- stale/fallback behavior.

### Conclusion

State clearly whether they are:

- fully redundant;
- partially overlapping;
- complementary;
- or structurally conflicting.

Explain why.

---

## 5. What Happens When Both Are Enabled

This section is important.

Explain the interaction between the two mechanisms.

Specifically address whether:

- a request can consume capacity in one mechanism while waiting on the other;
- requests can experience more than one concurrency gate;
- one configured limit can effectively dominate the other;
- requests can wait and later still be rejected;
- retries can interact with multiple controls;
- duplicate controls can reduce effective concurrency;
- extra locks, map lookups, semaphore operations, timers, or blocked goroutines are introduced.

Separate conclusions into:

### Verified from the code

State only things proven by static analysis.

### Potential runtime consequences

Examples may include:

- increased tail latency;
- reduced useful concurrency;
- more waiting goroutines;
- unnecessary hot-path work;
- harder capacity tuning.

Do not claim that these effects are already occurring in production unless the audit documents provide evidence.

---

## 6. Problems I Found

Consolidate findings from all three audits.

Do not list every minor issue.

Focus on the issues that are most important for correctness, reliability, performance, memory, or operability.

For each issue use this format:

### Issue N: `<clear title>`

**Severity:** High / Medium / Low

**Where:** relevant files/functions

### What I found

Explain the issue in first person.

### Why it matters

Explain the technical and operational impact.

### What I recommend

Provide a concrete correction.

Keep each issue compact.

---

# Findings That Must Be Considered

If confirmed by the audit documents, include these findings.

Do not include them if the audits disproved them.

### Request cancellation

Whether semaphore acquisition observes request cancellation.

If it does not, explain whether a canceled request can remain waiting.

### Semaphore map lifecycle

Whether per-backend semaphore entries are ever removed.

Explain whether this can result in retained state or metric-cardinality growth.

### BCL limiter map lifecycle

If `MutexMultiLimiter` retains backend keys, include this separately or combine it with the semaphore-map issue if that produces a clearer design-level finding.

### Sentinel error preservation

If retry-path semaphore exhaustion loses `ErrTimeoutConcurFetch` identity, explain the consequences for:

- `errors.Is`;
- metrics;
- stale/fallback behavior;
- final response behavior.

### Cache-Control response

If the malformed BCL `Cache-Control` value is confirmed, include it as a straightforward bug.

### Stale and fallback behavior

If overload responses bypass serve-stale or `FallbackHost`, explain whether that behavior is intentional or inconsistent.

### Interaction overhead

If a request can hold one concurrency slot while waiting on another gate, treat this as an architectural issue, not merely implementation cleanup.

---

## 7. What I Would Fix Regardless of Which Mechanism We Keep

Create a short list of corrections that should be made even before deciding whether one mechanism is removed.

For each item explain why it is independent of the larger architecture decision.

Examples, if supported by the audits:

- preserve sentinel error identity;
- make semaphore waits context-aware;
- fix malformed HTTP headers;
- fix lifecycle/eviction behavior;
- correct metrics;
- make stale/fallback behavior consistent.

Prioritize correctness fixes before optimization work.

---

## 8. Architectural Options

Evaluate these options.

### Option A — Keep both as they are

Explain:

- what protections remain;
- what problems remain;
- operational cost.

### Option B — Keep both, but fix and coordinate them

Explain:

- what should change;
- whether limits need explicit relationship;
- whether acquisition ordering should change;
- whether observability needs improvement.

### Option C — Keep BCL and disable/remove Fetch Semaphore

Explain:

- what BCL covers;
- what behavior would be lost;
- specifically address retry budget and blocking/backpressure;
- what would need replacement before removal is safe.

### Option D — Keep Fetch Semaphore and disable/remove BCL

Explain:

- what Fetch Semaphore covers;
- what fail-fast behavior would be lost;
- what other consequences follow.

Do not force all options to appear equally viable.

Use code evidence to explain the trade-offs.

---

## 9. My Recommendation

This section must be decisive but evidence-based.

State clearly:

### What I recommend now

For example, depending on the audit evidence:

- keep both temporarily and fix correctness issues first;
- keep BCL and redesign retry protection;
- disable Fetch Semaphore behind configuration before considering deletion;
- coordinate limits and acquisition ordering;
- another supported conclusion.

Do not choose the answer before reading the three audit documents.

### Why

Give the 3–5 strongest reasons.

### What I would not do yet

State actions that would be premature.

For example:

- deleting a mechanism before proving its unique responsibilities are replaced;
- sizing limits from unreliable metrics;
- making performance claims from static analysis.

### Conditions that would change my recommendation

Identify any unresolved evidence that could change the decision.

---

## 10. Recommended Actions

Create a concise prioritized action plan.

Use:

### P0 — Correctness

Things that should be fixed first.

### P1 — Architecture

Changes to concurrency-control behavior or interaction.

### P2 — Performance / Memory

Improvements that reduce overhead or retained state.

### P3 — Cleanup

Simplification and removal after behavior is proven safe.

For each action include:

- what should change;
- why;
- affected component;
- dependency, if any.

Do not turn this into a detailed implementation plan.

---

## 11. Proposed Decision Path

Provide a short sequence describing how I recommend moving from the current state to the target architecture.

Example structure:

1. Fix correctness issues that distort behavior or observability.
2. Add or correct required telemetry.
3. Make the mechanism intended for possible removal independently disableable if it is not already.
4. Validate behavior with one mechanism disabled.
5. Compare backend protection, errors, retries, stale/fallback behavior, and concurrency.
6. Only remove code after equivalent required protection is demonstrated.

The exact sequence must come from the audit findings.

Do not add a load-testing plan unless it is required to validate an architectural decision.

If runtime validation is required, state only what must be proven and which metrics/behaviors should be observed.

---

## 12. Remaining Questions

List only questions that materially affect the decision.

For each use:

**Question:**
`<question>`

**Why it matters:**
`<reason>`

**Evidence needed:**
`<specific artifact, config, metric, or runtime observation>`

Avoid generic open questions.

---

## 13. Final Takeaway

End with 1–3 short paragraphs written as if I am concluding the design review verbally.

The reader should leave knowing:

- whether the two mechanisms are equivalent;
- what the largest design concern is;
- what should be fixed immediately;
- what architectural direction I recommend.

---

# Comparison Table

Include one concise comparison table somewhere before the recommendation:

| Characteristic       | BackendConcurrencyLimiter | Fetch Semaphore |
| -------------------- | ------------------------- | --------------- |
| What it limits       |                           |                 |
| Key / scope          |                           |                 |
| Acquisition behavior |                           |                 |
| On limit reached     |                           |                 |
| Waiting              |                           |                 |
| Timeout              |                           |                 |
| Retry protection     |                           |                 |
| Response behavior    |                           |                 |
| Cancellation aware   |                           |                 |
| State lifecycle      |                           |                 |
| Main strength        |                           |                 |
| Main weakness        |                           |                 |

Only include values proven by the audit documents.

---

# Recommendation Quality Bar

Recommendations must be specific.

Avoid weak statements such as:

- "We should improve this."
- "We should monitor it."
- "Consider optimizing this."
- "More investigation is needed."

Instead say exactly what should change and why.

For example:

> I recommend making semaphore acquisition context-aware because the current acquisition path can continue waiting after the request context has been canceled.

Or:

> I would not remove the Fetch Semaphore solely because BCL limits backend concurrency. The retry semaphore provides a separate retry budget that BCL does not currently replace.

Only make statements like these if the source audits support them.

---

# Technical Accuracy Rules

Preserve these distinctions throughout the document:

**Concurrency limiting != rate limiting**

**Waiting/backpressure != fail-fast load shedding**

**Logical backend identity != necessarily backend host identity**

**Request concurrency != connection concurrency**

**Retry concurrency != initial-fetch concurrency**

**Potential performance impact != measured performance degradation**

Never collapse these into generic language such as:

"Both mechanisms rate limit backend traffic."

---

# Evidence

Every important conclusion must remain traceable to code evidence from the source audit documents.

Use:

`path/to/file.go` + symbol + line range

where useful.

Do not overwhelm the design review with code quotations.

Use the minimum code necessary to support the argument.

The detailed audit documents already contain the forensic evidence; this document should focus on the design decision.

---

# Output

Save the final document as:

`@Torbit/Odnd_Load_Testing/<ID>/backend_concurrency_design_review.md`

Use the same investigation ID as the three source audit documents when appropriate.

The final design review should be significantly shorter than the combined audits.

Target a document that a senior engineer can read in approximately **10–15 minutes** and understand:

- how the architecture works;
- what is wrong;
- what matters;
- what I recommend;
- what we should do next.
