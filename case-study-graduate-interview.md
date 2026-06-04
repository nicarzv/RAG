# Case Study Exercise — Graduate Technical Interview

**Role:** Junior Developer / Graduate Engineer  
**Duration:** 50–65 minutes live session  
**Format:** Candidate receives the scenario document at the start of the session. No preparation needed. Interviewer guides the conversation through three parts.  
**What we're testing:** Analytical thinking, problem-solving under ambiguity, technical reasoning (algorithm design), communication clarity  
**What we're NOT testing:** Azure, RAG, AI/ML knowledge, or specific frameworks, algorithms, or libraries by name (no need to know a particular search engine or technique)

---

## Instructions for the interviewer

Give the candidate the **Candidate Brief** (next section) printed or on screen. Let them read it for 3–4 minutes. Then walk through Parts 1–3 in order. Each part builds on the previous one. The candidate doesn't see the evaluator notes — those are for you.

**Pacing guide:**
- Reading + initial questions: 5 min
- Part 1 (system design): 15–20 min
- Part 2 (debugging): 15–20 min
- Part 3 (designing a similarity score): 10–15 min
- Wrap-up / candidate questions: 5 min

**Ground rules to tell the candidate:**
- There are no trick questions. We're interested in how you think, not whether you get a "right answer."
- If information is missing, say so. Make assumptions, state them explicitly, and move on.
- You can draw diagrams, write pseudocode, or just talk — whatever helps you explain your thinking.
- Ask us questions at any point. In a real job, you'd be working with a team, not in isolation.

---

## Candidate Brief

> **Scenario: SmartSearch — Internal Document Search for a Large Organization**
>
> Your company has 5,000 employees across 4 offices in Europe. They produce and consume thousands of internal documents: policies, technical reports, meeting notes, project plans, contracts, and training materials. These documents live in a shared file system organized by department.
>
> Today, employees find documents by browsing folder structures or using a basic keyword search tool. Common complaints:
>
> - "I know there's a policy about remote work expenses, but I can't find it."
> - "The search only works if I know the exact words in the document. If I search 'parental leave' but the document says 'family absence policy,' nothing comes up."
> - "I found a document but it's 120 pages. I just need the part about contractor rates."
> - "I don't know if the document I found is the latest version or if it's been superseded."
>
> **Management has approved a project to build a smarter internal search system.** The system should let employees type a question in natural language (e.g., "What is our policy on expensing home office equipment?") and get a direct, accurate answer with a reference to the source document.
>
> You've just joined the team that will build this. The team has a senior architect, two backend developers, one frontend developer, and you.

---

## Part 1 — System Design (15–20 min)

### Ask the candidate:

*"Before writing any code, the team needs to understand the problem and sketch out a high-level approach. Walk us through how you'd think about building this system. Don't worry about specific technologies — we're interested in how you break down the problem."*

### Follow-up prompts (use as needed to guide the conversation):

If the candidate jumps straight to implementation, pull them back:
- *"Before we talk about how to build it — what are the main sub-problems we need to solve?"*

If they give a high-level answer but don't go deeper:
- *"You mentioned the system needs to understand questions. Let's say a user types 'How many vacation days do I get?' — walk us through what happens between the user pressing Enter and seeing an answer."*

If they mention search but don't address the "120-page document" complaint:
- *"Good. Now, imagine the search finds the right document — it's a 200-page HR policy manual. How do we get the user a specific answer instead of just linking the whole document?"*

If they're doing well, push further:
- *"The company has 15,000 documents. Some are current, some are outdated. How would you handle the versioning problem — making sure the system doesn't answer based on a superseded policy?"*

Force a trade-off decision:
- *"The architect proposes two options: Option A processes all documents upfront and builds a searchable index (takes 2 weeks to set up, fast queries). Option B processes documents on-the-fly when a user searches (no setup, but each query takes 10–15 seconds). There's pressure to launch within a month. Which would you recommend and why?"*

### What good looks like (evaluator notes):

