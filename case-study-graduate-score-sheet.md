# Interviewer Score Sheet — Graduate Case Study (SmartSearch)

| | |
|---|---|
| **Candidate** | |
| **Date** | |
| **Interviewer(s)** | |
| **Role** | Junior Developer / Graduate Engineer |

> Use this sheet live. Capture **specific things the candidate said** ("identified the versioning sub-problem unprompted") rather than generic labels ("good communication"). The candidate sees only the brief — the prompts and signals here are for you.

---

## Pacing (tick as you go)

- [ ] Reading + ground rules (~5 min)
- [ ] Part 1 — System design (15–20 min)
- [ ] Part 2 — Debugging (15–20 min)
- [ ] Part 3 — Data model (12–15 min)
- [ ] Wrap-up / candidate questions (~5 min)

---

## Part 1 — System Design

**Prompt:** *"Walk me through how you'd think about building this system. Don't worry about specific technologies — I'm interested in how you break down the problem."*
Escalators available: the "120-page document" complaint · versioning of 15,000 docs · the Option A (pre-index, fast) vs Option B (on-the-fly, slow) trade-off under a one-month deadline.

**Signals**

- [ ] Identifies sub-problems unprompted (ingestion, search/matching, answer extraction, versioning, UI)
- [ ] Recognises keyword vs meaning-based search gap
- [ ] Reasons about breaking big docs into smaller pieces (chunking, by intuition)
- [ ] Considers the user experience of the answer (paragraph? link? both?)
- [ ] Makes a clear trade-off recommendation **with reasoning** (or proposes a hybrid)
- [ ] Asks clarifying questions (formats? languages? per-user access?)

**Quick read:**  ☐ Strong   ☐ Adequate   ☐ Weak

**Observations / verbatim quotes:**

_______________________________________________________________

_______________________________________________________________

_______________________________________________________________

---

## Part 2 — Debugging and Investigation

**Setup:** System live for 3 months. Finance reports SmartSearch answered *"approval limit for purchase orders is €5,000"* — but the current policy says €10,000; the €5,000 version was superseded 6 months ago. Happens repeatedly.
Escalators available: why the old answer persists even if marked superseded · the two phases (ingest vs query) · systematic check order across logs/DB/index · "both versions are indexed — why does the old one win?" · the delete-old vs keep-history design debate.

**Signals**

- [ ] Generates multiple hypotheses before committing (old doc still indexed / new doc never ingested / both indexed but old ranks higher / index not refreshed)
- [ ] Proposes a systematic investigation order, not random guessing
- [ ] Reasons about *why* the older doc might rank higher (age, click history, wording match, no date weighting)
- [ ] Treats delete-vs-keep as a design decision (e.g. keep but tag and de-prioritise, with a historical-search option)
- [ ] Engineering mindset: "is this systemic or a one-off? how many other docs are affected?"

**Quick read:**  ☐ Strong   ☐ Adequate   ☐ Weak

**Observations / verbatim quotes:**

_______________________________________________________________

_______________________________________________________________

_______________________________________________________________

---

## Part 3 — Data Model

**Setup:** Get concrete about how the system stores its information. Technology-agnostic — no SQL or specific product. *"What are the main things the system stores, and how do they relate?"* Boxes and arrows are fine.
Escalators available: what records get created for one 120-page doc · how the model represents version supersession + which is current (ties to Part 2) · where chunks live and how an answer traces back to the right place in the right version · how access/permissions are represented and what that means at search time · new-record-per-version vs update-in-place trade-off · storing text once vs duplicated in chunks · a document spanning/moving between departments.

**Signals**

- [ ] Identifies core entities unprompted (documents, versions, chunks/sections, users, access groups/permissions)
- [ ] **Separates document from version**; models "current" explicitly (flag, pointer, or version + effective dates)
- [ ] Traces a source reference down: answer → chunk → version → document (precise *and* version-correct)
- [ ] Represents access as a relationship (users → groups → documents/departments), filters at search time
- [ ] States a trade-off with reasoning (new-record vs in-place; normalize vs duplicate)
- [ ] Closes the loop with Part 2: data model is where the "wrong version" bug is prevented
- [ ] Asks clarifying questions (multi-department docs? keep history forever? per-doc or per-folder permissions?)

**Quick read:**  ☐ Strong   ☐ Adequate   ☐ Weak

**Observations / verbatim quotes:**

_______________________________________________________________

_______________________________________________________________

_______________________________________________________________

---

## Scoring

Score each competency 1–4. Multiply by weight to get the weighted points, then sum.

| Competency | Weight | Score (1–4) | Weighted (score × weight) |
|-----------|:------:|:-----------:|:-------------------------:|
| **Analytical thinking** | ×0.30 | | |
| **Problem-solving under ambiguity** | ×0.25 | | |
| **Data modeling & technical foundation** | ×0.30 | | |
| **Communication** | ×0.15 | | |
| | | **Weighted total (1–4):** | |

**Scale reference:** 1 — Insufficient · 2 — Developing · 3 — Meets expectations · 4 — Exceeds expectations

<details>
<summary>Level descriptors (expand if needed)</summary>

- **Analytical thinking** — 1: can't decompose, needs heavy guidance · 2: obvious sub-problems with prompting, shallow · 3: breaks down independently, 2–3 hypotheses before committing · 4: structures without prompting, non-obvious connections, probing questions, systemic thinking.
- **Problem-solving under ambiguity** — 1: freezes on incomplete info · 2: assumes but doesn't state it · 3: states assumptions, reasonable approaches, adapts to new info · 4: comfortable with ambiguity, separates known/unknown, reduces uncertainty, adjusts fluidly.
- **Data modeling & technical foundation** — 1: one flat structure, no versions/chunks/relationships, no feel for data flow · 2: some entities with prompting, conflates document and version, can't connect storage/search/retrieval · 3: separates document from version and models "current", reasonable data-flow model, access as a relationship · 4: clean entity model with version-correct source refs, reasons about trade-offs and failure modes unprompted, links back to the Part 2 bug.
- **Communication** — 1: disorganised, heavy jargon or hand-waving · 2: coherent but unstructured, hard to follow at points · 3: clear and structured across all parts, explains reasoning as they go · 4: concise, good analogies and structure, makes the interviewer's job easy.

</details>

**Overall recommendation** (based on weighted total scaled to a 4–16 raw band, or use the weighted 1–4 average):

- [ ] **Strong hire** — 12–16 raw / ~3.0+ weighted. Analytical and communication skills to grow into the role.
- [ ] **Potential hire** — 8–11 raw / ~2.0–2.9 weighted. Needs mentoring; enough foundation. Judge in context.
- [ ] **Below bar** — 4–7 raw / <2.0 weighted. Significant gaps that would make onboarding difficult.

---

### Reminders

- A candidate who asks great clarifying questions and only gets halfway through Part 1 can outscore one who rushes through everything shallowly.
- We're not looking for "correct" answers — we want structured thinking, curiosity, and clear communication.
- If the candidate is visibly nervous, spend an extra minute on the ground rules.

**Summary note / decision rationale:**

_______________________________________________________________

_______________________________________________________________
