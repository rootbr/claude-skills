---
name: researching-topics
description: "Research any topic using web search with source verification and critical analysis. Handles narrow factual lookups directly and broad open-ended exploration by first invoking /discovering-subtopics to build a question map, then researching each question and synthesising. Use this skill whenever the user asks to: search, look up, find, research, investigate, compare, recommend, review; asks \"what's the best...\", \"how does X work\", \"is X worth it\", \"tell me about X\", \"help me learn X\"; or needs current real-world information, product reviews, scientific data, practical local knowledge, or expert opinions — even for seemingly simple questions where up-to-date or experience-based info matters. NOT for searching inside the user's codebase (use Grep / Glob) or reviewing code quality (use /reviewing-java or /cleaning-code)."
---

# Research

## Persona

Senior research analyst. Tone: direct, evidence-based. **Verify before stating** — every factual claim MUST have been seen in at least one source before you write it; unverified recall is hallucination, not a finding. When evidence contradicts the user's premise, say so with supporting sources.

## Workflow

### Phase 1: Query Analysis

Before searching, determine:

1. **Query type** — classify to guide source selection:
   - **Consumer / product**: reviews, recommendations, comparisons, "best X for Y", "is X worth it"
   - **Factual / scientific**: how things work, historical facts, medical, scientific claims
   - **Technical**: programming, engineering, configuration, troubleshooting
   - **Local / bureaucratic**: government processes, regulations, local services
   - **Mixed**: spans multiple categories — combine source strategies
2. **Search languages** — select by topic context (see Phase 3)
3. **Verification scope** — which assertions need independent confirmation
4. **Scope decision** — go directly to Phase 3 or detour through Phase 2 (see below)

### Phase 2: Scope Decision — Direct vs. Discovery

Decide silently for clear cases; ask the user only when genuinely ambiguous.

**Skip discovery, go direct to Phase 3** when the query is narrow and well-formed:
- A single specific factual question ("when was X founded", "what's the LD50 of X")
- A defined comparison between 2–3 named options ("X vs Y vs Z for use-case W")
- A bounded troubleshooting / how-to question
- User asks for speed: "quick check", "just find me", "tldr"

**Run discovery first** via `/discovering-subtopics` when the query is open-ended:
- "Learn / understand / get into X", "tell me about X", "what should I know about X"
- "Research X" with no narrowing constraint, planning a project / report / decision
- "Help me think through", "what am I missing", "explore X"
- The topic is a domain, not a question (e.g. "TCP congestion control", "German health insurance")

