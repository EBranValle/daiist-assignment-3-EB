# Assignment 3 Report

- **Name**:
- **Student ID**:
- **Group**: [BBADBA 5A | BBADBA 5B | PPLEDBA 5A]

## Use case

*Who asks, what they do with the answer, and what a wrong answer costs compared with "I don't know".*

## Corpus

Four EU legal acts on investment services and retail investment products, in English, from EUR-Lex. All are under the EUR-Lex reuse policy (Commission Decision 2011/833/EU: reuse authorised, with acknowledgement of the source), and none contains personal data. The PDFs are committed in `corpus/`, because EUR-Lex blocks scripted downloads with a bot challenge.

| Document (source name) | URL | Date | Licence | Pages | Words |
|---|---|---|---|---|---|
| Directive 2014/65/EU, MiFID II (`MiFID II`) | https://eur-lex.europa.eu/legal-content/EN/TXT/PDF/?uri=CELEX:32014L0065 | 15 May 2014 | EUR-Lex reuse (Decision 2011/833/EU) | 148 | 75,950 |
| Regulation (EU) No 600/2014, MiFIR (`MiFIR`) | https://eur-lex.europa.eu/legal-content/EN/TXT/PDF/?uri=CELEX:32014R0600 | 15 May 2014 | EUR-Lex reuse (Decision 2011/833/EU) | 65 | 36,478 |
| Delegated Regulation (EU) 2017/565 (`MiFID II Delegated Regulation`) | https://eur-lex.europa.eu/legal-content/EN/TXT/PDF/?uri=CELEX:32017R0565 | 25 April 2016 | EUR-Lex reuse (Decision 2011/833/EU) | 83 | 47,080 |
| Regulation (EU) No 1286/2014, PRIIPs (`PRIIPs`) | https://eur-lex.europa.eu/legal-content/EN/TXT/PDF/?uri=CELEX:32014R1286 | 26 November 2014 | EUR-Lex reuse (Decision 2011/833/EU) | 23 | 13,891 |
| **Total** | | | | **319** | **173,399** |

The word counts are after cleaning (headers, footers and page numbers removed). Each act is the original text as published in the Official Journal, not a consolidated version, so later amendments (for example the 2021 MiFID II "quick fix") are not in the corpus.

Held back for the add-document demo (see *Adding a document*): Delegated Regulation (EU) 2017/653, the PRIIPs regulatory technical standards (`PRIIPs RTS`), https://eur-lex.europa.eu/legal-content/EN/TXT/PDF/?uri=CELEX:32017R0653, 8 March 2017, same licence.

## Ingestion

**Extract and clean** (`index.py`).
- pypdf extracts the text page by page.
- Cleaning normalises Unicode (NFKC), removes soft hyphens, re-joins words hyphenated across lines and collapses whitespace.
- It drops page numbers, running headers and footers (lines that repeat at the top or bottom of 30%+ of pages), and the Official Journal header line wherever pypdf places it.

**Structure.**
- A new section starts at every line that is exactly `Article N` or `ANNEX …`.
- When a document gives its articles titles, the title on the next line is added to the location, e.g. `Article 25 (Assessment of suitability and appropriateness and reporting to clients)`. PRIIPs has no article titles; this is detected per document.
- Text before Article 1 is labelled `Recitals`.

**Chunking: two designs** (see *Retrieval*).
- `sections`: one chunk per article. Articles longer than 350 words are split on sentence boundaries. The embedded text is prefixed with `source - heading`, so each chunk carries the article it belongs to.
- `words150`: fixed windows of 150 words with a 30-word overlap, ignoring structure, with no prefix.

**Chunk metadata.**
- `source`: the document name, as used in citations.
- `location`: the article with its title and page range, e.g. `Article 16 (Organisational requirements), pp. 49-50`. A `words150` window that crosses two articles is labelled `Article 3 to Article 4`.
- Chunk number and word count.
- Documents added with `add_document` also store their file's SHA-256 and path.
- Search results show the citation tag `[source | location]`.

