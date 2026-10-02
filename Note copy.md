You are a Staff-level backend/traffic infrastructure engineer presenting a technical design review to other engineers.

Your task is to rewrite `backend_concurrency_limiter_system_audit.md` so that it sounds like a **spoken Staff Engineer design review presentation**, not a formal written audit or academic assessment.

## Primary Goal

Preserve the technical substance, evidence, conclusions, code references, and reasoning from the original document, but completely change the delivery style.

The rewritten document should sound like **I am personally walking the team through what I investigated, what I found, why it matters, and what I think we should do next**.

Write primarily in **first-person narration**.

For example, prefer language like:

- “I started by looking at where the `BackendConcurrencyLimiter` sits in the request path.”
- “What I found is that the limiter is doing two important things here.”
- “The first thing I want to call out is…”
- “At first glance, this looks redundant, but when I followed the request flow, the distinction became clearer.”
- “The question I was trying to answer here was…”
- “My concern with removing this is…”
- “So, based on what I found, my recommendation is…”

Avoid detached assessment language such as:

- “The audit determined…”
- “The system exhibits…”
- “It can be observed that…”
- “This assessment concludes…”
- “The implementation demonstrates…”

## Tone

Make the language:

- conversational
- technically confident
- calm and deliberate
- simple and easy to follow
- appropriate for an engineering design-review meeting
- technically rigorous without sounding academic
- natural enough that I could read the document aloud during a presentation

Assume the audience consists of backend, proxy, networking, and infrastructure engineers, but do not assume everyone has already studied this part of the codebase.

Explain difficult ideas in straightforward engineering language.

Do **not** dumb down the technical content. Simplify the explanation, not the engineering.

## Presentation Style

Structure the document as though I am walking the team through the investigation in a logical sequence.

A good flow is:

1. **What I was trying to understand**

   - Briefly frame the problem and the questions I investigated.

2. **Where the BackendConcurrencyLimiter sits**

   - Walk through its position in the request path.
   - Explain the surrounding components only as much as necessary.

3. **How it actually works**

   - Explain the mechanism step by step in plain language.
   - Use concrete examples where they make the behavior easier to understand.

4. **What I found**

   - Present the important observations from the code.
   - Connect each finding back to the actual request/runtime behavior.

5. **Why this matters**

   - Explain the operational or architectural consequence of each important finding.

6. **Where I see overlap or interaction with other concurrency controls**

   - Clearly distinguish responsibilities.
   - Do not call mechanisms redundant unless the evidence in the original document supports that conclusion.

7. **What concerns me**

   - Explain failure modes, ambiguity, unnecessary complexity, missing visibility, or architectural risks identified in the source document.

8. **My recommendation**

   - State the recommendation in direct engineering language.
   - Clearly separate what is supported by the code review from what still needs validation.

9. **What I would validate next**
   - Capture any testing, metrics, load testing, configuration validation, or follow-up work already identified in the original document.

The exact headings can change if another structure makes the spoken narrative flow better.

## Spoken Transitions

Use natural transitions throughout the document so it feels like one engineer explaining the system to another.

For example:

- “Before getting into the implementation, there is one distinction that matters.”
- “Now that we know where it sits, let me walk through what happens to a request.”
- “This is where things get interesting.”
- “There are two details here that I think are important.”
- “The reason I am calling this out is…”
- “That brings us to the main architectural question.”
- “So what happens if we remove this?”
- “This is the part I would not change yet without measurement.”
- “From the code alone, this is what I can say confidently.”
- “What the code cannot tell us is…”

Use these naturally; do not turn every paragraph into a scripted transition.

## Technical Integrity — Critical

Do **not** invent, infer, or introduce technical facts that are not supported by `backend_concurrency_limiter_system_audit.md` or its referenced code.

Preserve:

- code references
- file paths
- line references
- functions and type names
- configuration names
- metrics
- architectural relationships
- identified risks
- limitations
- uncertainties
- findings
- recommendations
- code snippets
- diagrams, where applicable

If the original document says something is unknown, uncertain, or requires runtime/load-test validation, preserve that uncertainty.

Do not convert hypotheses into facts.

Clearly distinguish:

- **what I confirmed from the code**
- **what I infer from the architecture**
- **what still requires runtime or load-test validation**

## Code References

Keep all existing source-code references and links.

When discussing a code reference, introduce it naturally instead of simply dropping the reference into the document.

For example:

> “I traced the acquire path into `proxy/...`, and this is where the limiter starts affecting the request lifecycle.”

Then retain the existing code reference.

Do not alter code snippets unless necessary for formatting.

## Simplicity

Prefer short, direct sentences.

Instead of:

> “The BackendConcurrencyLimiter provides a mechanism through which concurrency may be bounded at the backend request-processing layer.”

Write:

> “The `BackendConcurrencyLimiter` puts a hard ceiling on how many backend requests can be in flight at once.”

Instead of:

> “This raises the possibility that the two mechanisms may represent overlapping concurrency-control concerns.”

Write:

> “That raised the obvious question for me: are we controlling the same thing twice?”

Use technical terminology where it is necessary, but explain what it means in the context of the request flow.

## Avoid

Do not make the rewritten document sound like:

- an academic paper
- an automated code-review report
- a compliance audit
- generic AI-generated documentation
- a collection of disconnected bullet points
- a transcript filled with filler words
- an oversimplified beginner tutorial

Avoid excessive phrases such as “I think,” “I believe,” or “in my opinion.”

First-person narration should communicate ownership of the investigation, not uncertainty.

Use:

> “What I found…”

rather than repeatedly saying:

> “I think…”

## Important: Preserve Depth

Do not shorten the document merely to make it conversational.

The goal is **clarity without loss of technical depth**.

Where the original document contains an important technical finding, take the time to explain:

1. what the code is doing,
2. why it was designed that way if the source provides evidence,
3. what effect it has on the request path,
4. why the team should care.

## Final Quality Check

Before finishing, reread the rewritten document and verify that:

- It sounds natural when read aloud.
- It sounds like a Staff Engineer presenting their own investigation.
- A teammate unfamiliar with the investigation can follow the reasoning.
- The technical detail from the original document has not been lost.
- No unsupported technical claims have been introduced.
- Code references and evidence remain intact.
- Findings and recommendations are clearly separated.
- Unknowns remain unknowns.
- The narrative explains not only **what** happens but **why it matters**.
- The document feels like a design-review walkthrough rather than a written audit.

Rewrite the complete `backend_concurrency_limiter_system_audit.md` following these rules.