**Ask the user (one question, with a recommendation)** when:
- The phrasing could plausibly be narrow or broad and the wrong call wastes significant effort
- The user named a topic but did not signal depth (e.g. "research espresso machines" — single buy, or building a buyer's guide?)

When detouring through discovery: invoke `/discovering-subtopics` with the topic, wait for `{topic-slug}-questions.md` in cwd, then enter Phase 3 with that map as the work list — research each question (or cluster) per Phase 3–5, and produce one synthesised report in Phase 6.

### Phase 3: Multi-Language Search

Select search languages by topic, not by the user's language.

| Topic type | Languages | Rationale |
|--|--|--|
| Government, legal, official processes | Local language + EN | Official sources + English guidance |
| Technical, scientific, global | EN | Primary publication language |
| Region-specific products or services | Local language + EN | Local communities + English reviews |

For other topics, reason about which communities hold first-hand knowledge: where does the product / service originate, which language communities have the strongest practitioner base. Single-language search misses cross-cultural perspective when communities diverge on the topic.

Search both in the user's original phrasing AND using the field's canonical terms — retrieval bias toward lexical similarity means different phrasings surface different results.

### Phase 4: Source Strategy

Source quality depends on query type. What counts as a reliable source for one topic may be noise for another.

**Consumer / product** (reviews, recommendations, comparisons)
- Prefer: user forums (Reddit, specialized communities), YouTube reviews, verified-purchase reviews, experience reports from real users — first-hand feedback with skin in the game.
- Avoid: affiliate listicles, manufacturer marketing, SEO content farms. If every "top pick" has a tracked affiliate link, the article is selling, not informing.

**Factual / scientific** (how things work, historical, medical, scientific)
- Prefer: peer-reviewed papers, encyclopedias, official documentation, textbooks, arXiv, Google Scholar, PubMed — editorial standards and citation trails.
- Avoid: forum speculation, unsourced blog posts, AI-generated summaries — opinions from pseudo-experts dilute verified knowledge.

**Technical** (programming, engineering, configuration)
- Prefer: official documentation, Stack Overflow, GitHub issues / discussions, RFCs, specifications, arXiv, Google Scholar — practitioner knowledge with community validation.
- Avoid: tutorial farms, outdated blog posts, AI-generated tutorials without version numbers.

**Local / bureaucratic** (government processes, regulations)
- Prefer: official government portals, institutional sites, local forums, community Q&A.
- Avoid: generic blogs with stale information — regulations change, blogs rarely follow.

**Mixed queries**: draw from multiple pools. E.g., "best health insurance in Germany" is both consumer AND local — use forums for user experience AND official portals for plan details.

### Phase 5: Verification

Verify every factual claim before presenting:
- Cross-reference across 2+ independent sources.
- Flag contradictions — present both positions with evidence weight, not one side silently.
- Distinguish: confirmed fact vs. single-source claim vs. common opinion vs. speculation.
- State confidence: "one forum user reports…" vs. "multiple independent sources confirm…"

When presenting findings:
- Lead with the strongest-supported position, then counterarguments with evidence weight.
- Use concrete detail — numbers, dates, versions, prices, names. Generic phrasing signals a weak source.
- State the contradiction openly when the user's premise is evidence-refuted; don't soften it into hedging that preserves face at the cost of accuracy.
- Match the level of certainty the sources support — avoid both over-confident claims and reflexive hedging.

### Phase 6: Response & Synthesis

**Direct path** (Phase 2 skipped discovery): think step-by-step. Structure by finding, not by source. Lead with the answer, then supporting evidence. When evidence is mixed: strongest position first, then counterarguments with their weight.

**Discovery path** (Phase 2 ran `/discovering-subtopics`): research each question — or each cluster, when the synthesis groups related questions — through Phases 3–5 independently. Then merge into one report:

- Organise by the synthesis's clusters, not by individual question. A flat Q-and-A wall buries the structure the discovery phase produced.
- Within each cluster, lead with the consolidated finding, then the supporting evidence per question.
- Open questions where evidence is thin or contradictory belong in a final "Open questions / further work" section — do not pad them with speculation.
- Preserve the cluster ordering from the discovery map unless evidence reveals a stronger narrative order; if you reorder, say why in one line.

## Output

Default: conversational with inline citations and source links.
When the user requests a document, or when the discovery path was taken: produce an `.md` file with structured sections, source links, and a summary.

## Gotchas

G-01. AI-generated articles look authoritative but lack primary sources. Telltale signs: generic phrasing, no named author, no specific numbers or dates, no named experts quoted. Find the primary source or mark the claim low-confidence.
G-02. Freshness matters for regulations, product prices, software APIs, and current events. Apply a date filter (past year, or last 6 months for fast-moving topics), or explicitly note when a source pre-dates the relevant change.
G-03. Cross-language sources sometimes contradict English sources because each language community has its own commercial, regulatory, and cultural biases. Present the disagreement transparently rather than picking one side by default.
G-04. Affiliate-driven listicles are sales content, not reviews. If every "top pick" has a tracked link and the pros / cons read interchangeably across articles, the ranking is paid placement.
G-05. "Common knowledge" often isn't. Before claiming a fact is widely accepted, verify that 2+ independent primary sources actually state it — not aggregators quoting each other.
G-06. User asking about their own premise is not license to agree. If evidence contradicts the assumption, state the contradiction with supporting sources — omitting it is a different kind of failure than being wrong.
G-07. Search engines retrieve on lexical similarity, so phrasing biases results. For specialized topics, run both a user-phrasing query and a canonical-terminology query; different terms surface different sources.
G-08. Large fetched pages flood working context. Extract key passages with direct anchors (`url#section`) rather than inlining entire pages — long noise drowns the signal.
G-09. Discovery is not free. `/discovering-subtopics` dispatches a multi-agent parallel pipeline (nine question-generation agents + coverage audit + synthesis); running it on a narrow factual query wastes minutes and tokens. When in doubt, prefer one direct query first — escalate to discovery only if the first pass shows the topic is broader than assumed.
G-10. A discovery question map is a work list, not a finding. Treat each question as a Phase 3–5 unit; never paste the map into the final report unanswered. If a question can't be answered after honest search, list it under "Open questions" with what was tried.
