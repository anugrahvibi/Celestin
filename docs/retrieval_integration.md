# Retrieval service integration contract

The retrieval package combines the production SQLite repositories and FTS5
migrations with Member 2's search, ranking, and context builder. It does not
modify the MCP server or its public tools.

## Public operations

```python
from orbit.retrieval import search_chat_history, retrieve_context

search_chat_history(project: str, query: str, limit: int = 10) -> list[dict]
retrieve_context(project: str, query: str, token_budget: int = 2_000) -> dict
```

Both functions open and close a connection using `orbit.database.connect()`
for each call. The `RetrievalService(connection)` class is also exported for
callers that already own a SQLite connection; its methods return the internal
`SearchResult` and `ContextBundle` DTOs. The public wrappers serialize results
for service-boundary use.

```python
from orbit.database.engine import connect
from orbit.retrieval import RetrievalService
```

Search returns a list of objects shaped like:

```json
{
  "message_id": "17",
  "project": "my-project",
  "source_message_id": "17",
  "conversation_id": "conversation-1",
  "role": null,
  "content": "SQLite stores the project data.",
  "created_at": "2026-10-11T12:34:56.789012Z",
  "score": -0.000001
}
```

The matched message ID is the source ID. `conversation_id` comes from the
production FTS metadata. `role` is `null` because the current FTS mapping does
not store it. BM25 `score` is a relevance value, not a probability.

Context returns an object with `project`, `query`, `items`, `excluded`,
`excluded_counts`, `rendered_context`, `token_count`, `token_budget`,
`token_count_is_estimate`, and `token_counter`. Each included item preserves
`record_id`, `project_id`, `source_message_id`, `created_at`, `record_kind`,
`status`, `content`, and `relevance_score`. Canonical decisions without a
database `source_message_id` retain `null`; the service does not fabricate a
source ID. The builder's JSON Lines context still uses internal DTO keys
`source_id` and `timestamp`.

## Production FTS mapping

Schema migration 2 creates `chat_messages_fts` and `project_memory_fts`,
backfills existing rows, and installs insert/update/delete triggers. SQLite
FTS5 must be enabled. The service uses these mappings:

| Field | Messages index | Decisions index |
|---|---|---|
| table | `chat_messages_fts` | `project_memory_fts` |
| record ID | `message_id` | `decision_id` |
| project field | `project_id` | `project_id` |
| FTS source metadata | `source_id` (`conversation_id`) | `source_id` (indexed value; service reads authoritative `source_message_id`) |
| timestamp | `created_at` | `updated_at` |
| indexed content | `content` | `value` |

The service queries the production tables after `connect()` applies migrations;
callers do not create or manage FTS tables. Migration triggers maintain both
indexes for inserts, updates, and deletes.

## Candidate assembly and ranking

`search_chat_history` searches only messages scoped to the requested project.
`retrieve_context` searches the message and decision FTS indexes, then loads
authoritative decision rows with `DecisionRepository.list_for_project(project,
include_deprecated=True)`. It uses each FTS index's BM25 score; no score is
invented from decision status or timestamps. Message candidates use the
matched message ID as provenance. Decision provenance is the stored
`source_message_id`, or `null` if absent.

The service applies `RankingPolicy`: approved canonical decisions precede
historical messages independent of their BM25 values; drafts and deprecated
decisions are excluded. The service limits each FTS result set to 100 by
default; `candidate_limit` on `RetrievalService` changes that internal limit.

## Budget, errors, and limitations

The requested context budget is passed to `build_context_bundle`. Its default
counter estimates tokens by counting UTF-8 bytes and reports
`token_count_is_estimate=True`; it is not an exact model-token count. The
builder measures the complete rendered JSON Lines, never truncates a record,
and skips oversized records while continuing to later candidates.

An unknown project is indistinguishable from a valid project with no matches
because there is no project registry; both return empty results. Empty and
punctuation-only queries return empty results. Missing/incompatible FTS tables
raise `SearchIndexUnavailable`; database and repository errors propagate
rather than becoming successful empty results.

The retrieval operation returns a `list[dict]`. The production MCP
`RetrievalServices.search_chat_history` adapter wraps that list in the public
`{"project", "query", "results"}` envelope; the MCP tool contract is
documented in `docs/interfaces.md`.

## Tests

`tests/retrieval/test_service_integration.py` uses synthetic rows in temporary
databases created by the production migration code. It covers FTS
synchronization, project isolation, decision/message assembly, ranking,
traceability, public serialization, missing indexes, database errors, and
budget accounting. Run all tests with `PYTHONPATH=src python -m pytest -q`.
