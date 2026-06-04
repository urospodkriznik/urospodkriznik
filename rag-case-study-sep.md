# Building a RAG system for academic self-evaluation reports

**Client:** A national university system responsible for quality oversight across 20+ faculties  
**Stack:** PHP · Vue 3 · Azure AI Search · Azure OpenAI · MySQL  
**Status:** Live in production, actively used for accreditation reviews

---

## The problem

Every year, each of roughly 450 academic programmes across the university's member faculties submits a self-evaluation report. The report is a structured questionnaire — 50+ open-ended questions covering internationalisation, curriculum quality, student support, research integration, and more — plus three reference tables, the most important being a forward-looking improvement plan that tracks weaknesses, goals, proposed actions, and responsible persons.

Responses vary significantly in length. Some programme coordinators write extensively; others keep answers brief. A typical report contains somewhere between 5,000 and 12,000 words of free-text responses, not counting question titles and structural text.

With five years of data in the system, this means approximately 1,600 reports in the archive, with 400–450 new ones added each year.

The team responsible for quality oversight — the people who read these reports, identify systemic problems, and prepare materials for accreditation reviews — had no way to search across them.

**What they were doing before:** One analyst exported reports to Word, uploaded batches of 30 into NotebookLM, and asked questions there. The problems were significant: question titles and structural boilerplate created noise in retrieval, the 30-report limit meant most of the archive was unreachable at once, and the tool had no understanding of the report's internal structure — it couldn't filter by faculty, programme type, year, or question.

But the deeper problem wasn't speed — it was that the knowledge was effectively inaccessible. Consider a concrete example: the analyst needs to find all programmes that mentioned internationalisation challenges across the past five years and understand what specific problems each one described. To do this manually, she would need to open each of roughly 400 reports, locate the relevant section, read it, note the context, and repeat for every report — then start over the next time a different topic comes up. In practice, this doesn't happen. Reports are read once during the approval process and then effectively lost. With AI search across the full archive, the same question takes minutes.

When a programme comes up for accreditation renewal, the team can now retrieve everything that programme has written about a specific topic across five years, see how they responded to previous recommendations, and ask the LLM to summarise patterns — all in a single session.

---

## Why this was harder than it looks

The data lived in a relational MySQL database — the same one used by the existing web application I had been maintaining and developing for the client. The reports weren't files or documents. They were answers to specific questions stored across multiple normalised tables: programmes, faculties, questionnaire pages, questions, text answers, table rows, and table cells.

Before any AI work, I had to answer: what is a "document" in this context, and what metadata needs to travel with each chunk?

A naive approach — exporting each full report as one text blob — would have destroyed the structure that makes the data useful. The analyst needs to filter by question, by section, by programme type, by year. That meant every chunk in the index needed to carry metadata: which faculty, which programme, which academic year, which questionnaire section, which specific question, and what type of programme the question was applicable to.

**The question visibility problem** was non-trivial. The questionnaire has conditional logic — some questions only apply to first-cycle programmes, some only to doctoral programmes, some only to dual-subject programmes. A question about doctoral thesis supervision has no business appearing in search results for a bachelor's degree programme. I encoded this visibility logic in the ingestion pipeline so that irrelevant question-programme combinations are excluded from the index entirely, not just filtered at query time.

**The noise problem** was equally important. Across 1,600 reports, hundreds of answers were empty or near-empty — formulaic non-answers submitted when a question didn't apply to a specific programme. Including these as searchable content would pollute retrieval with meaningless matches. I built a normalised exclude set of ~70 such phrases that the pipeline checks before deciding whether a chunk gets an embedding or is stored as an empty placeholder.

**Two types of static tables** required separate handling in both ingestion and retrieval. The improvement plan table has up to seven columns with different semantic meanings. For these, I support two search modes: by row (searching the full narrative content of a row) and by column (searching specific columns independently — useful when the analyst wants to find keywords in the "proposed actions" column specifically, rather than across the entire table).

---

## Architecture decisions

