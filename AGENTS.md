# AGENTS.md — sbi-ai-tools

Entry point for any AI tool (Claude Code, Cursor, Codex, ...) working in this repo.

## What this repo is

The knowledge base for the **SBI/TCS domain** in DAP, feeding a future AI triage
agent. See [README.md](README.md) for structure, provenance, and the service map.

## Ground rules

1. **Read the wiki first.** For any SBI/TCS domain question, start at
   [`index.md`](index.md) and follow `[[wikilinks]]` before reading code in
   [drivenets/dap](https://github.com/drivenets/dap).
2. **Schema-conformant writes only.** New pages must follow
   [`kc-config/sbi-tcs.schema.yaml`](kc-config/sbi-tcs.schema.yaml): correct
   `type`, sections in schema order, mandatory `## Source` citing real sources,
   `## See also` with in-vault `[[wikilinks]]`.
3. **Slugs are a contract.** Follow
   [`scripts/SLUG-CONVENTIONS.md`](scripts/SLUG-CONVENTIONS.md); never wikilink
   external repos (use markdown URLs); validate that every `[[wikilink]]`
   resolves before committing.
4. **Update indexes with content.** Adding/removing a page updates the category
   `index.md` and the root `index.md` page counts in the same commit.
5. **No images in wiki pages.** Diagrams as prose, tables, or ASCII arrows —
   images defeat grep/search retrieval.
6. **Honest claims.** Mark unverified statements as open questions or
   `UNRESOLVED (candidate: ...)`; never upgrade an inference to a fact.
