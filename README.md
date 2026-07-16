# pack-legal

Legal document intelligence for Embabel Me: **typed, graph-cached clause extraction** over ingested
contracts.

```cypher
MATCH (:Concept {value:'acme'})-[:RELEVANT_TO]->(d:Document)
MATCH (d)-[:HAS_CLAUSES]->(c:Clause)
WHERE c.category = 'Cap On Liability'
RETURN c.text
```

## What it ships

| File | What |
|---|---|
| `types/legal.yml` | The `Clause` type: CUAD-taxonomy `category`, verbatim `text`, normalized `answer`, `sourceChunkId` provenance; the `HAS_CLAUSES` join anchored on `Document`; few-shot text-to-cypher examples. |
| `producers/legal.yml` | `clauseExtraction` — a `kind: extract` producer (`cache: {kind: graph}`): collects the contract's chunks per document, extracts every clause as a typed record in one model call, persists each record as a committed scope-stamped node. |

## Behaviour

- **Extract once, query forever**: the first `HAS_CLAUSES` traversal extracts and persists; repeats
  are plain graph hits until the document's text changes, at which point the stale clause set is
  REPLACED (never orphaned).
- **Honest-empty**: a document with no recognizable clauses yields no rows — never fabricated ones.
- **Composes with the whole engine**: relevance chains (`RELEVANT_TO → HAS_CLAUSES`), parallel joins
  (`HAS_CLAUSES` + `HAS_SUMMARY` off one document), inline aggregation (`summarize(c.text, '…')` —
  "summarize the termination clauses"), subjective filtering (`WHERE c.ai_relevant = '…'`), and the
  demand budget (a `LIMIT 1` ask extracts only until satisfied).

## Clause vocabulary

The `category` values are CUAD's 41 expert labels (The Atticus Project, CC-BY-4.0) — Parties,
Agreement Date, Renewal Term, Governing Law, License Grant, Cap On Liability, Termination For
Convenience, Anti-Assignment, Audit Rights, Insurance, Source Code Escrow, … — the taxonomy
practicing lawyers use to review contracts, and the one the extraction prompt enumerates.

## Tests (in embabel/me)

- `ClauseExtractionFaithfulNeo4jIT` — engine contract (faked model): typed rows, per-record graph
  cache, stale-set replacement, honest-empty, steered-transient, cross-tenant, demand budget.
- `LegalPackFaithfulNeo4jIT` — THIS pack's YAML drives extraction end-to-end via `packDirs`.
- `ClauseCombinationsFaithfulNeo4jIT` — relevance chains, clause+summary composition,
  summarize-clauses-about-X, per-document digests.
- `ContractClausesLlmIT` (opt-in, real model) — extraction over the real Corio × Commerce One
  agreement (CUAD/EDGAR) asserted against CUAD's lawyer annotations.
