# DocuBot Model Card

This model card is a short reflection on your DocuBot system. Fill it out after you have implemented retrieval and experimented with all three modes:

1. Naive LLM over full docs  
2. Retrieval only  
3. RAG (retrieval plus LLM)

Use clear, honest descriptions. It is fine if your system is imperfect.

---

## 1. System Overview

**What is DocuBot trying to do?**  
Describe the overall goal in 2 to 3 sentences.

> DocuBot is basically a mini "chat with your docs" tool. The goal is to let a developer ask a plain-English question about this project (auth, database, API routes, setup) and get an answer that's actually grounded in the markdown files in `docs/`, instead of the model just guessing based on whatever it saw in training. It's set up so you can compare three ways of doing that — no retrieval at all, retrieval with no LLM, and retrieval + LLM together — which is really the whole point of the assignment.

**What inputs does DocuBot take?**  
For example: user question, docs in folder, environment variables.

> The main input is the user's question, typed in through `main.py`'s CLI. Behind that, it also reads in whatever `.md`/`.txt` files exist in the `docs/` folder at startup (that's the whole corpus it has to work with), and it needs a `GEMINI_API_KEY` env var (loaded from `.env`) if you want the naive or RAG modes — retrieval-only doesn't need it at all.

**What outputs does DocuBot produce?**

> Depends on the mode. Retrieval-only just dumps the raw top-k paragraph snippets with their source filenames, no rewriting at all. Naive and RAG modes both return a block of generated text from Gemini — naive is a free-form answer with no source docs involved, RAG is (supposed to be) an answer that sticks to the retrieved snippets and cites which files it used, or refuses if it can't find enough evidence.

---

## 2. Retrieval Design

**How does your retrieval system work?**  
Describe your choices for indexing and scoring.

- How do you turn documents into an index?
- How do you score relevance for a query?
- How do you choose top snippets?

> Nothing fancy — this is the "tiny" version on purpose. First, every doc gets split into paragraph-sized chunks by splitting on blank lines (`\n\n`), so DocuBot isn't stuck returning whole files as "the answer." Then I build an inverted index by lowercasing and `.split()`-ing each paragraph into words and mapping `token -> [filenames]`. Scoring is just term-frequency: lowercase the query, `.split()` it into words, and for each query word count how many times it shows up in the paragraph's word list, then sum those counts. `retrieve()` scores every paragraph, throws out anything scoring 0, sorts by score descending, and returns the top-k (default 3). It's honestly closer to a keyword search than real "search," but it's transparent and easy to reason about.

**What tradeoffs did you make?**  
For example: speed vs precision, simplicity vs accuracy.

> Definitely picked simplicity over precision here. Plain `.split()` with no stemming, no synonym handling, no TF-IDF/BM25 weighting means common words get treated the same as rare, meaningful ones, and exact word-form mismatches (singular vs plural, a word buried inside `snake_case_identifier.py`) can tank a paragraph's score even when it's clearly the most relevant one — I actually ran into this directly (see Section 5). The upside is it's fast, has zero dependencies, and the scoring logic fits in like 5 lines, which makes it easy to debug and explain. If I wanted better precision I'd add basic normalization (stemming, ignoring stopwords) or a real vector/embedding-based similarity, but that's a lot more moving parts for a tinker project like this.

---

## 3. Use of the LLM (Gemini)

**When does DocuBot call the LLM and when does it not?**  
Briefly describe how each mode behaves.

- Naive LLM mode:
- Retrieval only mode:
- RAG mode:

> Naive mode calls Gemini every single time, no exceptions, and doesn't even look at the docs — it just forwards the raw question in a generic "you are a documentation assistant" prompt. Retrieval-only never touches the LLM at all; it's pure `retrieve()` output, so if `GEMINI_API_KEY` isn't set this mode still works fine. RAG mode calls the LLM too, but only after `retrieve()` runs and only if there's "sufficient evidence" (see the guardrail below) — if retrieval comes back empty or too weak, it short-circuits and returns the refusal message without ever calling Gemini, which saves an API call on top of being honest.

**What instructions do you give the LLM to keep it grounded?**  
Summarize the rules from your prompt. For example: only use snippets, say "I do not know" when needed, cite files.

> The RAG prompt (in `llm_client.answer_from_snippets`) lays out the retrieved snippets labeled by filename, then gives the model explicit rules: (1) answer using only what's in the snippets, (2) don't invent functions, endpoints, or config values that aren't shown, (3) if the snippets don't provide enough to answer confidently, reply with the exact refusal string "I do not know based on the docs I have," and (4) when it does answer, briefly say which files it pulled from. Naive mode has none of these guardrails — it's just a bare question with no context and no refusal instruction, which is basically the point of that mode (show what happens with zero grounding).

---

## 4. Experiments and Comparisons

Run the **same set of queries** in all three modes. Fill in the table with short notes.

You can reuse or adapt the queries from `dataset.py`.

| Query | Naive LLM: helpful or harmful? | Retrieval only: helpful or harmful? | RAG: helpful or harmful? | Notes |
|------|---------------------------------|--------------------------------------|---------------------------|-------|
| Where is the auth token generated? | Harmful (confidently wrong) | Mostly accurate but incomplete/hard to read | Not tested live (503 from Gemini), but see analysis below on why RAG's ceiling is limited by retrieval here | Naive answer invents a generic Node/Firebase/Auth0 explanation with zero connection to this codebase. Retrieval pulled AUTH.md's signing paragraph + validation checklist + an unrelated API_REFERENCE 401/403 list, but **missed** the actual "Token Generation" sentence (`generate_access_token` in `auth_utils.py`) due to exact-word matching (query "token" singular vs doc "Tokens" plural; "auth" never appears as a standalone word, only inside `auth_utils.py`). |
| How do I connect to the database? | Helpful-sounding, but generic — talks about connection strings/ORMs in the abstract, never mentions `DATABASE_URL` or that SQLite is the default | Helpful — pulls the exact `DATABASE_URL` paragraph from `DATABASE.md` including the SQLite/PostgreSQL examples | Would likely be the best of the three, since retrieval actually nails this one — a synthesized answer citing `DATABASE.md` should be clean and correct | Good example of a query where retrieval genuinely works well, since "database" and "connect" both appear near-verbatim in the source text |
| Which endpoint lists all users? | Harmful — guesses a generic REST convention (`GET /users`) instead of this project's actual `GET /api/users` | Helpful — retrieves the `GET /api/users` section from `API_REFERENCE.md`, admin-only note included | Should be the strongest mode here — clear one-line answer plus the "admin only" caveat, grounded in the real route | Shows naive mode failing quietly — a plausible-looking REST guess that just happens to be wrong for this specific API |
| How does a client refresh an access token? | Reasonable-sounding but invented — talks about generic refresh-token rotation patterns, no mention of `/api/refresh` | Helpful but fragmented — returns the right paragraph from `AUTH.md` ("Refresh the token when it expires by calling `/api/refresh`") but as one snippet among others with no framing | Should combine both nicely: name `/api/refresh`, cite `AUTH.md`/`API_REFERENCE.md`, and skip the naive mode's invented rotation details | Another case where retrieval alone finds the right fact but doesn't "answer" the question the way a synthesized sentence would |

*Honesty check: the "auth token" row is from an actual run of all three modes side by side. The other three rows I only spot-checked in retrieval-only mode and reasoned about naive/RAG behavior from the doc content and prompt rules, since Gemini was returning 503s during testing — so treat those as informed predictions, not confirmed transcripts.*

**What patterns did you notice?**

- **Naive LLM looks impressive but is untrustworthy:** For "Where is the auth token generated?", the naive mode produced a well-organized, confident, multi-section answer (Node.js/Express, Firebase Auth, Auth0/Okta, OAuth2 token endpoints) that reads like expert advice — but it never saw the docs, so every specific claim (`jwt.sign()`, `JwtTokenProvider`, `/oauth/token`) is invented and unrelated to this project. It even asks the user follow-up questions instead of admitting it has no project context. This is the clearest "sounds confident but weakly grounded" case: fluency is high, grounding is zero.
- **Retrieval only is accurate but hard to interpret:** the retrieved paragraphs are all real, verbatim, and traceable to `AUTH.md` / `API_REFERENCE.md` — so nothing is fabricated. But the output is three disconnected chunks (a validation checklist, a signing-key sentence, an error-code list) with no synthesis. A developer has to manually infer "so... the token is signed with `AUTH_SECRET_KEY`, I guess?" There's no direct sentence answering "where," even though the real answer (`generate_access_token` in `auth_utils.py`) exists in the corpus — it just wasn't retrieved. Accurate per-snippet, but not obviously *complete* or easy to scan.
- **Where RAG should balance clarity and evidence:** in principle, feeding those same 3 snippets to Gemini with a "use only these snippets, cite files" instruction should produce a synthesized, readable answer instead of raw fragments, while still refusing to invent the function name if it isn't in context. That's the theoretical advantage — clarity of naive mode, groundedness of retrieval.
- **Where RAG still fails:** RAG's output quality is capped by retrieval quality — garbage in, garbage out. Since retrieval never surfaced the paragraph containing `generate_access_token`/`auth_utils.py`, even a perfectly-behaved RAG call could only synthesize from the *signing* and *validation* snippets. Best case, it would correctly say "I do not know the exact function" or vaguely describe signing; it structurally cannot recover the one fact that answers the question directly, because that fact was never in its context window. This shows RAG doesn't fix retrieval bugs — it just prevents the LLM from hallucinating on top of them.

---

## 5. Failure Cases and Guardrails

**Describe at least two concrete failure cases you observed.**  
For each one, say:

- What was the question?  
- What did the system do?  
- What should have happened instead?

> **Failure case 1 — naive mode fabricates a plausible but wrong answer.**
> Question: "Where is the auth token generated?" The naive LLM mode (no retrieval, no doc access) answered with generic industry knowledge — Node.js/Express `jwt.sign()`, Firebase Auth, Auth0/Okta OAuth2 endpoints — none of which exist in this project. It should have either said it has no access to the actual codebase, or (once paired with retrieval) pointed to `generate_access_token` in `auth_utils.py` as described in `AUTH.md`. Naive mode has no way to know it's wrong, and nothing in its output signals low confidence.

> **Failure case 2 — retrieval misses the single best paragraph due to exact-word matching.**
> Same question. `AUTH.md` contains the sentence "Tokens are created by the `generate_access_token` function in the `auth_utils.py` module" — the direct answer. But `retrieve()` scores by exact whitespace-token overlap with no stemming/lemmatization: the query uses "token" (singular) and "auth" as standalone words, while the source sentence has "Tokens" (plural) and "auth" only as a substring of `auth_utils.py`. So this paragraph scored lower than two less-relevant ones (a validation checklist, an unrelated error-code list) and was pushed out of the top-3. What should have happened: either basic stemming/lowercasing normalization (token/tokens) or a scoring boost for query terms appearing as substrings of code identifiers, so the most on-topic paragraph isn't starved out by coincidental word-form mismatches.

**When should DocuBot say “I do not know based on the docs I have”?**  
Give at least two specific situations.

> Two big ones I ran into: (1) when the question has literally nothing to do with the docs — I tested "when is my birthday?" and retrieval correctly came back empty since no paragraph shares any real words with that query, so it refuses. And (2) when the query does share a word or two with some paragraph, but only by coincidence — like a common word matching without the paragraph actually being about the topic. That second case is the sneakier one, because "at least one word matched" isn't the same as "this is actually relevant," which is exactly why I didn't want the refusal check to just be `if not snippets`.

**What guardrails did you implement?**  
Examples: refusal rules, thresholds, limits on snippets, safe defaults.

> Main one is `has_sufficient_evidence()` in `docubot.py`, which both `answer_retrieval_only` and `answer_rag` call before doing anything else. Instead of just checking "did retrieval return anything," it checks whether the top-ranked snippet actually shares at least half of the query's distinct words (minimum 1) — so a single stray word match isn't enough to count as "evidence." If that check fails, both modes return the same `NO_EVIDENCE_MESSAGE` constant instead of guessing. On top of that, `retrieve()` already filters out anything with a score of 0 before it even gets ranked, and `top_k` caps how many snippets get passed along so the LLM (in RAG mode) isn't drowning in low-relevance context. The RAG prompt itself is a guardrail too — explicitly telling Gemini to refuse rather than invent details it can't back up from the snippets.

---

## 6. Limitations and Future Improvements

**Current limitations**  
List at least three limitations of your DocuBot system.

1. Retrieval is exact-word-only with no stemming, so plural/singular mismatches or words buried inside code identifiers (like `auth` inside `auth_utils.py`) can bury the single best paragraph under less-relevant ones, like I saw with the auth token example.
2. Scoring is just raw term frequency with no weighting — a paragraph that happens to repeat a common word a bunch of times can outscore a paragraph that's actually about the topic but only mentions it once. There's no notion of "rare words matter more."
3. Paragraph chunking is a dumb blank-line split, so if a doc has weird formatting (no blank lines between sections, or a huge single paragraph), chunks can end up way too big or too small, which messes with both scoring and how readable the retrieved snippet is.
4. Naive mode has zero grounding by design, so it will confidently answer questions about this codebase using generic internet knowledge, and there's nothing telling the user to distrust it more than the other modes.

**Future improvements**  
List two or three changes that would most improve reliability or usefulness.

1. Add basic normalization to scoring/indexing — lowercasing is already there, but stemming (token/tokens) or stripping punctuation off word boundaries would fix the exact-match misses without needing a full embedding model.
2. Swap raw term-frequency counts for something like TF-IDF, so common words across the whole corpus don't get the same weight as distinctive ones — that alone would probably fix a good chunk of the "wrong paragraph ranked #1" issues.
3. Have RAG mode show the user which snippets it actually pulled from (not just trust the model to cite them correctly in prose), so it's easier to tell when the model is drawing conclusions the retrieved evidence doesn't fully support.

---

## 7. Responsible Use

**Where could this system cause real world harm if used carelessly?**  
Think about wrong answers, missing information, or over trusting the LLM.

> The naive mode is the scariest one honestly — it answers every question fluently and confidently regardless of whether it actually knows anything about the project, so a developer in a hurry could easily copy a made-up function name or endpoint straight into production code. Even RAG mode isn't bulletproof: it's only as grounded as what retrieval hands it, so if retrieval quietly misses the right paragraph (like it did for the auth token question), RAG can still produce an answer that sounds well-cited but is built on incomplete evidence. And on the flip side, if docs are out of date or wrong, all three modes will happily repeat that wrong information back with full confidence — none of them fact-check the docs themselves.

**What instructions would you give real developers who want to use DocuBot safely?**  
Write 2 to 4 short bullet points.

- Never treat naive mode's answers as project-specific truth — it's basically a generic chatbot with no idea what this codebase actually contains, so only trust it for general concepts, not exact function/route names.
- Even with RAG, double check the cited files yourself before making a decision based on the answer, especially for anything security- or auth-related — "cited a real file" isn't the same as "correctly captured everything that file says."
- Keep the source docs accurate and up to date, since every mode is only as good as what's sitting in `docs/` — garbage or stale docs in, garbage or stale answers out.
- If DocuBot says "I do not know based on the docs I have," treat that as a real signal to go read the source code or docs yourself rather than re-asking the question a different way hoping for a better guess.

---
