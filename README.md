# Elected Officials Data Acquisition

Senior Technical Analyst technical assessment — **Murmuration**. This repository contains work for building and maintaining a dataset of elected officials across U.S. county governments (e.g., County Commissioner, County Executive, County Clerk, Sheriff, and similar roles).

## Time & scope

| Part | Focus | Estimated time |
|------|--------|----------------|
| **Part 1** | Design & planning | ~1.5–2 hours |
| **Part 2** | *(Provided after Part 1 review)* | ~2–3 hours |

**Total:** about 4–5 hours. The goal is not perfection; reviewers want to see how you think, trade off options, and move from a plan to working code.

## Repository layout

| Path | Description |
|------|-------------|
| `docs/README.md` | Dimensional field model, source strategy, collection approach |
| `docs/part1-design.md` | Part 1 design document (draft — revise before submit) |
| *(TBD)* | Part 2 executable code, tests, or notebooks — *structure after materials arrive* |

Adjust paths above as you add files. Keep this README updated with **how to run** anything executable (environment, commands, inputs/outputs).

## Part 1 deliverable (design document)

Address the following in a markdown (or similar) document:

1. **Data model** — Database or flat-file schema: entities, fields, relationships; how to handle variability across states and counties (not every county has the same offices).
2. **Source strategy** — Candidate sources; what makes a source trustworthy or preferable; expected gaps and mitigations.
3. **Collection approach** — High-level architecture to collect and keep data current; reasoning, not necessarily a full implementation.
4. **Tradeoffs & open questions** — What you would do with more time; assumptions; known weaknesses.

**What “good” looks like:** thoughtful modeling for real-world variability, clear source judgment, validation mindset, readable reasoning for non-technical teammates, and sensible scoping for the time box.

## Part 2

Materials and instructions are sent after Part 1 passes review. Update this README with run instructions and any dependencies when Part 2 is in the repo.

## AI usage policy

AI-assisted tools are **encouraged**, provided the submission reflects **your** understanding.

For **each exercise part**, include a short note (1–3 sentences) covering:

- Where, if at all, you used AI assistance.
- How you verified or changed AI-generated suggestions.
- Which parts you wrote entirely on your own.

A convenient place is a subsection at the end of `docs/part1-design.md` (and similarly for Part 2), or a single `docs/ai-usage-notes.md` if you prefer one file.

## Submission

- Submit as a **GitHub repository** or **`.zip`**, with this README explaining how to run anything executable.
- Send to **May Multari** at [may.multari@murmuration.org](mailto:may.multari@murmuration.org) **within 24 hours** of receiving the exercise.

## Context (scenario)

County government structure varies by state; official sites range from excellent to absent; there is no single national authoritative source. The design should reflect that ambiguity and real-world messiness.
