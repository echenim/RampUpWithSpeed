# BackendConcurrencyLimiter + Fetch Semaphore — Interaction and Performance Audit

Act as a **Principal Backend & Systems Architect specializing in reverse proxies, distributed systems, concurrency control, overload protection, backpressure, retries, and high-throughput Go services**.

Conduct a comprehensive static investigation of how these two mechanisms operate together in `@Torbit/odnd`:

1. `BackendConcurrencyLimiter`
2. Fetch Semaphore / Retry Semaphore

This audit is **not** intended to re-audit each implementation independently.

Assume separate audits already establish how each mechanism works.

The purpose of this investigation is to determine:

- whether the two mechanisms can safely operate together;
- whether they protect the same or different resources;
- whether their interaction causes redundant concurrency control;
- whether one mechanism can block while consuming capacity from the other;
- whether their combined operation can introduce unnecessary latency, reduced throughput, goroutine accumulation, memory overhead, or operational complexity;
- whether both mechanisms should remain enabled together.

This is strictly a **static code investigation**.

Do not:

- run odnd;
- execute tests;
- perform benchmarks;
- perform load tests;
- collect runtime metrics;
- modify application code.

Where actual performance impact cannot be proven statically, state this explicitly.

Use:

**Potential performance consequence**

rather than presenting inferred behavior as measured degradation.

---

# Primary Questions

Answer each independently.

## Q1. Can the two mechanisms operate together correctly?

Determine whether their acquisition, rejection, and release semantics are compatible.

## Q2. Do they protect the same resource?

Determine precisely where responsibilities overlap and where they differ.

## Q3. What is the ordering between them?

Determine which gate a request encounters first.

## Q4. Can a request hold capacity in one mechanism while waiting on the other?

This is one of the most important questions in the investigation.

## Q5. Does combining the mechanisms introduce redundant blocking or rejection?

## Q6. Can the combination reduce effective concurrency below either configured limit?

## Q7. Can the combination increase request latency or tail latency?

## Q8. Can the combination increase goroutine retention?

## Q9. Can retries amplify the interaction?

## Q10. Does one mechanism make part of the other unnecessary?

## Q11. Does removing one eliminate behavior that the other cannot reproduce?

## Q12. Based strictly on code behavior, should they

- remain enabled together;
- have coordinated configuration;
- have mutually exclusive operation;
- be consolidated;
- have one disabled;
- require further runtime validation before a decision?

---

# Starting Points

For `BackendConcurrencyLimiter`, inspect:

- `proxy/concurrency/backend_concurrency.go`
- `proxy/reqflow/node_origin_fetch.go`
- `shared/concurrency/multi_limiter.go`
- `shared/concurrency/limiter.go`

For Fetch Semaphore, inspect:

- `originconfig/fetch.go`
- `originconfig/host.go`
- `shared/TimeoutSemaphore.go`
- `shared/http_retry.go`

Also inspect:

- `proxy/handlers.go`
- relevant origin fetch path;
- transport setup;
- connection limits;
- retry behavior;
- fallback-host behavior;
- stale-serving behavior;
- origin concurrency controls;
- configuration.

---

# 1. Build One Combined Request Flow

Trace a single request through both mechanisms.

Determine exact ordering.

The diagram must show something similar to:

`request`
→ `backend/origin selection`
→ `BCL acquisition or Fetch Semaphore acquisition`
→ `other concurrency control`
→ `backend attempt`
→ `retry decision`
→ `retry semaphore`
→ `fetch semaphore`
→ `backend retry`
→ `release`

Do not assume this ordering.

Prove it from code.

Create a Mermaid diagram showing:

- every concurrency gate;
- every blocking point;
- every immediate rejection point;
- when capacity is acquired;
- when it is released;
- retry behavior.

---

# 2. Compare the Controlled Unit

For both mechanisms identify:

| Property             | BackendConcurrencyLimiter | Fetch Semaphore |
| -------------------- | ------------------------- | --------------- |
| Unit limited         |                           |                 |
| Scope                |                           |                 |
| Key                  |                           |                 |
| Acquisition point    |                           |                 |
| Release point        |                           |                 |
| Blocking?            |                           |                 |
| Queue/wait?          |                           |                 |
| Timeout?             |                           |                 |
| Immediate rejection? |                           |                 |
| Retry budget?        |                           |                 |
| Client response      |                           |                 |

Determine whether both limits apply to:

- the same backend operation;
- different stages;
- different identities;
- different attempts.