**Chunking strategy:** I did not chunk by token count. The fundamental unit is one answer to one question in one programme's report — that is the semantically meaningful unit for this domain. If an answer is 20 words, it becomes one small chunk. If an answer is 2,000 words, it becomes up to three chunks split on paragraph boundaries, falling back to sentence boundaries. This is intentionally different from generic RAG implementations that chunk everything into fixed 500-token windows regardless of content boundaries. In those implementations, a single chunk might contain the end of one programme's answer and the beginning of another programme's entirely different answer to a different question. For this use case that would be wrong — the answer boundary is the document boundary.

**Why Azure AI Search over a vector-only solution:** The queries analysts run are not purely semantic. "Show me all programmes that mentioned internationalisation" is a keyword search. "Find programmes describing problems with Erasmus student integration" is semantic. "Filter to first-cycle programmes at a specific faculty from the last two years" is structured metadata filtering. A pure vector database can't handle the first and third cases. Azure AI Search supports hybrid queries — BM25 keyword search combined with HNSW vector search — with metadata filtering on top. This was the right architecture for the actual query patterns.

**Index field design:** The index has 50+ fields. Most are not searchable — they're filterable, facetable, or retrievable metadata used by the frontend to build filter dropdowns, display results in context, and construct the evidence payload for LLM analysis. Only the main content field and the static table column fields are full-text searchable. The vector field is not retrievable (no reason to return 1,536 floats to the client) but supports approximate nearest neighbour queries.

**The analysis layer:** Search results are returned as a JSON array to the Vue frontend. When the analyst wants LLM analysis, the current result set is packaged as evidence and sent to Azure OpenAI with a system prompt that instructs it to answer only from the provided evidence, cite document IDs, and return structured HTML. The evidence payload is capped by character count before sending — different limits for different models (350k for GPT-4o-mini, 750k for GPT-5 variants). Token usage and estimated cost are shown per analysis call so the team can choose which model to use based on the complexity of the question.

**Integration into the existing application:** The entire system lives inside the existing PHP application — no separate service, no new infrastructure beyond the Azure resources themselves. The ingestion pipeline is a PHP CLI script that reads from the same MySQL database the app already uses. The API is a PHP class in the existing backend. The frontend components are Vue 3, integrated into the existing app. The team already knew how to use the application, the data was already there, and a separate service would have added operational overhead with no benefit to the user.

---

## The hard parts

**Getting the relational-to-flat mapping right** took most of the design time. The final SQL for the main question export is an 8-way join. Understanding which records should produce which documents required deep knowledge of the application's domain model — knowledge built up over years of maintaining the codebase.

**Schema migrations on a live index** required writing a backfill utility. When I added new fields to the index definition after data was already ingested, I needed to populate those fields on existing documents without re-ingesting everything from scratch. The backfill script pages through null-valued records using Azure Search's filter and merge API, with progress logging and a dry-run mode.

**Rate limiting and retry logic** mattered more than expected. Embedding 1,600 reports in batches means hundreds of API calls. The pipeline implements exponential backoff for 429 and 5xx responses so that a transient Azure outage doesn't corrupt a partially-ingested batch.

---

## Results

The system is live and used regularly. The primary use case is accreditation preparation: when a programme comes up for review, the analyst can in minutes retrieve everything that programme has written about a specific topic across five years of reports and ask the LLM to summarise patterns, recurring problems, or unaddressed commitments.

The more significant change is not time saved but capability unlocked. The manual approach wasn't just slow — it didn't scale to the volume of data. Knowledge embedded in 1,600 reports was effectively inaccessible. It now isn't.

---

## What's next

**Conversation memory for the analysis layer.** The LLM analysis is currently stateless — each call is independent. An analyst who wants to refine or follow up on a previous result has to re-run the search. Adding a lightweight conversation layer on top of the existing analysis history is the most requested improvement.

**Formal evaluation.** As the system gets used for higher-stakes decisions like accreditation reviews, adding a structured eval set becomes important — a collection of specific queries with known correct answers, measuring retrieval precision and recall. This would make it possible to measure the impact of any changes to chunking parameters, search configuration, or prompting, and to demonstrate retrieval quality to stakeholders beyond "the results feel useful."
