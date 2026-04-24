# Method 07 — Field Intelligence

## Role

Independent agent that runs **Field Intelligence gathering** — retrieving the questions the field itself considers hardest or most neglected — and writes a flat numbered list of questions.

This agent operates by **retrieval**, not generation.

## Shared scaffold

```
Topic: {topic}
Depth: {depth_budget}
Run directory: {run_dir}
Output path: {output_path}  (= {run_dir}/outputs/07-field-intelligence.md)

Format: flat numbered list of questions only.
Tag each question with the sub-technique: "N. [Tag] Question text?"
No answers, no explanations, no markdown sub-sections beyond the single top-level heading.
The last non-blank line MUST be exactly: <!-- COMPLETE -->
```

## Task

Surface questions already recorded publicly by serious practitioners and scholars. Attribute each question to the source where it was found (inline in the question text or via a short citation note inside the brackets: `[Experts:{name}]`, `[Papers:{first-author-year}]`).

No repetition across sub-techniques.

## Tools

Requires `WebSearch`; `WebFetch` also useful for named-page retrieval.

## Search constraints (mandatory)

FAILURE TO FOLLOW THESE WILL CAUSE AGENT FREEZE:

**WebSearch iteration limit**: Maximum 3 WebSearch calls per sub-section (Experts, Books, Conferences, Papers, Communities). If 3 calls don't yield 2 verified sources, proceed with what you have and mark the gap.


## Method

Follow the triangulation discipline in [../../references/source-triangulation.md](../../references/source-triangulation.md). **Every named source MUST be verified by at least one independent search result** before writing.

### Recognized experts
Identify 3–5 significant thinkers. Produce questions they publicly identify as the field's hardest open problems. Tag: `[Experts:{name}]`.

### Canonical texts
3–5 foundational books or papers. Produce questions they open rather than close. Tag: `[Books:{author-year}]`.

### Conferences and talks
3–5 venues. Produce questions from recurring debates or standing-room sessions. Tag: `[Conferences:{venue}]`.

### Scientific publications and preprints
5–10 high-citation works. Extract open questions from their introductions, conclusions, and "future work" sections. Tag: `[Papers:{author-year}]`.

### Communities and practitioners
3–5 forums, working groups, society chapters. Produce questions that recur; questions newcomers always ask that veterans struggle to answer. Tag: `[Communities:{forum-name}]`.

## Output shape

```markdown
# Method 07 — Field Intelligence

1. [Experts:{Dedre Gentner}] What would a truly productive analogy across X and Y look like, given that most candidate mappings turn out to be surface-similarity illusions?
2. [Papers:{Smith 2024}] How can the replication gap between finding A and finding B be closed without abandoning the original framework?
3. [Communities:{subreddit-name}] Why does the question "how do I get started with X" never get a good answer here despite being asked weekly?
…

<!-- COMPLETE -->
```

## Length targets

| depth_budget | Total |
|--|--|
| quick | 20–30 |
| standard | 30–45 |
| deep | 45–65 |

## Verification

1. Every named expert / author / venue / community appears in at least one independent search result (no recall without retrieval).
2. Every expert entry cites a **specific contribution** or **publicly-stated open problem**, not a general reputation. Quote or paraphrase with attribution.
3. Every cited paper's open question is traced to a specific paper section.
4. If a sub-slot genuinely has no source, write e.g. `[Experts] No recognized expert could be confirmed for this topic — gap open`.
5. Every line `N. [Tag] …?`. Single heading. No sub-sections.
6. Last non-blank line is exactly `<!-- COMPLETE -->`.

## Gotchas

G-01. This agent is the most prone to **fabricated citations** because retrieval is easier to skip than generation. Never name an author / title / venue without an independent web hit.

G-02. An expert's "general reputation" is not a question source. The expert must **explicitly** name an open question in a paper, talk, interview, or retirement lecture.

G-03. For humanities / craft topics without formal BOKs, substitute retrieval channels with their analogs — guild publications, regional tradition writings, cookbook editions, extension-service bulletins.

G-04. Recent-paper retrieval can false-negative within 90 days of publication. If the topic is actively moving, add a final entry like `[Experts] Field is actively moving as of {YYYY-MM}; re-run this agent in 90 days to catch newer open problems` as its own question.

G-05. Completion marker rules — see [../../references/checkpointing-protocol.md](../../references/checkpointing-protocol.md).

G-06. **SEARCH FREEZE PREVENTION**: If you find yourself repeating the same search with minor variations, or if the same 3-5 URLs keep appearing across searches, abort the search cycle and write `[WebSearch] Exhausted all productive search vectors — gaps noted`. This is not failure; it is productive termination of unproductive search.

G-07. If a named expert has no verifiable connection to the topic, **do not retry with more searches**. Mark the gap and move to another expert. Repeated failure on a single named source means the source is not relevant — replace it.
