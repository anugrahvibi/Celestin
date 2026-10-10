# Orbit — Public MCP Tool Contracts

**Status:** Implementation-verified baseline for the current local stdio server. This document defines public tool arguments and response schemas; database and retrieval implementation details are documented in `docs/retrieval_integration.md` and `docs/retrieval.md`. Cross-layer policy choices and any public behavior changes still require team agreement.

## Ownership and compatibility

The MCP server owns tool names, public arguments, boundary validation, and serialization. It delegates persistence to the database/repository layer and search/ranking/context construction to the retrieval layer through separately agreed service interfaces. Tool handlers must not contain SQL, embedding, ranking, or token-selection logic. Every delegated call carries the requested `project` scope; the implementation must enforce that scope in the underlying operation, including source-message lookup.

The MCP server exposes the tools defined below and adapts the persistence and retrieval implementations to these public DTOs. The schemas are independent of storage names and remain the compatibility boundary for MCP clients.

## Shared conventions

- `project` is a required, non-empty string identifying the project namespace. The MCP boundary rejects whitespace-only values. The repository trims surrounding whitespace for persistence, while retrieval filters by the supplied value as-is; project IDs with surrounding whitespace are therefore not consistent across write and read paths. Until the team settles normalization policy, callers should send project IDs without surrounding whitespace. There is no project registry: any non-empty ID is a namespace, unknown-project reads return empty results, and ingest may create a namespace. All operations, including source-message resolution and retrieval candidates, are scoped to the requested project.
- Text arguments are Unicode strings. Required text must contain at least one non-whitespace character; preserve original text except where a tool's downstream search semantics apply. Reject wrong JSON types (including booleans where integers are required) and invalid values at the MCP boundary.
- MCP boundary limits (Unicode code points): `project` 200; `conversation_id` 512; `role` 32; message `content` 100,000; decision `key` 256; decision `value` 100,000; `source_message_id` 64; and `query` 4,096. Reject embedded NUL characters. `limit` is 1–100 (default 10); `token_budget` is 1–20,000.
- IDs are opaque, non-empty strings. Message IDs are stable IDs assigned by the persistence layer on ingestion and returned as `message_id`. They are project-scoped; clients must not infer their format. `source_message_id` refers to exactly such an ID, and must identify a message in the same requested project.
- New timestamps use UTC and the `Z` designator. Message `created_at` is the message insertion time. The decision table stores only `updated_at`, changed on save/upsert and status transition. The MCP `save_decision` adapter currently returns that same `updated_at` value as both `created_at` and `updated_at`; `created_at` is not a durable original-creation time and can change on an upsert. Context responses map the candidate timestamp to `created_at`; for canonical decisions that timestamp is the stored `updated_at`. Existing timestamp strings are passed through rather than normalized by the MCP adapter.
- Successful tool results are JSON objects matching the schemas below (MCP structured content). Error names below are stable categories; clients should not depend on backend exception text. Invalid arguments: `INVALID_ARGUMENT`; missing or cross-project source: `SOURCE_MESSAGE_NOT_FOUND`; configured retrieval/index unavailable: `RETRIEVAL_UNAVAILABLE`; persistence failure: `STORAGE_ERROR`; unexpected internal failure: `INTERNAL_ERROR`. Unknown project IDs are not an error because no project registry exists. Return an MCP tool error with a safe message and category; do not leak SQL, paths, or provider credentials.
- Error payload convention: `{ "error": { "code": "ERROR_CODE", "message": "safe human-readable explanation" } }`. The MCP SDK may encode this as an error result rather than successful structured content; the category and fields must remain equivalent.
- Arrays have deterministic order as described per tool. Empty results are successful empty arrays, not errors.

## `ingest_chat_message`

**Purpose:** Persist one chat message under a project and conversation, returning its stable source ID and creation time.

Signature: `ingest_chat_message(project: string, conversation_id: string, role: string, content: string) -> IngestChatMessageResult`

| Argument | Required | Type and validation |
|---|---:|---|
| `project` | yes | Non-empty project identifier; shared project rules apply. |
| `conversation_id` | yes | Non-empty opaque string scoped to `project`. |
| `role` | yes | One of `system`, `user`, or `assistant`, matching the database layer's accepted values. |
| `content` | yes | Non-empty after trimming check; original content is stored/returned without trimming. |

Result:

```json
{
  "project": "string",
  "message_id": "string",
  "conversation_id": "string",
  "role": "string",
  "content": "string",
  "created_at": "RFC3339 UTC timestamp"
}
```