**Embedding and storage.**
- Chunks are embedded with `gemini-embedding-2` at 768 dimensions, normalised to unit length, as `RETRIEVAL_DOCUMENT`; questions are embedded as `RETRIEVAL_QUERY`.
- Vectors are written to ChromaDB (cosine distance) in `index/` after every batch of 50. A rerun embeds only chunks without a saved vector, and query vectors are cached in `index/query_vectors.json`.
- `index/` is 34 MB.

**Number of chunks.**

| Document | `sections` | `words150` |
|---|---|---|
| MiFID II | 291 | 633 |
| MiFIR | 145 | 304 |
| MiFID II Delegated Regulation | 191 | 393 |
| PRIIPs | 65 | 116 |
| **Total** | **692** (mean 251 words) | **1,446** (mean 149 words) |

The add-document demo later added 92 `sections` chunks for the PRIIPs RTS.

**Extraction problem fixed: false article headings.**
- *The problem.* The first heading pattern accepted any line starting with "Article N" or a number followed by text. Numbered paragraphs ("1. This Regulation establishes uniform requirements…") and cross-references at the start of a line ("Article 114 of the Treaty on the Functioning of the European Union…", "Article 4(1)(52) of Directive 2014/65/EU;") became new sections, and so did the Official Journal header ("31.3.2017 L 87/1 …").
- *The size of it.* MiFIR has 55 articles but produced 109 headings; the Delegated Regulation produced 240 headings for 91 articles.
- *Why it matters.* The location of every chunk after a false heading was wrong, so citations would have pointed to a paragraph or a cross-referenced act instead of the real article.
- *The fix.* A heading must be a line that is exactly `Article N` (optionally with a letter, e.g. `Article 4a`) or `ANNEX …`.
- *The result.* MiFIR now has 56 sections (55 articles plus Recitals) and the Delegated Regulation 96 (91 articles plus annexes and recitals). Every article in MiFIR, MiFID II and the Delegated Regulation now carries its title.

## Test set

**How it was written.**
- I started from summaries of the articles in the four PDFs (KID rules, suitability, best execution, reporting deadlines) and asked what a compliance analyst would need to check before advising or selling to a retail client.
- The questions were drafted with the help of an AI assistant (see *AI use*). I checked every reference answer and article number against the text.
- The questions are phrased as the analyst would ask them ("Can we…", "When must we…").

**Unanswerable questions.** I chose topics just outside the corpus, so they are close to its subject, not obviously off-topic:
- *Mentioned but not detailed in the corpus:* the summary risk indicator (q34) and the performance scenarios (q35), which PRIIPs leaves to the technical standards; and the CFD leverage limits (q38), where MiFIR only gives ESMA the power to intervene.
- *Later amendments:* the 2021 MiFID II quick fix (q39) and sustainability preferences (q37).
- *Law that is not in the corpus:* fines imposed by the Spanish CNMV (q36) and MiCA (q40).

**When it was written.** `eval/questions.csv` was committed on 21 November, before the evaluation ran the same day, and no question has been changed since (see the git history).

**Counts.**

| Kind | Questions |
|---|---|
| single (one passage) | 26 |
| multi (two passages) | 7 |
| unanswerable | 7 |
| **Total** | **40** |

By expected source: PRIIPs 14, MiFID II 13, MiFID II Delegated Regulation 9, MiFIR 4. Each multi-passage question counts once per source.

**Two examples.**

| id | Kind | Question | Expected | Reference answer |
|---|---|---|---|---|
| q13 | single | How long must recordings of client telephone conversations be kept? | MiFID II, Article 16 | Five years, and up to seven years if the competent authority requests it. |
| q27 | multi | What information must we collect from a client before giving investment advice, and what must the recommendation satisfy? | MiFID II, Article 25; MiFID II Delegated Regulation, Article 54 | Knowledge and experience, financial situation including ability to bear losses, and objectives including risk tolerance; the recommendation must meet the objectives, be financially bearable and understood, and nothing is recommended if nothing is suitable. |

## Retrieval