---

# 3. Compare Keying

Determine the relationship between:

`b.OriginBackend()`

and:

`backendHost`

Answer:

- Are they always identical?
- Can multiple `backendHost` values map to one BCL key?
- Can one backendHost correspond to multiple BCL keys?
- Do ports affect one but not the other?
- Do fallback or dynamically resolved backends change one key but not the other?

If equivalence cannot be proven, mark it:

**Unverified**

and explain the code or data needed.

---

# 4. Determine Acquisition Ordering

Establish which mechanism is reached first.

Then determine whether the first mechanism's capacity remains occupied while the request interacts with the second.

Example concern:

If BCL capacity is incremented before a request waits on Fetch Semaphore, then a request waiting for the semaphore may consume BCL capacity without actively using the backend.

Do not assume this happens.

Trace it.

Likewise determine whether a semaphore permit can be held while the request waits on another limiter.

---

# 5. Analyze Effective Concurrency

Suppose:

`BCL limit = X`

and:

`Fetch Semaphore limit = Y`

Determine from the code what effective maximum concurrency can be.

Analyze:

### X < Y

### X > Y

### X == Y

Determine:

- which control dominates;
- whether the higher limit becomes irrelevant;
- whether requests queue behind one mechanism only to be rejected by another;
- whether usable backend concurrency can be lower than `min(X, Y)` because of lifecycle ordering.

Do not invent numeric runtime results.

---

# 6. Analyze Blocking + Fail-Fast Interaction

The mechanisms appear to have different overload semantics.

Verify:

### BackendConcurrencyLimiter

Immediate rejection when capacity is exhausted.

### Fetch Semaphore

Waits up to a timeout.

Analyze what happens when they are combined.

Potential patterns to prove or disprove:

### Pattern A

Request passes BCL, then waits on Fetch Semaphore.

Possible consequence:

BCL capacity may be occupied by waiting work.

### Pattern B

Request waits on semaphore, then reaches BCL and gets rejected.

Possible consequence:

The request waits before ultimately being rejected.

### Pattern C

Retry waits on retry semaphore and fetch semaphore while another concurrency permit remains held.

Determine actual behavior from code.

---

# 7. Analyze Latency Implications

Identify every scenario where using both mechanisms can add waiting compared with using one.

Trace:

- initial fetch;
- retry;
- cancellation;
- timeout;
- fallback;
- alternate backend.

Determine whether requests can experience:

- semaphore wait;
- retry backoff;
- retry semaphore wait;
- fetch semaphore wait;
- eventual BCL rejection.

If possible, derive theoretical maximum waiting time from configuration.

Label it as:

**Configured worst-case wait**

not measured latency.

---

# 8. Analyze Throughput Implications

From static design, determine whether both controls can unnecessarily reduce useful backend concurrency.

Look for:

- permits consumed while waiting;
- nested limits;
- mismatch in configured capacities;
- unrelated requests blocked by shared state;
- retries competing with initial requests;
- double enforcement of the same backend work.

Separate:

**Structural throughput limitation**

from:

**Measured throughput impact**

The second cannot be established through static review.

---

# 9. Analyze Goroutine and Memory Implications

Determine whether combining the mechanisms can increase:

- number of blocked goroutines;
- timer objects;
- semaphore state;
- limiter state;
- backend-key maps;
- metric cardinality.

Pay special attention to requests that:

- pass one mechanism;
- wait on another;
- are canceled while waiting;
- retry multiple times.

---

# 10. Analyze Retry Behavior

Determine how retries interact with both mechanisms.

Answer:

1. Does BCL count each retry as a separate backend operation?
2. Does a retry acquire `retrySema`?
3. Does a retry also acquire `fetchSema`?
4. Does it reacquire BCL capacity?
5. In what order?
6. Can retries consume capacity from all three controls?
7. Can a retry wait while holding a permit from another control?

Create a dedicated retry sequence diagram if necessary.

---

# 11. Analyze Failure Semantics

Compare:

### BCL exhaustion

Potentially immediate 429.

### Fetch Semaphore exhaustion

Potentially timeout and another status/error path.

Determine what happens if both are enabled.

Can identical backend overload result in different client responses depending on which control fires first?

Analyze whether this creates:

- inconsistent status codes;
- inconsistent stale behavior;
- inconsistent fallback behavior;
- inconsistent metrics.

---

# 12. Analyze Operational Complexity

Determine whether running both mechanisms introduces configuration complexity.

Identify all relevant limits:

