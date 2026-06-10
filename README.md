# The Unofficial Guide — Project 1

> **How to use this template:**
> Complete each section *after* you've built and tested the corresponding part of your system.
> Do not write placeholder text — if a section isn't done yet, leave it blank and come back.
> Every section below is required for submission. One-liners will not receive full credit.

---

## Domain

<!-- What topic or category of knowledge does your system cover?
     Why is this knowledge valuable, and why is it hard to find through official channels?
     Example: "Student reviews of CS professors at [university] — useful because official
     course descriptions don't reflect teaching style, exam difficulty, or workload." -->
The Domain I chose was Advice NJIT upperclassmen and graduates wish they had received as freshmen. I decided to go with this because as a transfer student I believed most of the resources available at NJIT weren't adequately shared. Such as a free certification in Bloomberg Market Concepts as an Engineer and being able to do financial presentation as an Engineer. 
Most students just focus on their discipline and advisors do as well. The oppurtunities that exist in specific departments aren't as widely share to the NJIT student body.

---

## Document Sources

<!-- List every source you collected documents from.
     Be specific: include URLs, subreddit names, forum thread titles, or file names.
     Aim for variety — sources that together cover different subtopics or perspectives. -->

| # | Source | Description | URL or location |
|---|--------|-------------|-----------------|
| 1 | Academics | REGISTRATION, PROFESSORS, AND EXAM TIPS | https://www.ratemyprofessors.com/professor/130268 |
| 2 | Campus_life | DINING, STUDY SPACES, CLUBS, AND NEWARK DINING | https://www.roomsurf.com/dorm-reviews/njit |
| 3 | Careers | NJIT CAREER AND INTERNSHIPS GUIDE | https://www.njit.edu/careerservices/cooperative-education-and-internships |
| 4 | Commuter_Resource | COMMUTE PLANNING | https://www.njit.edu/commuters/commuter-tips-and-recommendations |
| 5 | Engineering_guide | ENGINEERING DEPARTMEN OVERVIEW | ai201-project1-guide-starter/documents/Engineering_guide.txt |
| 6 | Housing | NJIT HOUSING GUIDE | ai201-project1-guide-starter/documents/Housing.txt |
| 7 | LinkedIn_networking | NJIT INTERNSHIP GUIDE | ai201-project1-guide-starter/documents/LinkedIn_networking.txt |
| 8 | Reddit_miscallaneous | INFO SCRAPED FROM REDDIT | ai201-project1-guide-starter/documents/Reddit_miscallaneous.txt |
| 9 | Skill_Building | NJIT AS A SKILL-BUILDING RESOURCE | ai201-project1-guide-starter/documents/Skill_Building.txt |
| 10 | Student_Service | NJIT STUDENT SERVICES GUIDE | ai201-project1-guide-starter/documents/Student_Service.txt |

---

## Chunking Strategy

<!-- Describe your chunking approach with enough specificity that someone else could reproduce it.
     Include:
     - Chunk size (characters or tokens) and why that size fits your documents
     - Overlap size and why (or why not) you used overlap
     - Any preprocessing you did before chunking (e.g., stripping HTML, removing headers)
     - What your final chunk count was across all documents -->
     Chunk size = 500 characters: long enough to carry the semantic meaning of a single line, short enough to return targeted results
     Overlap = 75 characters: @50 there were parts thats that were cutoff that related to both chunks but needed more context. Cheap solution was to increase the overlap
     No preprocessing done, manually extracted information from sites and pasted them into .txt files.
     

**Chunk size:**
Chunk size = 500

**Overlap:**
Overlap = 75

**Why these choices fit your documents:**
Allowed model to produce lowest vector distance chunks

**Final chunk count:**
161 chunks

---

## Embedding Model

<!-- Name the embedding model you used and explain your choice.
     Then answer: if you were deploying this system for real users and cost wasn't a constraint,
     what tradeoffs would you weigh in choosing a different model?
     Consider: context length limits, multilingual support, accuracy on domain-specific text,
     latency, and local vs. API-hosted. -->

**Model used:**
all-MiniLM-L6-v2 (sentence-transformers). Chosen because it runs locally with no API key, and no rate limits, produces 384-dim unit-normalized embeddings, and embeds all 161 chunks plus each query in milliseconds on CPU — fine for a corpus this size.

**Production tradeoff reflection:**
If cost weren't a constraint, I'd weigh a few tradeoffs against a larger model like all-mpnet-base-v2 or an API-hosted model (e.g., OpenAI text-embedding-3-large):

