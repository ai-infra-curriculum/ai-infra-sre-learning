# Exercise 04 — Dickerson's Hierarchy Applied to AI Workloads

**Estimated effort:** 2 hours · **Output:** one Markdown file — the layer-by-layer AI-mapping + a gap-analysis matrix for a chosen platform

## Objective

Produce a layer-by-layer mapping of Dickerson's hierarchy onto a specific AI platform you can name — an audit that says, for each layer of the pyramid, which practices are in place, which are partially in place, and which are missing. Use the audit to justify a specific investment plan for the next two quarters.

The chapter-03 presentation is general. The exercise asks you to apply it to one platform and *name* the gaps.

## Prerequisites

- Chapter 03 (Dickerson's hierarchy applied to AI reliability) in this module.
- Enough familiarity with a concrete AI platform to describe its monitoring, incident response, post-mortem culture, release pipeline, capacity-planning posture, and product-dev ritual honestly. Reuse the team context from the earlier exercises.

## Problem statement

Your VP of Engineering has asked you to deliver a two-page document at the next planning review:

> *"What's the state of reliability on our AI platform? Where should we invest next?"*

You could answer this by listing projects or by venting about on-call pain, but the discipline Dickerson's hierarchy gives you is to answer in the *order* the pyramid names — monitoring, then incident response, then post-mortem, then testing + release, then capacity, then product-dev, then UX. Investing above a layer that is broken wastes the investment; investing below a layer that is done over-saturates it. This audit does the layering.

## Requirements

Produce a single Markdown file with the following sections.

1. **Platform context.** One paragraph naming the platform, its surfaces, its consumers, and its team size. Same conventions as the earlier exercises.
2. **The hierarchy, annotated per-layer.** Seven subsections, one per layer:
   1. Monitoring
   2. Incident response
   3. Post-mortem / root-cause culture
   4. Testing and release procedures
   5. Capacity planning
   6. Product development
   7. UX
   For each layer, answer three questions:
   - What is the classical content? (one line)
   - What is the AI-specific content? (two to four lines; use chapter 03 as reference)
   - What is the state of this layer on your platform? (one of: *in place*, *partial*, *missing*, with a two-sentence justification)
3. **Gap-analysis matrix.** A table with one row per layer and three columns: *layer*, *state*, *top-one-gap*. The top-one-gap for each layer is the single highest-leverage thing you would fix at that layer if you had one engineer-month next quarter.
4. **Investment plan.** Walking the pyramid bottom-up, propose investments for the next two quarters. The rule: do not propose a capacity-planning investment (layer 5) if layer 1 or 2 is *missing*. Do not propose a product-dev investment (layer 6) if layer 4 is missing. Explicitly justify any investment that violates the strict bottom-up order (there are legitimate exceptions — e.g., a regulator deadline for a DR plan — but the exception has to be named).
5. **Module-sequence sanity-check.** Match your investment plan against the curriculum module order (`mod-102` → `mod-114`). If your platform's gaps suggest a different sequence (e.g., chaos before capacity, because the upstream provider's reliability has moved), name the deviation and the reason.

## Starter guidance

- Be honest about layer state. A platform claiming *in place* for every layer is almost certainly not auditing honestly. In practice, most platforms have partial coverage at layers 3 and 4 and missing coverage at layer 5 or 7.
- The hardest layer to be honest about is layer 3 (post-mortem culture). A team that *has* post-mortem templates but does not *follow through* on action items is at *partial*, not *in place*. The signal is whether last quarter's post-mortem remediation items are closed.
- The *top-one-gap* column is the most valuable artifact in this exercise. Do not propose three things per layer; propose one. The constraint of picking one forces you to actually prioritise.
- For the investment plan, map each proposed investment to a specific module in this curriculum. Investments that map to no module are either outside the role scope (route back to a peer) or are a signal that a future module should exist.

## Acceptance criteria

- Markdown file is under 1500 words.
- All seven Dickerson layers are addressed, in order.
- Each layer has a stated state (*in place* / *partial* / *missing*) with a two-sentence justification.
- Gap-analysis matrix has one top-one-gap per layer.
- Investment plan proposes at least four and at most eight investments across two quarters.
- Any investment violating the strict bottom-up pyramid order is explicitly justified.
- Each proposed investment maps to a specific module in this curriculum (or is marked as route-to-peer).
- A reader unfamiliar with the platform leaves able to answer the VP's question in one paragraph.

## Stretch goals

- **Second-platform comparison.** Repeat the audit for a second platform (a sibling team at the same company, an open-source project you follow, or a published-engineering-blog case study). Compare the two audits — where they agree on the hardest gaps, and where the two platforms' contexts make them diverge.
- **Layer-skipping failure mode.** Describe (from memory or from a public post-mortem) an incident caused by investing in a higher layer while a lower one was broken. The archetype is "we had great dashboards but no alert policy, so nobody woke up for the degradation" (investment in layer 1's visualisation side with layer 2's paging side missing). Write it up as a short case study.
- **Next-quarter OKR draft.** Convert the investment plan into three OKRs for next quarter, with measurable key results. Make sure each KR is traceable back to a layer of the pyramid and a module of this curriculum.
