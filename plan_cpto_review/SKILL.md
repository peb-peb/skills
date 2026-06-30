---
name: cpto-doc
description: >-
  Rewrite or draft any document — PRD, technical design, RFC, ops runbook,
  strategy memo, status update — into concise, jargon-free, decision-ready prose
  for a CPTO / executive technical audience. Use whenever the user writes, edits,
  tightens, or asks to "make concise" a doc, or invokes /cpto-doc.
---

# CPTO Doc Style

Turn any document into something a time-poor technical executive can act on in one read. Apply these rules every time you write or revise a doc under this skill.

## Audience model

The reader is a CPTO. They read to decide, not to admire prose. They want, in order:

1. The ask or recommendation.
2. Why it matters now (impact, cost of inaction).
3. Options and tradeoffs.
4. Effort, cost, risk, timeline.
5. What happens next and who owns it.

## Do

- Lead with the bottom line. Ask or recommendation in the first 2 sentences.
- Keep every fact, number, date, name, link, and constraint from the source.
- One idea per sentence. Short, declarative. Concrete nouns/verbs over abstract.
- One clear heading per section where it fits.
- State tradeoffs, risks, and what you are explicitly NOT doing.

## Cut

- Buzzwords: leverage, synergize, holistic, paradigm, "in order to", "it is important to note".
- Throat-clearing and hedging.
- Repetition and sentences that restate a heading.
- Adjectives and adverbs that carry no information.

## Never

- Do not invent facts, numbers, or conclusions not in the source. Missing fact → write `[TODO: <what's needed>]`.
- Do not change technical meaning. If simplifying risks accuracy, keep the precise term and define it once inline.

## Length

Cut at least 30% of word count unless that removes real information.

## Output

1. The rewritten (or drafted) document.
2. A 2–3 line **What changed** note: what you cut, and any `[TODO]`s the CPTO must fill.

## Self-check before returning

- [ ] Ask/recommendation in the first 2 sentences.
- [ ] No word from the Cut list survives.
- [ ] Every source number, date, and name is preserved.
- [ ] No invented facts; gaps marked `[TODO]`.
- [ ] ≥30% shorter, or a one-line explanation of why not.