*Hit rate at 1/3/5 per design, words added to the prompt, the design you chose.*

## Tools and subagent

*Your tools' descriptions, one search result, the subagent's instructions, one transcript from your editor.*

 ▐▛███▛█   Claude Code v2.1.330
▝▜██████▀  Opus 5.5 · Claude Pro
 ▝▝   ▝▝   /workspaces/daiist-assignment-3-EB

▎ [✻] [✻] [✻] · Share Claude Code and earn $10 in usage credits · /passes

❯ /mcp 
  ⎿  MCP dialog dismissed

❯ /agents                                                                       
  ⎿  The /agents wizard has been removed.

     Ask Claude to create or update subagents for you (e.g. "create a code-reviewer subagent that ..."),
     or edit the files directly:
       • .claude/agents/       (this project)
       • ~/.claude/agents/     (all projects)

     Docs: https://code.claude.com/docs/en/sub-agents

❯ Which subagents are available in this project?                                

  Read 1 file, listed 1 directory · 1 failed (ctrl+o to expand)

● This project defines one subagent of its own, in
  .claude/agents/corpus-researcher.md:

  - corpus-researcher: answers questions about EU investment-services rules
    using the indexed legal texts: MiFID II (Directive 2014/65/EU), the MiFID II
    Delegated Regulation (EU) 2017/565, MiFIR (Regulation 600/2014) and the
    PRIIPs Regulation (1286/2014).
    - Tools: only mcp__corpus__search.
    - Rules: it answers only from the search results and cites every claim with
      the exact passage tag. If the corpus doesn't have the answer, it replies
      The corpus does not cover this question. It also labels partial answers
      and keeps replies short.
    - What it doesn't cover: it notes that national law, supervisors' practice,
      later amendments and other regulations (e.g. MiCA or the PRIIPs technical
      standards) usually aren't in the corpus.

  You have no user-level subagents in ~/.claude/agents/.

  Claude Code also comes with these general agents: general-purpose, Explore,
  Plan, claude-code-guide, statusline-setup, and a catch-all claude agent.

✻ Baked for 8s · done 6:17 PM

❯ Use the corpus-researcher subagent to answer: When must a suitability         
  statement be given to a retail client?                                        

● corpus-researcher(Suitability statement timing question)
  ⎿  Backgrounded agent (↓ to manage · ctrl+o to expand)

● The corpus-researcher subagent is now searching the corpus for this question.
  I'll share its answer, with citations, as soon as it finishes.

✻ Cooked for 5s · done 6:18 PM