Errors: `INVALID_ARGUMENT`, `STORAGE_ERROR`, `INTERNAL_ERROR`.

## `save_decision`

**Purpose:** Persist a project decision linked to the message that records its source.

Signature: `save_decision(project: string, key: string, value: string, source_message_id: string) -> SaveDecisionResult`

| Argument | Required | Type and validation |
|---|---:|---|
| `project` | yes | Non-empty project identifier; shared project rules apply. |
| `key` | yes | Non-empty decision key. |
| `value` | yes | Non-empty decision text/value. |
| `source_message_id` | yes | Non-empty message ID; must resolve to a message in the same project. |

Result:

```json
{
  "project": "string",
  "decision_id": "string",
  "key": "string",
  "value": "string",
  "status": "draft | approved | deprecated",
  "source_message_id": "string",
  "created_at": "RFC3339 UTC timestamp",
  "updated_at": "RFC3339 UTC timestamp"
}
```

Status behavior: the database accepts exactly `draft`, `approved`, and `deprecated`. MCP `save_decision` always creates or upserts a `draft`; it never approves a decision, and repeated saves reset the row to `draft`. The repository exposes `DecisionRepository.update_status(project, key, status)` for lifecycle transitions, but no MCP tool exposes that operation. The four-tool MCP API therefore cannot approve or deprecate decisions. Retrieval excludes drafts and deprecated decisions by default and prioritizes approved decisions over historical messages.

`DecisionRepository.save` permits a null source ID and the database column is nullable. The public MCP `save_decision` argument remains required and non-null. A source-less decision created through the repository can appear in `retrieve_context` with `source_message_id: null`.

Errors: `INVALID_ARGUMENT`, `SOURCE_MESSAGE_NOT_FOUND` (including cross-project source IDs), `STORAGE_ERROR`, `INTERNAL_ERROR`.

## `search_chat_history`

**Purpose:** Find matching chat messages in one project. Search semantics/ranking are owned by retrieval; this tool is only the public adapter.

Signature: `search_chat_history(project: string, query: string, limit?: integer = 10) -> SearchChatHistoryResult`

| Argument | Required | Type and validation |
|---|---:|---|
| `project` | yes | Non-empty project identifier; shared project rules apply. |
| `query` | yes | Non-empty after trimming check at the MCP boundary. Retrieval treats query as plain Unicode word terms joined with AND; punctuation is discarded and FTS operators/phrase syntax are not public. A punctuation-only query is valid at MCP but returns no matches. |
| `limit` | no | Integer, not boolean, range 1–100; default 10. This is enforced by the MCP boundary. |

Result:

```json
{
  "project": "string",
  "query": "string",
  "results": [
    {
      "message_id": "string",
      "project": "string",
      "source_message_id": "string",
      "conversation_id": "string | null",
      "role": "string | null",
      "content": "string",
      "created_at": "RFC3339 UTC timestamp",
      "score": "number"
    }
  ]
}
```

`source_message_id` is the ID of the matched message itself, allowing direct linking to `save_decision`; `message_id` is an equivalent explicit result identifier. `score` is the retrieval relevance value (FTS5 BM25 may be negative; smaller/more negative ranks first under the current implementation), not a probability. Results are ordered by retrieval rank and capped at `limit`. The adapter maps the retrieval result's message ID to `message_id` and `source_message_id`, its `source_id` (the conversation ID in chat search) to `conversation_id`, `content` to `content`, `timestamp` to `created_at`, and `relevance_score` to `score`. The current retrieval schema does not include message role, so `role` is `null`.

Errors: `INVALID_ARGUMENT`, `RETRIEVAL_UNAVAILABLE`, `INTERNAL_ERROR`.

## `retrieve_context`

**Purpose:** Return a bounded, traceable context bundle for a project/query, combining relevant messages and eligible canonical decisions.

Signature: `retrieve_context(project: string, query: string, token_budget: integer) -> RetrieveContextResult`

| Argument | Required | Type and validation |
|---|---:|---|
| `project` | yes | Non-empty project identifier; shared project rules apply. |
| `query` | yes | Non-empty after trimming check; retrieval semantics apply. |
| `token_budget` | yes | Positive integer, not boolean; MCP maximum is 20,000. The MCP argument is required. The direct Python retrieval service separately defaults to 2,000 when called without an explicit budget. |

Result:

