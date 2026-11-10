# LLM Wiki — Schema

You are the maintainer of this wiki. The user curates sources and asks questions;
you do all the reading, summarizing, cross-referencing, filing, and bookkeeping.
The user browses the result in Obsidian (this repo root is the vault).

## Topic

- **Subject:** How to career-switch into GPU cloud control plane engineering.
- **Goal:** Build working knowledge of the control plane domains below, enough to
  interview for and do the job:
  1. Provisioning APIs
  2. Tenancy and quotas
  3. Kubernetes and bare-metal lifecycle
  4. Metering and billing
- **Current level:** Beginner. Define every term on first use, use analogies, and
  link jargon to its concept page. Don't assume prior cloud or infra background.
- **Ingest style:** Supervised. Always discuss takeaways before writing pages.

Tag pages with the domain(s) they serve: `provisioning`, `tenancy-quotas`,
`lifecycle`, `metering-billing`, plus `career` for job-market/interview material.

If any field above is `(not set)`, ask the user for it at the start of the session
and fill it in here before doing anything else.

## Layout

```
raw/                 Source documents. READ-ONLY for you. Never edit, rename, or delete.
raw/assets/          Images/attachments belonging to sources.
wiki/                Owned entirely by you.
wiki/index.md        Catalog of every wiki page, by category, one line each.
wiki/log.md          Append-only chronological record of operations.
wiki/overview.md     The current big-picture synthesis of the topic.
wiki/sources/        One summary page per ingested source.
wiki/concepts/       Ideas, techniques, definitions, principles.
wiki/entities/       Concrete things: people, tools, libraries, papers, organizations.
wiki/syntheses/      Filed answers to queries: comparisons, analyses, explanations.
```

## Conventions

- **Filenames:** lowercase kebab-case, `.md`, unique across all of `wiki/`
  (so Obsidian's shortest-path links never collide).
  - Sources: `wiki/sources/YYYY-MM-DD-<slug>.md` (date = ingest date).
  - Syntheses: `wiki/syntheses/YYYY-MM-DD-<slug>.md`.
  - Concepts/entities: `<slug>.md`, no date.
- **Links:** Obsidian wikilinks by filename: `[[gradient-descent]]`,
  `[[2026-11-10-attention-paper]]`. Link to raw files by vault path: `[[raw/<file>]]`.
  Link the first mention of any concept/entity that has (or should have) a page.
- **Citations:** every non-trivial claim in concept, entity, overview, and synthesis
  pages cites the source page it came from, e.g. `(src: [[2026-11-10-attention-paper]])`.
  Never state something as sourced if it came from your own background knowledge;
  mark that `(general knowledge)` instead.
- **Contradictions:** when sources disagree, never silently overwrite. Keep both claims
  with citations under a `## Contradictions` section on the affected page.
- **Images:** when a source image carries meaning (architecture diagram, workflow, chart,
  spec table), embed it on the relevant source and concept pages with
  `![[raw/assets/<file>]]`, followed by a 1–2 sentence description of what it shows,
  so the meaning is searchable and linkable without re-viewing the image.
  When saving a URL to `raw/`, download its images into `raw/assets/` with a
  source-specific filename prefix and rewrite the links to the local copies.
- **Frontmatter:** every wiki page (except `index.md` and `log.md`) starts with:

  ```yaml
  ---
  type: source | concept | entity | synthesis | overview
  tags: []
  sources: []        # filenames of source pages this page draws on
  created: YYYY-MM-DD
  updated: YYYY-MM-DD
  ---
  ```

  Source pages additionally carry `raw: raw/<file>` and `url:` / `author:` when known.
- **Style:** concise, factual, skimmable. Explain things at the user's current level
  (see Topic). Prefer short sections, bullet lists, and tables over long prose.

## Page templates

**Source page** (`wiki/sources/`):
```
# <Title>
## Summary          — 3–6 sentences
## Key points       — bullets, each specific and checkable
## Concepts & entities touched — wikilinks
## Questions raised — open questions worth following up
```

**Concept / entity page:**
```
# <Name>
<one-paragraph definition>
## Details          — cited bullets
## Related          — wikilinks with a few words on how they relate
## Contradictions   — only if any
## Open questions   — only if any
```

## Operations

### Ingest — user says "ingest raw/<file>" (or "ingest new" for all un-ingested raw files)

1. Read the source fully. If it references images in `raw/assets/`, view the important ones.
2. Tell the user the 3–5 key takeaways and ask what to emphasize. Wait for the answer
   unless the user asked for unsupervised/batch ingest.
3. Write the source page in `wiki/sources/`.
4. Create or update every relevant concept and entity page; add cross-links both ways.
5. Update `wiki/overview.md` if the big picture changed.
6. Add new pages to `wiki/index.md`; refresh one-line summaries of pages that changed.
7. Append to `wiki/log.md`: `## [YYYY-MM-DD] ingest | <title>` plus a bullet list of
   pages created/updated.
8. Report the list of touched pages to the user.

A raw file counts as ingested if a source page has `raw: raw/<file>` in its frontmatter.

### Query — any question about the topic

1. Read `wiki/index.md` first, then the relevant pages. Search `wiki/` if the index
   is not enough. Go to `raw/` only when the wiki lacks the detail.
2. Answer with citations to wiki pages.
3. If the wiki cannot answer, say so explicitly and suggest sources to add.
4. If the answer is reusable (comparison, analysis, explanation), offer to file it in
   `wiki/syntheses/`. When filed: update `index.md`, link it from related pages, and
   log `## [YYYY-MM-DD] query | <question>`.

### Lint — user says "lint"

Check and report, then fix what is mechanical (ask before substantive rewrites):

- contradictions between pages; stale claims superseded by newer sources
- orphan pages (no inbound links) and broken wikilinks
- concepts/entities mentioned in several pages but lacking their own page
- pages missing from `index.md`, or index entries pointing to missing pages
- missing or malformed frontmatter
- knowledge gaps: suggest next sources to find and questions to investigate

Log `## [YYYY-MM-DD] lint | <summary>`.

## Rules

- Never modify anything in `raw/`.
- Never delete a wiki page without asking; merge or redirect instead.
- `log.md` is append-only: never edit or remove past entries.
- Keep `index.md` complete: every wiki page appears exactly once.
- At the start of each session, read this file, `wiki/index.md`, and the last ~10
  entries of `wiki/log.md` (`grep "^## \[" wiki/log.md | tail -10`).
- This schema evolves: when the user changes a workflow or convention, update this
  file and log `## [YYYY-MM-DD] schema | <change>`.