- BCL limit;
- fetch semaphore maximum;
- fetch semaphore timeout;
- retry semaphore percentage;
- retry semaphore timeout;
- related origin concurrency controls.

Explain whether operators must understand relationships between these values to avoid contradictory configurations.

Do not make a recommendation based only on simplicity.

Preserve behavior where it provides distinct protection.

---

# 13. Determine Whether the Controls Are Redundant

Classify each capability as:

### Fully overlapping

Both mechanisms provide materially equivalent protection.

### Partially overlapping

Both constrain similar work but with different semantics or scope.

### Non-overlapping

Only one mechanism provides the behavior.

Include at least:

- active backend-attempt cap;
- backpressure;
- immediate load shedding;
- retry budget;
- cancellation behavior;
- per-backend isolation;
- timeout behavior;
- stale/fallback behavior;
- metrics.

---

# 14. Performance Degradation Assessment

Provide a dedicated section:

# Can Running Both Mechanisms Cause Performance Degradation?

Separate conclusions into three levels.

## Verified from code

Examples:

- additional lock operation;
- additional map lookup;
- additional semaphore acquisition;
- request can wait at a second gate;
- additional timer allocation.

## Potential runtime impact

Examples:

- increased tail latency;
- lower throughput;
- additional goroutines waiting;
- more lock contention.

Only include consequences that logically follow from the implementation.

## Cannot be established statically

Examples:

- actual p99 increase;
- percentage throughput loss;
- CPU overhead;
- memory delta;
- saturation point.

Do not fabricate these values.

---

# 15. Decision Matrix

Create a table for:

| Configuration        | Protection | Blocking | Load Shedding | Retry Budget | Potential Concern |
| -------------------- | ---------- | -------- | ------------- | ------------ | ----------------- |
| Both enabled         |            |          |               |              |                   |
| BCL only             |            |          |               |              |                   |
| Fetch Semaphore only |            |          |               |              |                   |
| Both disabled        |            |          |               |              |                   |

Explain what is gained and lost in each configuration.

---

# 16. Recommendation

Provide an architecture recommendation based on code evidence.

Possible conclusions include:

- retain both;
- retain both but coordinate their limits;
- retain BCL and redesign retry protection;
- retain Fetch Semaphore and remove BCL;
- disable one behind configuration;
- redesign their acquisition ordering;
- further runtime validation is required before removal.

Do not force a predetermined outcome.

The recommendation must state:

### Recommended state

### Why

### Protections preserved

### Protections removed

### Required code changes

### Configuration implications

### Risks

### Confidence

### Conditions that would change the recommendation

---

# 17. Top Four Interaction Findings

Identify the four most important issues specifically caused by, or exposed by, the interaction between the mechanisms.

Examples may include:

- holding one permit while waiting on another;
- duplicated concurrency enforcement;
- contradictory overload responses;
- retry amplification;
- mismatched keying;
- inconsistent configuration;
- avoidable hot-path overhead.

Do not include issues that belong solely to one mechanism unless they materially affect interaction.

For each provide:

- Finding;
- Severity;
- Evidence;
- Trigger;
- Current behavior;
- Performance or correctness implication;
- Recommendation;
- Confidence.

---

# Required Report

Save to:

`@Torbit/Odnd_Load_Testing/<ID>/bcl_fetch_semaphore_interaction_audit.md`

Structure:

## Executive Summary

## 1. Combined Request Flow

## 2. Mechanism Comparison

## 3. Keying Comparison

## 4. Acquisition and Release Ordering

## 5. Effective Concurrency

## 6. Blocking vs Load-Shedding Interaction

## 7. Retry Interaction

## 8. Cancellation and Timeout Interaction

## 9. Client Response, Stale and Fallback Behavior

## 10. Memory and Goroutine Implications

## 11. Performance Degradation Assessment

## 12. Configuration Interaction

## 13. Redundancy vs Complementary Behavior

## 14. Four-Configuration Decision Matrix

## 15. Top Four Interaction Findings

## 16. Recommendation

## 17. Unverified Questions

## 18. Review Coverage

---

# Evidence Rules

Every factual conclusion must cite:

`path/to/file.go` + symbol + line range.

Use GitHub links when available.

Clearly distinguish:

**Verified behavior**

**Architectural inference**

**Potential performance consequence**

**Unverified**

Never present static analysis as benchmark evidence.

The final document must be simple enough for the team to read quickly, but technically rigorous enough that every conclusion can be challenged against a specific code path.