- **Accuracy on domain-specific text:** During retrieval testing, MiniLM's cosine distances for genuinely relevant chunks often sat around 0.4-0.6 rather than near 0, and one test query initially missed its expected chunk entirely because the query's wording didn't closely match the source document's phrasing. A higher-capacity model (more dimensions, more training data) would likely generalize better across paraphrases of the same idea — directly relevant for a Q&A system where users won't phrase questions the way the source documents are written.
- **Context length:** MiniLM truncates at 256 tokens, which comfortably covers our ~500-character chunks, but a model with an 8K+ token context (like text-embedding-3-large) would let us use larger chunks without truncation risk — useful since some of our chunks currently split a single paragraph (e.g., a professor review) mid-sentence.
- **Multilingual support:** Not needed for this NJIT-English corpus, but if the guide expanded to cover resources for international students in their native languages, a multilingual model (e.g., multilingual-e5-large) would become necessary.
- **Latency / local vs. API-hosted:** Local CPU inference means zero network latency and no per-token cost, which matters when re-indexing the whole corpus during development. An API-hosted embedding model adds network round-trip latency and ongoing cost on every query and re-index — worth it at scale where embedding quality directly affects user-facing answer quality, but unnecessary overhead for a project this size.

---

## Grounded Generation

<!-- Explain how your system enforces grounding — how does it prevent the LLM from answering
     beyond the retrieved documents?
     Describe both your system prompt (what instruction you gave the model) and any structural
     choices (e.g., how you formatted the context, whether you filtered low-relevance chunks).
     Do not just say "I told it to use the documents" — show the actual instruction or explain
     the mechanism. -->

**System prompt grounding instruction:**
The full system prompt sent to Groq's llama-3.3-70b-versatile (in `generation.py`) is:

```
You are a helpful assistant answering questions about NJIT using ONLY the context
provided below. The context comes from student-written guides and reviews, not
official NJIT documentation.

Rules:
- Answer using only information stated in the context. Do not add facts from your
  own knowledge, even if you believe them to be true.
- If the context does not contain enough information to answer the question,
  respond with exactly: "I don't have enough information on that." and nothing else.
- Do not mention "the context" or "the documents" in your answer — write as if you
  are giving advice directly.
```

The retrieved chunks are passed in the user message as labeled blocks (`[Document N: <source filename>]\n<chunk text>`) so the model can see which document each fact comes from, but it's instructed not to reference "the context" itself in its answer. The exact-match refusal string lets the code (not a vague "did it refuse?" judgment) detect when the model couldn't answer — this is checked programmatically to decide whether to show sources at all (see below). I verified this works by asking an out-of-scope question ("What is the capital of France?"), which returned the exact refusal string instead of a generated answer.

**How source attribution is surfaced in the response:**
Source attribution is computed in code, not generated by the model. `generate_answer()` in `generation.py` builds `sources = sorted({chunk["metadata"]["source"] for chunk in chunks})` from the metadata of the chunks that were actually retrieved and passed as context — the LLM never writes citations itself. In the Gradio UI (`app.py`), this list is displayed in a separate "Retrieved from" textbox, bullet-prefixed (`• Campus_life.txt`). If the model's answer exactly matches the refusal string, `sources` is forced to an empty list, so the UI shows "(none — no grounded answer found)" instead of attributing sources to an answer that wasn't actually given.

---

## Evaluation Report

<!-- Run your 5 test questions from planning.md through your system and record the results.
     Be honest — a partially accurate or inaccurate result that you explain well is more
     valuable than a suspiciously perfect result. -->

| # | Question | Expected answer | System response (summarized) | Retrieval quality | Response accuracy |
|---|----------|-----------------|------------------------------|-------------------|-------------------|
| 1 | What are some courses or certifications I can take? | The Bloomberg terminal in CAPS allows you to take a self-paced certification course in Bloomberg Market Concepts | Recommends the BMC (Bloomberg Market Concepts) certification, plus mentions accelerated BS/MS programs | Relevant | Partially accurate |
| 2 | If I were to be looking for a career shift which department should I contact? | CDS from NJIT helps you connect with recruiters and companies that otherwise would be harder if you were to be cold applying or cold outreaching | Suggests contacting your department's graduate coordinator, and separately mentions CDS / the CareerShift tool | Partially relevant | Partially accurate |
| 3 | How can I apply what I learn in class to actual projects? | Join clubs in order to apply theoretical concepts from classes in a real-world scenario | Recommends building physical class projects in the Makerspace and pursuing independent study/research courses | Relevant | Partially accurate |
| 4 | What are some alternative path to experience NJIT offers? | Highlnder Launchpad connects students to student-founded startups that help them gain real-world experience. | Highlights Highlander Launchpad (student-founded startups), plus research positions, co-op, and The Vector newspaper | Relevant | Accurate |
| 5 | What is there to do in Newark? | NYC is is 35 mintues from newark and Ironbound is 15-20 minutes by foot which is known for the best food. | Lists Newark Museum of Art, Prudential Center, Branch Brook Park, and the Ironbound neighborhood | Relevant | Partially accurate |

**Retrieval quality:** Relevant / Partially relevant / Off-target  
**Response accuracy:** Accurate / Partially accurate / Inaccurate

---

## Failure Case Analysis

<!-- Identify at least one question where retrieval or generation did not work as expected.
     Write a specific explanation of *why* it failed, tied to a part of the pipeline.

     "The answer was wrong" is not an explanation.

     "The relevant information was split across a chunk boundary, so retrieval returned
     only half the context — the model didn't have enough to answer correctly" is an explanation.

     "The embedding model treated the professor's nickname as out-of-vocabulary and returned
     results from an unrelated review" is an explanation. -->