```json
{
  "project": "string",
  "query": "string",
  "items": [
    {
      "record_id": "string",
      "source_message_id": "string | null",
      "created_at": "RFC3339 UTC timestamp",
      "record_kind": "chat_message | canonical_decision",
      "status": "draft | approved | deprecated | null",
      "content": "string",
      "relevance_score": "number"
    }
  ],
  "rendered_context": "string",
  "token_count": "integer",
  "token_budget": "integer",
  "token_count_is_estimate": "boolean",
  "token_counter": "string",
  "excluded": [
    {
      "record_id": "string",
      "source_message_id": "string | null",
      "created_at": "RFC3339 UTC timestamp",
      "record_kind": "chat_message | canonical_decision",
      "status": "draft | approved | deprecated | null",
      "reason": "token_budget | duplicate_content"
    }
  ]
}
```

The output item and exclusion fields map the retrieval `ContextBundle` DTO. The top-level `project` identifies the scope; timestamps map to `created_at`. For chat records, `source_message_id` is the record's own message ID. For canonical decisions, it is the stored source-message provenance ID and may be `null`. Context must contain only records from the requested project. Approved canonical decisions take priority in ordering over historical messages. Deprecated decisions are excluded by default; drafts are also excluded by default. The public tool has no inclusion flags.

Budget behavior: the complete serialized `rendered_context`, including metadata, must have `token_count <= token_budget`. Include whole records only; never truncate a record to fit. Scan ranked candidates in order, skip and report records that do not fit, and continue so a later smaller record can fit. Exact normalized-content duplicates may be skipped and reported. An empty context is valid, with count zero. The default counter is `utf8-byte-upper-bound`: it counts UTF-8 bytes in the complete rendered context and reports `token_count_is_estimate: true`; this is not an exact model-token count. The MCP maximum is 20,000. The public request has no default budget; 2,000 is the default only for direct Python retrieval calls.

Errors: `INVALID_ARGUMENT`, `RETRIEVAL_UNAVAILABLE`, `INTERNAL_ERROR`.

## Confirmed Baseline And Remaining Decisions

The following current behavior is covered by the database, retrieval, and MCP tests: message roles are `system`, `user`, and `assistant`; `search_chat_history(project, query, limit=10)` returns a list from retrieval and the MCP adapter wraps it as `{project, query, results}`; search results include the matched message/source IDs, project, conversation ID, null role, content, stored timestamp, and BM25 score. Search uses plain Unicode word terms joined with AND, discards punctuation, and does not expose FTS syntax. `limit` is 1–100 at the MCP boundary.

`retrieve_context(project, query, token_budget=2000)` is the direct Python retrieval signature; the MCP tool requires an explicit budget from 1 through 20,000. Both preserve project scope, rank approved decisions before messages, and exclude drafts/deprecated decisions by default. The default estimate is UTF-8 bytes over the full rendered context. The database/repository layer checks source-message IDs within the requested project. An unknown project is indistinguishable from an empty project because there is no registry; reads return empty results, and ingestion can create that namespace. The current baseline preserves that behavior unless the team explicitly approves a registry and changed error contract.

Before treating the cross-layer contract as final, the team still needs to decide:

1. Whether project IDs with surrounding whitespace should be rejected or normalized consistently. The current MCP boundary accepts them, repository validation trims them for persistence, and retrieval uses the supplied value unchanged.
2. Whether the MCP `save_decision` response should retain its current `created_at` compatibility field (which is an alias of `updated_at` and changes on upsert), or whether a future version should expose only the actual timestamp semantics. Do not change this public response without team approval.
3. Whether source-less canonical decisions should remain repository-only or whether a future MCP operation should accept a nullable `source_message_id`. Current `save_decision` requires a source; retrieved source-less decisions serialize `source_message_id` as `null`.
4. Who owns the administrative use of `DecisionRepository.update_status` for approvals/deprecations. The repository supports transitions, but the MCP API has no status-transition tool; `save_decision` always writes a draft.

For future remote access, the identity-to-project policy, read/write authorization, transport architecture, and permitted project data sent to ChatGPT remain open in `docs/authorization-design.md`. This local stdio contract does not settle those decisions.

The server currently supports local stdio transport only. Before exposing it over a network or other remote transport, add authentication and transport security, and review authorization and project access controls. Do not expose an unauthenticated remote endpoint.

No existing public function signature is changed by this document. The proposed four tools are new contracts; current `ping()` remains compatible. Any implementation proposal that changes these contracts must be brought to the team for approval before changing public signatures.