**Strong candidate:**
- Identifies sub-problems without being told: document ingestion/processing, search/matching, answer extraction, versioning/freshness, user interface
- Recognizes the gap between keyword search and meaning-based search (doesn't need to know the term "semantic search" — saying "the system needs to understand what the user means, not just match words" is enough)
- Thinks about the document processing step: "we need to break large documents into smaller pieces so we can find the relevant section" (this is chunking — they don't need to know the word)
- Considers the user experience: what does the answer look like? Is it a paragraph? A link? Both?
- On the trade-off question: makes a clear recommendation with reasoning. The "right" answer is Option A (pre-process) but what matters is the reasoning — mentioning user experience (10-15s is too slow), the one-month deadline still being achievable with Option A, or suggesting a hybrid (start with a subset of the most-used documents) all show good thinking
- Asks clarifying questions: "Are documents in different formats? Are they all in one language? Do different employees have access to different documents?"

**Adequate candidate:**
- Gets to sub-problems with prompting
- Describes a reasonable flow (user asks → system searches → shows answer)
- Makes a trade-off decision but with shallow reasoning

**Areas of concern:**
- Finds it difficult to break the problem down without substantial guidance
- Tends to reach for specific technologies ("we should use Elasticsearch") before understanding the problem
- Doesn't yet consider edge cases or ask clarifying questions
- Has trouble articulating a trade-off — picks an option but finds it hard to explain why

---

## Part 2 — Debugging and Investigation (15–20 min)

### Setup — read this to the candidate:

*"Good. Now imagine the system has been built and deployed. It's been running for 3 months. We're going to describe a situation, and we'd like you to help us figure out what's going wrong."*

*"We're getting this report from the finance department:"*

> **User report:** "I asked SmartSearch 'What is the approval limit for purchase orders?' and it gave me an answer: 'The approval limit for purchase orders is €5,000.' But when I checked the actual document, the current policy says €10,000. The €5,000 limit was from an older version of the policy that was replaced 6 months ago. This has happened multiple times with different finance policies."

### Ask the candidate:

*"Where would you start investigating this? What are the possible causes?"*

### Follow-up prompts:

If they identify the versioning issue quickly:
- *"Good. Now assume the old document was properly marked as superseded in the file system. The new document is also in the system. Why might the search still return the old answer?"*

If they need help:
- *"Think about the two phases: (1) when documents were processed and stored, and (2) when the user searches. Where in that pipeline could the old version still be active?"*

Push to a systematic approach:
- *"You have access to the system's logs, the document database, and the search index. Walk us through exactly what you'd check, in what order, and why."*

Add a new wrinkle:
- *"While investigating, you discover something else: the system actually has BOTH versions indexed — the old €5,000 policy and the new €10,000 policy. The search returned the old one. Why might the system prefer the older document over the newer one?"*

If they're strong, introduce ambiguity:
- *"A colleague says 'just delete the old version from the index, problem solved.' Another says 'we should keep old versions because some queries might be about historical policies.' Who's right? How would you design for both needs?"*

### What good looks like (evaluator notes):

**Strong candidate:**
- Immediately identifies multiple hypotheses before diving into one:
  - The old document is still in the index and wasn't removed when superseded
  - The new document was never processed / ingested
  - Both are in the index but the old one ranks higher
  - The document was updated but the index wasn't refreshed
- Proposes a systematic investigation order (not random guessing):
  1. First check: is the new document in the index? (data problem)
  2. Second check: if both are there, what determines which one wins? (ranking/scoring problem)
  3. Third check: is there a process that should have removed or de-prioritized the old one? (process problem)
- On the "both versions indexed" question: reasons about why the older document might rank higher — it's been in the system longer and may have been accessed/clicked more, or the old document's language might match the query better, or there's no date-based weighting in the ranking
- On the "delete vs keep" debate: recognizes this is a design decision, not a purely technical one. A good answer: "we could keep old versions but tag them and de-prioritize them in search by default, with an option for users to specifically search historical policies"
- Shows an engineering mindset: "before fixing this specific case, how many other documents have this problem? Is this systemic or a one-off?"

**Adequate candidate:**
- Identifies the main cause (old document still in the index)
- Proposes a reasonable investigation approach
- Makes a decision on delete vs keep with some reasoning

**Areas of concern:**
- Tends to settle on a single possibility and commit to it straight away
- Finds it difficult to describe an investigation process, moving to solutions before diagnosing
- On the design question, gives a one-sided answer with limited nuance
- Doesn't yet consider whether this might be a systemic issue

---

## Part 3 — Designing a Similarity Score (≤15 min)

A concrete, self-contained problem where the candidate proposes and reasons through an approach of their own. No hidden trick, no domain knowledge needed. We give them a simple representation so they actually build something rather than name a technique. The framing is deliberately open ("how would you approach this?") so we see how they start, structure their thinking, and refine.

### Setup — read this to the candidate:

*"For the last part, a focused technical problem. To match a question to documents, the system needs to score how well each document matches the question. Let's make it concrete: imagine each document has been reduced to the set of distinct words it contains, and the question is also just a set of words."*

### The problem:

*"Given those two sets of words — the question's and a document's — how would you approach building a score for how well they match? Walk us through how you'd start: your first, simplest idea, the aspects you'd want to consider, and how your approach would evolve as you think it through."*

### Pushing further (use as the candidate gives you room):

- **Length:** *"A document that's 100 pages long contains a huge number of words. Would your score unfairly favour it? Why?"*
- **Common words:** *"The word 'the' appears in almost every document. Should it count as much as the word 'parental'?"*
- **Synonyms:** *"The question says 'parental leave,' but the right document says 'family absence.' What does your score do — and what would you need to handle that?"*
- **Stretch:** *"How would you turn your score into a ranked list of the top documents for a query?"*

### What good looks like (evaluator notes)

**Strong candidate:**
- Starts with a clear, simple idea — e.g. counting the words the two sets share (the size of the overlap) — and states it precisely before refining
- Quickly sees it needs normalizing: a longer document shares more words just by being long, so divides by something — the total distinct words across both (the union), or the number of words in the question — to make scores comparable across documents
- Recognises not all words are equal: very common words ("the," "is") shouldn't count as much as rare, distinctive ones, and proposes down-weighting frequent words or removing them (this is the idea behind TF-IDF, reasoned from first principles — they don't need the name)
- Identifies the real limit of word-matching: different words for the same idea (synonyms) never match, so genuinely relevant documents get missed — and gestures at needing a representation of *meaning*, not just exact words
- Structures the discussion well — starts simple, names the aspects worth considering (length, word frequency, synonyms, ranking), and justifies each design choice ("I divide by the union so a document can't win just by being long")

**Adequate candidate:**
- Proposes counting shared words and, with prompting, sees the need to normalize for length and/or down-weight common words; spots the synonym problem once it's pointed out

**Areas of concern:**
- Proposes counting shared words and stops; doesn't see that document length or common words distort the score even when prompted; treats raw overlap as enough

---

## Scoring Guide

| Competency | Weight | 1 — Insufficient | 2 — Developing | 3 — Meets expectations | 4 — Exceeds expectations |
|-----------|--------|-------------------|-----------------|----------------------|------------------------|
| **Analytical thinking** | 30% | Finds it difficult to decompose problems. Needs substantial guidance at each step. | Identifies obvious sub-problems with prompting. Shallow analysis. | Breaks down problems independently. Considers multiple angles. Identifies at least 2-3 hypotheses before committing. | Structures analysis without prompting. Identifies non-obvious connections. Asks probing clarifying questions. Thinks about systemic implications. |
| **Problem-solving under ambiguity** | 25% | Finds it hard to proceed when information is incomplete. Tends to wait for "the right answer." | Makes assumptions but doesn't state them. Needs guidance to handle unknowns. | States assumptions explicitly. Proposes reasonable approaches when information is missing. Adapts when new information is introduced. | Comfortable with ambiguity. Identifies what's known vs unknown. Proposes ways to reduce uncertainty. Adjusts approach fluidly when the problem changes mid-discussion. |
| **Technical reasoning (algorithm design)** | 30% | Proposes counting shared words and stops; doesn't see that length or common words distort the score even when prompted. | Counts shared words and, with prompting, sees the need to normalize for length and/or down-weight common words. | Starts simple, then normalizes for length and reasons about weighting words by how common they are. | Starts simple and refines unprompted — normalizes, down-weights common words (TF-IDF idea from scratch), identifies the synonym limit, and structures the whole discussion clearly. |
| **Communication** | 15% | Disorganized explanations. Heavy jargon or vague hand-waving throughout. | Coherent but unstructured. Hard to follow at points. | Clear, structured communication across all three parts. Explains reasoning as they go. | Concise and precise. Uses analogies and structure effectively ("There are three things going on here..."). Makes the interviewer's job easy. |

**Overall scoring** (sum the four scores for a 4–16 raw total):
- **12–16:** Strong hire signal. Candidate demonstrates the analytical, structuring, and communication skills needed to grow into the role.
- **8–11:** Potential hire. May need more mentoring, but shows enough foundation. Consider in context of other candidates.
- **4–7:** Below bar. Significant gaps in analytical reasoning, structured thinking, or communication that would make onboarding difficult.

**Important notes for interviewers:**
- A graduate who asks 5 great clarifying questions and only gets halfway through Part 1 can score higher than one who rushes through everything with shallow answers.
- We're not looking for "correct" answers. We're looking for structured thinking, intellectual curiosity, and the ability to communicate clearly.
- If the candidate is visibly nervous, spend an extra minute on the ground rules and reassure them this is a discussion, not an exam.
- Take notes on specific things they said — "candidate identified the versioning sub-problem unprompted" is useful. "Good communication" is not.

---

*Prepared — May 2026*
