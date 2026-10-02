Redraft `concurrency_design_review.md` so it sounds like a real engineer speaking to the team during a design review, not like an AI-generated report.

The current version is too formal, too polished, and too long in places.

Keep the technical conclusions and evidence, but rewrite the document with a more natural, direct, human tone.

## Goals

- Make it sound like I am explaining the findings to my team.
- Use simple, clear language.
- Keep the technical rigor.
- Shorten long paragraphs.
- Get to the point quickly.
- Remove repetitive explanations.
- Remove unnecessary formal or academic wording.
- Keep the reasoning easy to follow.
- Preserve first-person narration.

Use language like:

- “What I found is…”
- “The main difference here is…”
- “This matters because…”
- “The problem with the current flow is…”
- “I would fix this first…”
- “I don’t think we should remove this yet because…”
- “From the code, this is what is happening…”

Avoid language like:

- “It is important to note that…”
- “The analysis indicates…”
- “This section evaluates…”
- “From an architectural perspective…”
- “The mechanism provides a comprehensive…”
- “It can therefore be concluded that…”

## Paragraph Style

Keep paragraphs short.

Most paragraphs should be **1–3 sentences**.

Do not write large blocks of text when the same point can be made in a few direct sentences.

For example, instead of:

> The BackendConcurrencyLimiter and Fetch Semaphore both provide mechanisms for controlling concurrent backend operations, but they differ significantly in their enforcement semantics and the operational behavior that occurs when their respective configured limits are reached.

Write:

> Both mechanisms limit backend work, but they behave differently when the limit is reached. BCL rejects immediately. The Fetch Semaphore waits for capacity.

Prefer that style throughout the document.

## Keep the Document Focused

The design review should answer these questions quickly:

1. What does BCL do?
2. What does the Fetch Semaphore do?
3. Are they doing the same thing?
4. What happens when both are enabled?
5. What problems did I find?
6. What should we fix?
7. What should we keep, disable, or remove?
8. What do I recommend the team do next?

If a paragraph does not help answer one of these questions, remove or shorten it.

## Technical Precision

Do not simplify away important distinctions.

Preserve:

- concurrency limiting vs rate limiting;
- fail-fast rejection vs waiting/backpressure;
- initial fetch concurrency vs retry concurrency;
- backend identity vs backend host;
- request cancellation behavior;
- permit lifecycle;
- stale/fallback behavior;
- static-analysis findings vs measured production behavior.

Do not make new claims.

Use only the findings already established in `concurrency_design_review.md` and its source audit documents.

## Recommendations

Make the recommendation sections especially direct.

Instead of vague wording like:

> Consider making semaphore acquisition context-aware.

Use:

> I would make semaphore acquisition context-aware so canceled requests stop waiting immediately.

Instead of:

> Additional investigation may be necessary before removing the Fetch Semaphore.

Use:

> I would not remove the Fetch Semaphore yet. BCL does not currently replace the retry budget behavior.

Only keep statements that are supported by the audit evidence.

## Findings Format

For each issue, keep the format simple:

### Issue: `<name>`

**What I found**

2–4 short sentences.

**Why it matters**

1–3 short sentences.

**What I would change**

1–3 short sentences.

Avoid long background sections unless they are necessary to understand the issue.

## Tables

Keep useful comparison tables, but remove columns or rows that do not help the decision.

Tables should make the design easier to understand, not make the document look more formal.

## Final Recommendation

End with a short conclusion that sounds like something I could say in the meeting.

For example:

> My recommendation is to fix the correctness issues first and keep both mechanisms in place for now. I would not remove the Fetch Semaphore until we prove that the retry protection and waiting behavior are either unnecessary or replaced somewhere else.
>
> After that, we can validate whether BCL alone gives us the backend protection we need and simplify the design if the evidence supports it.

Do not copy this conclusion unless it matches the actual findings.

## Output

Rewrite the existing:

`concurrency_design_review.md`

Keep the same technical meaning, but make it:

- shorter;
- clearer;
- more conversational;
- easier to present verbally;
- less repetitive;
- less formal;
- more like a senior engineer talking to the team.

The final document should feel like a **design review I wrote after studying the code**, not a generated research report.