› Message from corpus-researcher
  **Answer**

  **Timing.** When a firm gives investment advice, it must give the client a
  statement on suitability in a durable medium before the transaction is made.
  The statement must set out the advice given and how that advice meets the
  retail client's preferences, objectives and other characteristics [MiFID II |
  Article 25 (Assessment of suitability and appropriateness and reporting to
  clients), p. 62]. The Delegated Regulation also says the firm must give the
  retail client a report outlining the advice and explaining how the
  recommendation suits the client. That explanation covers the investment term
  required, the client's knowledge and experience, the client's attitude to risk
  and capacity for loss, and whether the client is likely to need a periodic
  review [MiFID II Delegated Regulation | Article 54 (Assessment of suitability
  and suitability reports), pp. 51-52]. Recital 83 asks firms to judge "in good
  time" by the client's need for enough time to read and understand information
  before deciding to invest [MiFID II | Recitals, p. 15].

  **Distance communication exception.** Sometimes the agreement to buy or sell
  is concluded by a means of distance communication that prevents giving the
  statement beforehand. In that case the firm may give the written statement in
  a durable medium immediately after the client is bound by any agreement, but
  only if both of these conditions are met:
  - (a) the client has agreed to receive the statement without undue delay after
  the transaction is concluded; and
  - (b) the firm has offered the client the option of delaying the transaction
  to receive the statement in advance.

  [MiFID II | Article 25 (Assessment of suitability and appropriateness and
  reporting to clients), pp. 62-63]

  **Periodic suitability.**
  - Where a firm provides portfolio management, or has told the client it will
  carry out a periodic suitability assessment, the periodic report must contain
  an updated statement of how the investment meets the client's preferences,
  objectives and other characteristics [MiFID II | Article 25 (Assessment of
  suitability and appropriateness and reporting to clients), pp. 62-63].
  - After the first report, later reports may cover only changes in the services
  or instruments and/or in the client's circumstances [MiFID II Delegated
  Regulation | Article 54 (Assessment of suitability and suitability reports),
  pp. 51-52].
  - Firms that provide a periodic suitability assessment must review the
  suitability of their recommendations at least once a year, and more often
  depending on the client's risk profile and the type of instruments [MiFID II
  Delegated Regulation | Article 54 (Assessment of suitability and suitability
  reports), p. 52].
  - Those firms must also disclose three things: how often and how far the
  assessment goes (including what triggers it), how far information already
  collected will be reassessed, and how an updated recommendation will be
  communicated [MiFID II Delegated Regulation | Article 52 (Information about
  investment advice), p. 49].

  The search results give no fixed deadline (number of days) for "without undue
  delay". They also say nothing about national rules or later amendments to
  these provisions.

  **Sources cited**
  - [MiFID II | Article 25 (Assessment of suitability and appropriateness and
  reporting to clients), p. 62]
  - [MiFID II | Article 25 (Assessment of suitability and appropriateness and
  reporting to clients), pp. 62-63]
  - [MiFID II Delegated Regulation | Article 54 (Assessment of suitability and
  suitability reports), pp. 51-52]
  - [MiFID II Delegated Regulation | Article 54 (Assessment of suitability and
  suitability reports), p. 52]
  - [MiFID II Delegated Regulation | Article 52 (Information about investment
  advice), p. 49]
  - [MiFID II | Recitals, p. 15]

● A suitability statement must be given before the transaction is made. When a
  firm gives investment advice, it must provide the statement in a durable
  medium. The statement sets out the advice and how it meets the retail client's
  preferences, objectives and other characteristics [MiFID II | Article 25
  (Assessment of suitability and appropriateness and reporting to clients), p.
  62]. The report must also explain how the recommendation suits the client,
  including the client's attitude to risk and capacity for loss [MiFID II
  Delegated Regulation | Article 54 (Assessment of suitability and suitability
  reports), pp. 51-52].

  Exception for distance communication: sometimes the agreement is concluded by
  a means of distance communication that prevents giving the statement
  beforehand. The firm may then give it immediately after the client is bound,
  but only if both of these are true:
  - the client agreed to receive it without undue delay after the transaction;
  - the firm offered the client the option of delaying the transaction to get
    the statement first.

  [MiFID II | Article 25 (Assessment of suitability and appropriateness and
  reporting to clients), pp. 62-63]

  Ongoing advice: this applies where the firm provides portfolio management or
  has promised a periodic suitability assessment.
  - Each periodic report must contain an updated suitability statement [MiFID II
    | Article 25 (Assessment of suitability and appropriateness and reporting
    to clients), pp. 62-63].
  - Suitability must be reviewed at least once a year [MiFID II Delegated
    Regulation | Article 54 (Assessment of suitability and suitability reports),
    p. 52].

  Not covered by the corpus: a fixed deadline in days for "without undue delay",
  national rules and later amendments.

✻ Cooked for 5s · done 6:18 PM

● Agent "Suitability statement timing question" finished · 22s




## Adding a document

*What happens when a document is already indexed. The unanswerable question, the document you added, the answer before and after.*

## Harness and guardrails

*Your AGENTS.md rules, the approval step, and your extra guardrail with one case where it fired.*

## Evaluation

*Code checks, the judge's rubric, your agreement with the judge, and results: correct, unsupported, correct refusals, wrong refusals.*

## Failures

*Failed questions by cause, with counts, and what to fix first.*

## Decision

*Would you let users rely on it, for what, with which safeguard? Cost and latency per question.*

## AI use

*What you used and how, or "none".*