**Question that failed:**
Q2 — "If I were to be looking for a career shift which department should I contact?"

**What the system returned:**
"If you're looking for a career shift, you should talk to your department's graduate coordinator, but specifically for career-related inquiries, you can also utilize the resources provided by the Career Development Services (CDS). Additionally, CareerShift, a job search and networking tool, is available through the CDS Find Opportunities page." (Sources: Academics.txt, Careers.txt, Reddit_miscallaneous.txt)

The expected answer is all about CDS helping you connect with recruiters and companies for cold outreach. My system *does* eventually get there, but it opens with advice about talking to your department's graduate coordinator, which isn't really what the question is asking about.

**Root cause (tied to a specific pipeline stage):**
This comes down to the word "department" being ambiguous, and retrieval picking the wrong meaning. In the question, "department" means an NJIT office like CDS that helps with career stuff. But the top-ranked chunk (Reddit_miscallaneous.txt chunk_15, distance=0.4658) is actually about a totally different topic — switching from a regular BS into the accelerated BS/MS program, where it says "Talk to your department's graduate coordinator in sophomore year, not junior year." That chunk only mentions CDS in passing, in one sentence near the end ("Career Shift is genuinely different from what else CDS offers"). But that one sentence was apparently enough overlap with my query to push this chunk above Careers.txt chunk_0 (distance=0.6210), which is actually about CDS as a career-services office. Basically, the embedding model can't tell "your academic department's grad coordinator" apart from "the NJIT department that helps with careers" — they look similar enough in vector space that a chunk about the wrong one can still win.

**What you would change to fix it:**
Two things could help here. First, I could rephrase the question itself — the same trick that fixed test question 3. Something like "If I want to switch industries after graduating, which NJIT office helps me network with recruiters and companies?" uses words closer to how Careers.txt actually talks about CDS, instead of "career shift"/"department," which also happens to match the unrelated BS/MS content.

Second, this is partly a chunking problem. chunk_15 crams together several sentences about the BS/MS program with one unrelated sentence about CareerShift, just because that's how the 500-character window landed on the original Reddit text. If chunks were smaller or split more by topic/sentence, that CareerShift sentence would end up in its own chunk, where it would actually compete fairly against Careers.txt's CDS content instead of getting dragged along by an unrelated paragraph it happens to share a keyword with.

---

## Spec Reflection

<!-- Reflect on how planning.md shaped your implementation.
     Answer both questions with at least 2–3 sentences each. -->

**One way the spec helped you during implementation:**
The five test questions with expected answers gave me a real way to check retrieval instead of just trusting it. Running test question 1 with top-k=3 didn't return the chunk with the actual answer, so I raised top-k to 5 before moving on to generation.

**One way your implementation diverged from the spec, and why:**
The plan set chunk overlap to 50, but after reading sample chunks I saw this sometimes cut a word or short phrase right at the boundary, making chunks harder to read on their own. I raised the overlap to 75, which fixed most of these cases.

---

## AI Usage

<!-- Describe at least 2 specific instances where you used an AI tool during this project.
     For each: what did you give the AI as input, what did it produce, and what did you
     change, override, or direct differently?

     "I used Claude to help me code" is not sufficient.
     "I gave Claude my Chunking Strategy section from planning.md and asked it to implement
     chunk_text(). It returned a function using a fixed character split. I overrode the
     chunk size from 500 to 200 because my documents are short reviews, not long guides." -->

**Instance 1**

- *What I gave the AI:* The Documents and Chunking Strategy sections from planning.md, and asked for a script to load my .txt files, clean them, and split them into chunks.
- *What it produced:* `ingest.py`, which strips out empty lines and Reddit-style usernames, then splits each document into 500-character chunks with 50-character overlap, snapping the end of each chunk back to the nearest space.
- *What I changed or overrode:* After printing sample chunks, I saw the start of a chunk could still land mid-word (e.g. "line class search" instead of "Pipeline class search"). I had it fix `chunk_text()` to push the start forward to the next space too, and later raised the overlap from 50 to 75 after checking how often words still got split across chunks.

**Instance 2**

- *What I gave the AI:* My pipeline diagram, Retrieval Approach section, and the chunks from Milestone 3, and asked for code to embed the chunks, store them with source metadata, and return the top-k matches for a query.
- *What it produced:* `retrieval.py`, which loads `all-MiniLM-L6-v2`, stores chunks in ChromaDB with their source filename and position, and returns the top-k chunks with distance scores. By default, ChromaDB scored distance using squared L2.
- *What I changed or overrode:* When I ran my 5 test questions, the distance scores didn't line up with the reference numbers I was given. I asked the AI to explain the mismatch — it found ChromaDB was using squared L2 instead of cosine distance by default, and fixed it by setting `hnsw:space` to `"cosine"` and rebuilding the index. I also raised top-k from 3 to 5 after test question 1's expected chunk didn't show up in the top 3.
