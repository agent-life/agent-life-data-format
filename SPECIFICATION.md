# Agent Life Format (ALF) — Specification

**Version:** 1.0.0-rc.1  
**Date:** 2026-02-26  
**Status:** Release Candidate  
**Repository:** https://github.com/agent-life/agent-life-data-format  
**Site:** https://agent-life.ai  
**License:** MIT

---

### Abstract

The Agent Life Format (ALF) is an open, portable serialization format for the complete durable state of an AI agent — memories, identity, relationships, and credentials. ALF enables agent backup, migration between runtimes, and incremental sync to cloud storage without information loss.

ALF is designed as a **runtime-neutral superset**: it captures the union of data models used by today's agent frameworks so that exporting from any supported runtime preserves all information. The format is intentionally decoupled from any specific LLM provider, embedding model, or execution engine.

### Conformance

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

### Notation

- **§** references point to sections within this document (e.g., §3.1.1).
- JSON examples are illustrative, not normative — the JSON Schema definitions in the `schemas/` directory are the authoritative machine-readable specification.
- Field types in schema tables use JSON type names (`string`, `integer`, `array`, `object`).

---

## 1. Purpose and Scope

This document specifies the **Agent Life Format (ALF)** — an open, portable serialization format for the complete durable state of an AI agent. The format satisfies two goals:

1. **Storage-efficient for backup and sync** — compact enough for incremental transfer to an off-host service.
2. **Query-efficient for direct use** — structured enough that a cloud agent runtime could use the neutral store as its primary persistence layer (future goal).

ALF is designed as a **superset** of the data models used by today's major agent runtimes, so that exporting from any supported runtime loses no information. The initial target runtimes are **OpenClaw** and **ZeroClaw**, with the architecture open to additional adapters.

**Scope boundaries — what ALF stores and what it does not:**

ALF captures an agent's **knowledge state** (memories, learned facts, preferences, relational links), **identity state** (persona, behavioral configuration, capabilities, sub-agent roster), **relationship state** (principals, user context), and **credentials** (encrypted). These are the durable, portable layers that define who an agent is and what it knows — the things that matter when backing up, migrating, or restoring an agent.

ALF **explicitly excludes active execution state**: in-flight tool calls, pending workflow steps, loop counters, call stacks, LangGraph checkpoints, queued callbacks, and any other transient runtime process state. Execution state is inherently runtime-specific (a LangGraph checkpoint cannot be restored into an OpenClaw runtime, and vice versa), ephemeral (it exists only for the duration of a task), and not part of the agent's durable identity or knowledge. Attempting to serialize execution state would couple the format to every runtime's internal execution model, violating the core design philosophy of runtime neutrality.

**Practical implication:** Migrating or restoring an agent mid-task will abort the in-progress task. The agent retains all knowledge accumulated up to the point of export (including memories from the current session), but any unfinished multi-step workflow is lost. This is the correct tradeoff — execution state is cheap to recreate (re-run the task), while knowledge and identity are expensive to recreate (months of accumulated context).

---

## 2. Background — Current Landscape

*This section is informational. It summarizes the data models of existing agent frameworks to motivate the design choices in subsequent sections.*

### 2.1 OpenClaw Data Model

OpenClaw stores all agent state as plain Markdown files in a workspace directory. The key files are:

| File | Layer | Description |
|------|-------|-------------|
| `SOUL.md` | Identity | Personality, tone, values, boundaries. Freeform prose loaded every session. |
| `IDENTITY.md` | Identity | Structured profile: name, role, goals, voice. |
| `AGENTS.md` | Identity / Procedural | Operating instructions, priorities, workflow rules, memory management rules. |
| `USER.md` | User Context | Communication preferences, timezone, work context, known constraints. |
| `MEMORY.md` | Memory (long-term) | Curated persistent facts and compressed history. Recommended <200 lines. |
| `memory/YYYY-MM-DD.md` | Memory (daily) | Per-day working notes. Auto-preprompted; older logs retrieved via semantic search. |
| `TOOLS.md` | Capabilities | Notes about available local tools and conventions (guidance, not enforcement). |
| `HEARTBEAT.md` | Procedural | Checklist for periodic autonomous runs. |
| `BOOT.md` | Procedural | Startup checklist executed on gateway restart. |
| `BOOTSTRAP.md` | Procedural | One-time first-run interview script. Deleted after setup. |

Memory search is hybrid: 70% vector (semantic via sqlite-vec) + 30% BM25 keyword (FTS5). The Markdown files are the source of truth; vector indexes are derivative.

Credentials live in `~/.openclaw/` (config, sessions, environment variables), not in the workspace.

### 2.2 ZeroClaw Data Model

ZeroClaw uses a TOML config (`~/.zeroclaw/config.toml`) and supports multiple memory backends:

| Backend | Format | Features |
|---------|--------|----------|
| SQLite (default) | `memories.db` | Hybrid search (FTS5 + vector cosine), categories (`conversation`, `daily`, `core`), embedding cache. |
| Markdown | `memory/*.md` | Daily files + session files, archive directory, hygiene system. OpenClaw-compatible layout. |
| Lucid | CLI sync + SQLite fallback | External memory service bridge. |
| None | No-op | Stateless mode. |

Identity supports two formats via `[identity] format`:

- `"openclaw"` — reads SOUL.md, IDENTITY.md, USER.md, AGENTS.md from workspace (OpenClaw-compatible).
- `"aieos"` — inline or file-referenced AIEOS v1.1 JSON with typed fields for psychology, linguistics, motivations.

Credentials are encrypted at rest using ChaCha20-Poly1305 with a local `.secret_key`. OAuth tokens and auth profiles are stored in `.zeroclaw/state/auth_profiles.json`, also encrypted.

Memory tools: `memory_store`, `memory_recall`, `memory_forget` (agent-invoked, auto-save optional).

### 2.3 AIEOS v1.1 (AI Entity Object Specification)

AIEOS is a JSON schema standard focused on **identity portability**. Core structure:

- **Identity & Physicality** — names, bio, somatotype, facial features, aesthetic archetypes.
- **Psychology & Neural Matrix** — normalized 0.0–1.0 cognitive drivers, OCEAN personality traits, moral alignment.
- **Linguistics & Idiolect** — vocal acoustics, syntax style, formality level, verbal tics, slang usage.
- **History & Motivations** — origin story, life events, professional background, core drive.
- **Capabilities & Skills** — standardized tools and executable functions with priority scale (1–10).

AIEOS covers the identity/persona layer well but does not address experiential memory, user context, or credentials.

### 2.4 Mem0 Memory Architecture

Mem0 defines a memory-centric approach with three pillars:

- **State** — what's happening right now (working context).
- **Persistence** — knowledge retained across sessions.
- **Selection** — deciding what's worth remembering.

Key distinctions from RAG: memory is stateful, user-aware, temporally aware, and adaptive. Mem0 stores memories as embeddings (the embedding *is* the primary copy, unlike OpenClaw where Markdown is the source of truth).

### 2.5 LangChain/LangGraph Memory Taxonomy

LangChain categorizes long-term memory into three types drawn from cognitive science:

- **Semantic memory** — facts and knowledge. Can be stored as a continuously-updated profile (JSON document) or a collection of individual memory documents with semantic search.
- **Episodic memory** — past experiences and actions. Typically implemented as few-shot examples for task learning.
- **Procedural memory** — instructions and rules. Stored as system prompts, tool descriptions, or agent instructions that can be updated over time.

Memory can be written "in the hot path" (synchronously during interaction) or "in the background" (asynchronously after interaction). Both approaches have tradeoffs in latency, consistency, and completeness.

### 2.6 AWS AgentCore Memory

AgentCore implements a multi-stage long-term memory pipeline:

1. **Extraction** — LLM analyzes conversations to extract memories per strategy (semantic facts, user preferences, conversation summaries).
2. **Consolidation** — new memories are compared against existing memories via semantic similarity; the system ADDs, UPDATEs, or NO-OPs to avoid duplicates and resolve conflicts. Outdated memories are marked INVALID (immutable audit trail) rather than deleted.
3. **Retrieval** — semantic search returns relevant memories in ~200ms.

Compression rates of 89–95% are achieved (memory tokens vs. full conversation history tokens), which is critical for staying within context windows.

---

## 3. Data Layer Specification

The neutral format organizes agent state into four layers, each with its own schema, sensitivity, and sync characteristics.

### 3.1 Layer 1 — Memory and Experience

This is the highest-volume, most frequently updated layer. It must capture memories from all observed runtime formats without information loss.

#### 3.1.1 Memory Record Schema

Each memory record MUST contain:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string (UUID v7) | Yes | Globally unique, time-sortable identifier. |
| `agent_id` | string (UUID) | Yes | The agent this memory belongs to. |
| `content` | string | Yes | The memory text. Plain text or Markdown. |
| `memory_type` | enum | Yes | One of: `semantic`, `episodic`, `procedural`, `preference`, `summary`. See §3.1.2. |
| `category` | string | No | Runtime-defined category (e.g. ZeroClaw's `conversation`, `daily`, `core`). Free-form, for adapter round-trip fidelity. |
| `source` | object | Yes | Provenance. See §3.1.3. |
| `temporal` | object | Yes | Temporal metadata. See §3.1.4. |
| `status` | enum | Yes | One of: `active`, `superseded`, `archived`, `deleted`. |
| `supersedes` | string (UUID) | No | ID of the memory this record replaces (for consolidation audit trail). |
| `confidence` | float (0.0–1.0) | No | Confidence score if the source runtime provides one. |
| `entities` | array of object | No | Extracted entities with type and name. See §3.1.5. |
| `tags` | array of string | No | User or agent-defined tags for filtering. |
| `namespace` | string | Yes | Scoping namespace. Default: `"default"`. Enables separation of user-context memories, project-scoped memories, etc. |
| `embeddings` | array of object | No | Pre-computed embedding vectors, tagged by model. See §3.1.6. |
| `related_records` | array of object | No | Links to other memory records with typed relationships. See §3.1.12. |
| `raw_source_format` | object | No | Opaque blob preserving the original runtime-specific representation for lossless round-trip. |

#### 3.1.2 Memory Types

The type taxonomy is drawn from the LangChain/cognitive-science model, extended with `preference` and `summary` from AgentCore:

- **`semantic`** — facts, knowledge, learned information. ("User's company has 500 employees." "The project deadline is March 15.")
- **`episodic`** — records of specific interactions or events. ("On Feb 10, user asked me to rewrite the API docs and preferred bullet-point format." "Agent successfully deployed the staging environment using the custom Docker config.")
- **`procedural`** — rules, instructions, workflows. ("When user says 'ship it', run the deploy pipeline." "Always ask for confirmation before deleting files.")
- **`preference`** — explicit and inferred user preferences. ("User prefers short answers." "User likes Thai food.") This overlaps with semantic but is separated for frameworks (like AgentCore) that treat preferences as a distinct strategy.
- **`summary`** — condensed narratives of conversations or time periods. Maps to OpenClaw's MEMORY.md entries and AgentCore's summary strategy.

#### 3.1.3 Source Provenance

Every memory must record where it came from:

```json
{
  "source": {
    "runtime": "openclaw",
    "runtime_version": "1.2.0",
    "origin": "daily_log",
    "origin_file": "memory/2026-02-10.md",
    "extraction_method": "agent_written",
    "session_id": "abc123",
    "interaction_id": "msg_456",
    "identity_version": 3
  }
}
```

- `runtime` — which agent framework produced this memory.
- `origin` — the native storage location or mechanism (e.g. `daily_log`, `memory_md`, `sqlite_store`, `agent_tool_call`).
- `extraction_method` — how the memory was created: `agent_written` (agent wrote it during interaction), `llm_extracted` (extracted from conversation by a post-processing LLM), `user_authored` (human directly edited the file), `migrated` (imported from another format).
- `identity_version` — (integer, optional) the identity layer version that was active when this memory was created. Links what the agent did to who the agent was at the time, making memory-to-identity lineage explicit rather than requiring timestamp correlation. See §3.1.10.

#### 3.1.4 Temporal Metadata

Temporal context is critical for memory consolidation and conflict resolution:

```json
{
  "temporal": {
    "created_at": "2026-02-10T14:30:00Z",
    "updated_at": "2026-02-15T09:00:00Z",
    "observed_at": "2026-02-10T14:25:00Z",
    "valid_from": "2026-02-10T00:00:00Z",
    "valid_until": null,
    "last_accessed_at": "2026-02-20T11:00:00Z",
    "access_count": 7
  }
}
```

- `created_at` — when the memory record was created in the neutral format.
- `updated_at` — last modification time.
- `observed_at` — when the underlying fact or event was originally observed (may differ from created_at if extracted asynchronously).
- `valid_from` / `valid_until` — temporal validity window. A `null` `valid_until` means "still current." Used for preferences that change over time.
- `last_accessed_at` / `access_count` — usage statistics for relevance scoring and memory decay.

#### 3.1.5 Entity References

For memory records that reference named entities:

```json
{
  "entities": [
    { "name": "Acme Corp", "type": "organization", "role": "employer" },
    { "name": "Jane", "type": "person", "role": "colleague" }
  ]
}
```

Entity types: `person`, `organization`, `project`, `location`, `tool`, `service`, `other`.

#### 3.1.6 Embeddings (Store-and-Tag Approach)

**Design decision:** The format stores embeddings as-is from the source runtime and tags them explicitly by model, rather than standardizing on a single "neutral" embedding model. This avoids coupling the format to a specific vendor, eliminates expensive bulk re-embedding on initial migration, and ages well as embedding models improve.

**Rationale:**

- Standardizing a neutral embedding model would force a full re-embed of every memory record on first sync — potentially thousands of records and a non-trivial API cost. Incremental ingestion would only need to embed new records, but the initial migration cost is the real barrier.
- More fundamentally, blessing a specific model (e.g., `text-embedding-3-small`) is a **policy decision disguised as a format decision**. It couples the spec to a vendor API, a fixed dimensionality, and a quality/cost tradeoff that degrades as models improve. Embedding models advance rapidly; locking the format to one forces mass re-embedding on every "blessed model" upgrade.
- The store-and-tag approach pushes embedding decisions to the edges (adapters and the sync service), where they belong. The format is a faithful archive, not an opinionated runtime.

**Schema:** Each memory record MAY contain zero or more embeddings, keyed by model:

```json
{
  "embeddings": [
    {
      "model": "openai/text-embedding-3-small",
      "dimensions": 1536,
      "vector": [0.012, -0.034, ...],
      "computed_at": "2026-02-10T14:30:00Z",
      "source": "runtime"
    }
  ]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `model` | string | Yes | Embedding model identifier in `provider/model` format. |
| `dimensions` | integer | Yes | Vector dimensionality. |
| `vector` | array of float | Yes | The embedding vector. |
| `computed_at` | ISO 8601 timestamp | Yes | When this embedding was computed. |
| `source` | enum | Yes | Who computed it: `runtime`, `sync_service`, `import_adapter`. |

**Behavioral rules:**

- **On export:** The adapter writes exactly one entry per record — whatever the runtime already has. Zero extra computation. Records with no runtime-computed embeddings (e.g., OpenClaw's markdown-first model where vector indexes are derivative) have an empty array.
- **On import:** The adapter checks for an embedding matching its preferred model. If one exists, it uses it directly (no re-embedding). If not, it re-embeds from `content` into its preferred model. This is the same work runtimes already do when building indexes from raw content.
- **On the sync service (future):** The service MAY lazily add a second embedding in a service-standard model space for its own query index, without modifying or removing the original entry. This enables the future cloud runtime query API without forcing all clients onto one model.
- **Model transitions:** If an agent switches embedding models mid-life (e.g., user upgrades from `openai/ada-002` to `openai/text-embedding-3-small`), both vectors coexist in the array. The importer picks the one it prefers.
- **Embeddings are always derivative.** They can be recomputed from `content` at any time. Adapters and sync services MUST NOT treat embeddings as authoritative data — the `content` field is the source of truth.

#### 3.1.7 Daily Logs (OpenClaw Compatibility)

OpenClaw's daily log pattern (`memory/YYYY-MM-DD.md`) is a special case. The neutral format represents these as:

- One memory record per daily log file, with `memory_type: "episodic"`, `category: "daily_log"`, and the full Markdown content preserved in `content`.
- Alternatively, an adapter MAY decompose a daily log into multiple fine-grained memory records if the extraction pipeline supports it. The `raw_source_format` field preserves the original file for lossless round-trip.

#### 3.1.8 Memory Consolidation Metadata

To support AgentCore-style consolidation:

- When a memory is updated (merged with new information), the old record's status changes to `superseded` and the new record's `supersedes` field references the old record's ID.
- The full chain of supersession is preserved, enabling audit and rollback.
- Deleted memories are soft-deleted (`status: "deleted"`) with a tombstone, not physically removed, to support sync conflict resolution.

#### 3.1.9 Workspace Artifact References

Agent workspaces contain files beyond the agent's core memory and identity — generated code, data files, images, documents, and other artifacts the agent produced or referenced during tasks. These need to be classified and handled differently from managed agent state.

**Three-tier model:**

The format distinguishes three tiers of workspace files, based on their relationship to the agent's operational state — not on whether they are binary or text:

**Tier 1 — Managed agent state.** Runtime files the agent reads and writes as part of its core operation. Examples: OpenClaw's SOUL.md, MEMORY.md, shares_docs.md; ZeroClaw's config.toml, memories.db. These are the agent's mind — small, actively curated, and essential to runtime behavior. They go in `raw/{runtime}/` and are represented in the structured layers (memory records, identity, principals). **Already handled by the existing spec — no changes needed.**

**Tier 2 — Portable artifacts.** Small workspace files worth carrying with the agent across migrations. Examples: a short script the agent wrote, a small data file it references frequently, a brief CSV it maintains. These are under a size threshold (default: **100 KB per file**) and provide useful context if the agent migrates to a new runtime. Too large to inline in a memory record's `content` field, but small enough that excluding them loses value. These are stored in the `artifacts/` directory within the archive and referenced from the artifact index.

**Tier 3 — Referenced artifacts.** Large workspace files that would bloat the archive. Examples: a 10,000-line CSV, a 50-file Python codebase, a collection of images, large PDFs. Cataloged by reference only (filename, hash, size, source path). The agent's *knowledge about* them is captured in memory records; the files themselves stay behind. A future version of the sync service can store these online (via the `remote_ref` field) without a schema change.

**Why not binary vs. text?** The original design used "binary file" as a proxy for "large, not agent state." But a 10,000-line CSV is text and would bloat the archive just as badly as a PNG. Conversely, a small icon used in a generated UI is binary but trivially portable. The meaningful distinction is managed state vs. artifact, and small vs. large — not binary vs. text.

**Schema:** The artifact index is a top-level file in the ALF archive (`attachments.json`):

```json
{
  "artifact_size_threshold": 102400,
  "attachments": [
    {
      "id": "uuid",
      "filename": "shares_tracker.csv",
      "media_type": "text/csv",
      "size_bytes": 4200,
      "hash": {
        "algorithm": "sha256",
        "value": "a1b2c3d4..."
      },
      "source_path": "workspace/shares_tracker.csv",
      "archive_path": "artifacts/shares_tracker.csv",
      "remote_ref": null,
      "referenced_by": ["memory-record-uuid-1"]
    },
    {
      "id": "uuid",
      "filename": "architecture-diagram.png",
      "media_type": "image/png",
      "size_bytes": 284190,
      "hash": {
        "algorithm": "sha256",
        "value": "e3b0c442..."
      },
      "source_path": "workspace/images/architecture-diagram.png",
      "archive_path": null,
      "remote_ref": null,
      "referenced_by": ["memory-record-uuid-2", "memory-record-uuid-3"]
    }
  ]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `artifact_size_threshold` | integer | No | The size threshold (in bytes) used to classify Tier 2 vs. Tier 3. Default: 102400 (100 KB). Present at the top level for documentation and reproducibility. |
| `id` | string (UUID) | Yes | Unique identifier for this artifact reference. |
| `filename` | string | Yes | Original filename. |
| `media_type` | string | Yes | MIME type (e.g., `image/png`, `text/csv`, `text/x-python`). |
| `size_bytes` | integer | Yes | File size at time of export. |
| `hash` | object | Yes | Content hash. `algorithm` (default `sha256`) + `value`. |
| `source_path` | string | Yes | Path relative to the runtime workspace where the file was found. |
| `archive_path` | string or null | Yes | Path within the ALF archive if the file is included (Tier 2). `null` if reference-only (Tier 3). |
| `remote_ref` | string or null | Yes | URL for future online storage. `null` in initial releases. |
| `referenced_by` | array of string (UUIDs) | No | Memory record IDs that reference this artifact. |

**Behavioral rules:**

- **On export:** The adapter walks the workspace and classifies each non-runtime file. Files that are part of the runtime's managed state (Tier 1) go in `raw/` as before. Remaining files under the size threshold (Tier 2) are copied into `artifacts/` and their `archive_path` is set. Files over the threshold (Tier 3) are cataloged by reference only (`archive_path: null`). The adapter SHOULD report the artifact inventory to the user, distinguishing included (Tier 2) from reference-only (Tier 3).
- **On import:** The adapter extracts Tier 2 artifacts from `artifacts/` and places them in the target workspace. For Tier 3 (reference-only) artifacts, the adapter presents the index so the user knows which files are not included. Memory records that reference unresolved artifacts are imported normally — the `referenced_by` linkage is the mechanism for identifying incomplete context.
- **Size threshold:** 100 KB is the default. Adapters MAY allow users to configure a different threshold. The threshold used is recorded in `artifact_size_threshold` so consumers know how the classification was made.
- **Future online storage:** When the sync service adds artifact storage, it populates `remote_ref` with a content-addressable URL. The hash enables integrity verification on download. This requires no schema changes — only a transition from `null` to a URL. Tier 3 artifacts become downloadable without changing their index entries.

#### 3.1.10 Memory-to-Identity Lineage

Agents evolve. An agent might act differently on Day 100 than on Day 1 because its system prompt or identity configuration was updated by the user. Episodic and procedural memories are particularly sensitive to this — if a memory says "I refused to answer this question" but the current identity says "answer all questions," it looks like a contradiction without context.

The identity version timeline and memory timestamps are both present in the format, so lineage *could* be reconstructed by correlating timestamps. But implicit correlation is fragile: it requires maintaining a separate index of "identity version N was active from T1 to T2" and doing range lookups. If an identity was updated twice in one session, time resolution might not disambiguate.

**Solution:** The `source` object on each memory record includes an optional `identity_version` integer field (see §3.1.3). On export, the adapter stamps the current identity version onto each new memory record. This makes lineage explicit and queryable — "show me all memories created under identity v3" is a simple filter, and "this refusal made sense because the identity at that time said X" is a direct lookup into the identity version history.

**Design notes:**

- The field is on all memory records uniformly, not just episodic/procedural. The schema is simpler, and it's useful to know "this fact was learned during identity v3" even if the fact itself isn't identity-dependent.
- The field is optional to maintain backward compatibility. Records without it (e.g., migrated from a runtime that doesn't version identity) simply have unknown lineage, which is the same as the pre-lineage state.
- The identity version referenced here is the same integer from the identity layer's existing versioning scheme (§3.2.3). No new versioning mechanism is needed.

#### 3.1.11 Hard Deletion and Data Purge

**Problem:** Section 3.1.8 defines soft deletes (tombstones) as the default deletion mechanism, and §7.2.2 states that sealed partitions are immutable. However, soft deletes are legally insufficient when a user exercises GDPR Article 17 ("right to erasure"), CCPA deletion rights, or simply needs to purge accidentally memorized sensitive data (passwords, government IDs, medical information). The format and sync protocol must support hard deletion that physically removes content.

**Two deletion scopes:**

**1. Full agent deletion** — the most common compliance request. A user asks for their entire account or a specific agent to be permanently erased. This is straightforward: the sync service destroys all objects associated with the agent (all partitions, identity, principals, credentials, deltas, the metadata index entry, and any compacted snapshots). No partition rewriting is needed — everything is deleted wholesale. The service retains only a minimal audit record (see below). Self-hosted deployments delete the `.alf` file directly.

**2. Surgical record purge** — less common but necessary. A user identifies specific memory records containing sensitive data that must be physically removed while the rest of the agent's state is preserved. This requires rewriting the affected partition.

**Surgical purge operation:**

The sync service (or a local tool for self-hosted deployments) performs the following:

1. Identify the target records by ID and locate which partition(s) contain them.
2. Read the affected partition, filter out the target records, and write a **new partition** with the same time range and `sealed: true`. The old partition is not modified — it is replaced.
3. Update the manifest to reference the new partition (new file path, updated record count, new checksum).
4. Destroy the old partition blob from object storage.
5. Purge any delta blobs that contain the target records.
6. Remove the target records from any search indexes or metadata caches.
7. Emit an audit record (see below).

This preserves the mental model that any given partition is immutable — you never mutate one, you replace it. Caches keyed on partition hash naturally invalidate because the replacement has a different hash.

**Audit trail:** Both deletion scopes produce an audit record that proves the deletion occurred without revealing the purged content:

```json
{
  "purge_id": "uuid",
  "agent_id": "uuid",
  "scope": "full_agent | record_purge",
  "record_ids": ["uuid-1", "uuid-2"],
  "partitions_affected": ["memory/partitions/2025-Q3.jsonl"],
  "reason": "gdpr_article_17 | ccpa_deletion | user_request | security_incident",
  "requested_at": "2026-03-01T10:00:00Z",
  "completed_at": "2026-03-01T10:00:05Z",
  "requested_by": "principal-uuid"
}
```

For full agent deletions, `record_ids` is omitted (everything was deleted) and `partitions_affected` lists all partitions that existed. The audit record itself MUST NOT contain any of the purged content. Audit records are retained according to the service's compliance retention policy.

**Relationship to soft deletes:** Soft deletes (§3.1.8) remain the default for normal agent operation — they're cheap, reversible, and support sync conflict resolution. Hard purge is a separate, exceptional operation triggered by explicit user request or compliance workflow. The two mechanisms coexist: a soft-deleted record still physically exists in its partition (and could be restored); a hard-purged record is physically gone.

**Format-level requirement:** The `.alf` container format itself does not need a special structure for hard deletes — they operate at the service and storage layer. However, adapters and tools that produce `.alf` files locally MUST support a `purge` command that rewrites the archive excluding specified record IDs, for self-hosted deployments that don't use the sync service.

#### 3.1.12 Memory Relational Links

Advanced memory systems model memories as graphs, not just lists. A failed task (episodic) might trigger a new rule (procedural). Two memories might contradict each other without one superseding the other. A summary might elaborate on a collection of episodic memories. The format should preserve these relationships for runtimes that use them.

**Schema:** Each memory record MAY contain an array of typed links to other records:

```json
{
  "related_records": [
    {
      "id": "target-memory-uuid",
      "relation": "caused_by"
    },
    {
      "id": "another-memory-uuid",
      "relation": "contradicts"
    }
  ]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string (UUID) | Yes | ID of the related memory record. |
| `relation` | string | Yes | The type of relationship. Free-form — runtimes may define their own relation types. Well-known values are documented below. |

**Well-known relation types:**

| Relation | Meaning | Example |
|----------|---------|---------|
| `caused_by` | This memory was created as a direct result of the linked memory. | A procedural rule created because a task failed (the episodic failure record). |
| `caused` | Inverse of `caused_by`. The linked memory was created as a result of this one. | The episodic failure record linking forward to the rule it prompted. |
| `contradicts` | This memory conflicts with the linked memory. Neither supersedes the other — the contradiction is unresolved or intentional. | Two user preferences that conflict ("User likes spicy food" vs. "User asked to avoid spicy food last week"). |
| `elaborates_on` | This memory provides additional detail or context for the linked memory. | A detailed episodic memory linked to a summary that covers the same event. |
| `derived_from` | This memory was synthesized or extracted from the linked memory. | A summary derived from a collection of episodic memories. |
| `related_to` | General association without a specific directional meaning. | Two memories about the same project that are contextually related. |

**Relationship to `supersedes`:** The `supersedes` field (§3.1.1) remains a separate top-level field because it has specific lifecycle semantics — it changes the old record's `status` to `superseded` and is used by compaction. `related_records` is for informational and navigational links that don't affect record lifecycle. A memory can both supersede another record (via `supersedes`) and have relational links to other records (via `related_records`).

**Behavioral rules:**

- Relation types are **free-form strings**, not a closed enum. The well-known values above are conventions, not constraints. Runtimes that invent their own relationship types (LangGraph causal chains, custom reasoning graphs) can use whatever labels are meaningful, and the format preserves them faithfully.
- Links are **one-directional** — each entry points from this record to the target. Bidirectional relationships (like `contradicts`) should be recorded on both records when possible, but readers MUST NOT assume symmetry.
- Adapters for runtimes that don't use graph relationships simply ignore the field. Adapters for runtimes that do should preserve and populate it.
- Dangling references (links to records that don't exist in the archive, e.g., due to a surgical purge) are tolerated. Readers SHOULD handle missing targets gracefully.

### 3.2 Layer 2 — Identity and Persona

This layer captures who the agent is and how it behaves.

#### 3.2.1 Dual Representation Requirement

The fundamental tension is between prose-based identity and structured identity. The format MUST support both:

1. **Structured fields** — typed, machine-readable, portable. Based on AIEOS v1.1 as the starting point.
2. **Prose blocks** — rich, contextual, interpreted by the LLM at runtime. Based on OpenClaw's SOUL.md / IDENTITY.md pattern.

An adapter importing into a prose-based runtime (OpenClaw) uses the prose blocks. An adapter importing into a structured runtime (ZeroClaw with AIEOS) uses the structured fields. If one representation is missing, the adapter MAY synthesize it (e.g., LLM-generate a SOUL.md from structured fields), but this is adapter behavior, not a format requirement.

#### 3.2.2 Identity Schema

```json
{
  "identity": {
    "id": "uuid",
    "agent_id": "uuid",
    "version": 3,
    "updated_at": "2026-02-15T09:00:00Z",

    "structured": {
      "names": {
        "primary": "Nova",
        "nickname": "N",
        "full": "Nova Assistant"
      },
      "role": "Personal AI assistant",
      "goals": ["Help user be productive", "Learn user preferences over time"],

      "psychology": {
        "neural_matrix": {
          "creativity": 0.9,
          "logic": 0.8,
          "empathy": 0.85,
          "assertiveness": 0.6,
          "curiosity": 0.95
        },
        "personality_traits": {
          "framework": "OCEAN",
          "scores": {
            "openness": 0.9,
            "conscientiousness": 0.75,
            "extraversion": 0.6,
            "agreeableness": 0.8,
            "neuroticism": 0.2
          }
        },
        "moral_alignment": "Neutral Good",
        "mbti": "ENTP"
      },

      "linguistics": {
        "formality_level": 0.4,
        "verbosity": 0.5,
        "humor_level": 0.6,
        "slang_usage": false,
        "preferred_language": "en",
        "idiolect": {
          "catchphrases": [],
          "verbal_tics": [],
          "avoided_words": ["honestly", "genuinely"]
        }
      },

      "capabilities": [
        {
          "name": "web_search",
          "description": "Search the web for current information",
          "priority": 2,
          "portability": "intrinsic"
        },
        {
          "name": "creative_writing",
          "description": "Write fiction, poetry, marketing copy",
          "priority": 1,
          "portability": "intrinsic"
        },
        {
          "name": "docker_management",
          "description": "Build and manage Docker containers",
          "priority": 3,
          "portability": "host_dependent",
          "host_requirements": "Docker Engine installed and accessible via CLI"
        },
        {
          "name": "github_repo_reader",
          "description": "Read and analyze GitHub repositories",
          "priority": 2,
          "portability": "intrinsic",
          "credential_ids": ["credential-uuid-for-github-token"]
        }
      ],

      "aieos_extensions": {}
    },

    "prose": {
      "soul": "Full content of SOUL.md or equivalent...",
      "operating_instructions": "Full content of AGENTS.md or equivalent...",
      "identity_profile": "Full content of IDENTITY.md or equivalent...",
      "custom_blocks": {
        "boot_checklist": "Content of BOOT.md...",
        "heartbeat_checklist": "Content of HEARTBEAT.md...",
        "tools_guidance": "Content of TOOLS.md..."
      }
    },

    "source_format": "openclaw",
    "raw_source": {}
  }
}
```

#### 3.2.3 Identity Versioning

Identity MUST be versioned. Every change increments the version number and records a timestamp. The sync service retains prior versions for rollback. This enables the "Framework Evaluation" use case — snapshot identity before a trial migration, restore on return.

#### 3.2.4 Sub-Agent Roster

A managing agent that orchestrates sub-agents needs to persist a structured roster of those sub-agents — what they can do, when to use them, and what they ran on. This is distinct from free-text memories *about* sub-agents because it must be structured enough for programmatic routing decisions, not just LLM context.

The sub-agent roster is stored in the identity layer because it describes the agent's operational capabilities and relationships, not its experiential memory.

**Schema:** The roster is an array within `identity.structured`:

```json
{
  "sub_agents": [
    {
      "agent_id": "uuid-or-null",
      "name": "Research Assistant",
      "description": "Deep research agent with multi-source synthesis",
      "capabilities": [
        {
          "name": "web_research",
          "description": "Deep research with source synthesis",
          "priority": 1
        },
        {
          "name": "summarization",
          "description": "Condense long documents",
          "priority": 3
        }
      ],
      "model_hints": {
        "primary_model": "anthropic/claude-sonnet-4-5",
        "last_model": "anthropic/claude-sonnet-4-5"
      },
      "routing_hints": "Use for any task requiring multi-source research or fact verification. Not suitable for creative writing.",
      "status": "active",
      "last_invoked_at": "2026-02-20T11:00:00Z",
      "performance_notes": "Excellent at technical topics. Occasionally over-cites sources."
    }
  ]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `agent_id` | string (UUID) or null | Yes | References the sub-agent's own `.alf` if it has one. `null` for ephemeral sub-agents that the managing agent creates and destroys per task. |
| `name` | string | Yes | Display name of the sub-agent. |
| `description` | string | No | Brief summary of what this sub-agent does. |
| `capabilities` | array of object | Yes | What the sub-agent can do, with `name`, `description`, and `priority` (1 = highest). Used by the managing agent for task routing. |
| `model_hints` | object | No | `primary_model` (model the sub-agent spent most of its life on) and `last_model` (most recently used). See §3.2.5. |
| `routing_hints` | string | No | Prose guidance for when and how to use this sub-agent. Interpreted by the managing agent's LLM. |
| `status` | enum | Yes | One of: `active`, `inactive`, `unavailable`. |
| `last_invoked_at` | ISO 8601 timestamp | No | When the managing agent last delegated a task to this sub-agent. |
| `performance_notes` | string | No | The managing agent's observations about the sub-agent's strengths and weaknesses. |

**Behavioral rules:**

- The roster records what the managing agent *knows about* its sub-agents. It does NOT contain the sub-agent's state — each agent gets its own `.alf` file.
- `agent_id` is nullable because not all sub-agents have their own persistent state. Ephemeral tool-agents (created per task, destroyed after) are still worth recording in the roster if the managing agent has learned routing heuristics about them.
- On restore to a new host, the sub-agents will not exist. The roster gives the managing agent the information it needs to recreate them on request: what capabilities are needed, what model tier they require, and how they should behave.

#### 3.2.5 Model Hints

Model information is recorded as **hints** — factual metadata about what the agent ran on, not prescriptive requirements. The format does not dictate model choices (consistent with design philosophy #1), but preserving this information helps users and adapters set appropriate expectations.

**Main agent model hints** are stored in the manifest (§4.2) as `runtime_hints`, since the model is a runtime configuration choice rather than part of the agent's identity:

```json
{
  "runtime_hints": {
    "primary_model": "anthropic/claude-opus-4-5",
    "last_model": "anthropic/claude-opus-4-5",
    "provider": "anthropic",
    "context_window": 200000,
    "notes": null
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `primary_model` | string | Yes | The model the agent spent most of its life on, in `provider/model` format. Represents the capability baseline the persona was shaped by. |
| `last_model` | string | Yes | The most recently used model. May differ from `primary_model` if the agent was temporarily switched for cost or availability reasons. |
| `provider` | string | No | The LLM provider (e.g., `anthropic`, `openai`, `openrouter`). |
| `context_window` | integer | No | Context window size in tokens. Relevant because it affects memory management — an agent tuned for 200K tokens will have different boot file and MEMORY.md sizing expectations than one running in 32K. |
| `notes` | string or null | No | Free-form notes about the runtime configuration. |

**Sub-agent model hints** use the same `primary_model` / `last_model` pair (see §3.2.4) without the provider-level fields, since sub-agents may use different providers than the managing agent.

#### 3.2.6 AIEOS Compatibility (Compatible Superset with Opinionated Field Promotion)

AIEOS v1.1 is the only existing open standard for AI agent identity. ZeroClaw supports it natively. Rather than building a parallel schema, the ALF identity layer is a **compatible superset** of AIEOS — every AIEOS field can be stored and round-tripped — but we selectively **promote** only the fields that map to real agent behavior into first-class structured fields. Everything else is preserved in the `aieos_extensions` passthrough object.

**Promoted to first-class fields** (these appear in `identity.structured` with their own schema):

| AIEOS Field | ALF Field | Rationale |
|------------|-----------|-----------|
| `identity.names` | `structured.names` | Core identity — every agent has a name. |
| `psychology.neural_matrix` | `structured.psychology.neural_matrix` | AIEOS-specific but actively used by ZeroClaw for behavior tuning. Compact, useful. |
| `psychology.traits.ocean` | `structured.psychology.personality_traits` (with `framework: "OCEAN"`) | OCEAN/Big Five is well-established personality science with validated psychometric properties. |
| `linguistics.text_style` | `structured.linguistics` | Directly controls agent output behavior (formality, verbosity). |
| `linguistics.idiolect` | `structured.linguistics.idiolect` | Catchphrases, verbal tics, avoided words — concrete behavioral specification. |
| `motivations.core_drive`, `motivations.goals` | `structured.goals` | Functional — drives agent planning and prioritization. |
| `capabilities.skills`, `capabilities.tools` | `structured.capabilities` | Functional — used for task routing and tool discovery. Annotated with portability (see below). |

**Capability portability (intrinsic vs. host-dependent):**

Capabilities are stored in the identity layer because the agent needs to know what it can do for task routing and planning. However, not all capabilities are equally portable across environments. A `creative_writing` capability is intrinsic to the LLM — it works everywhere. A `docker_management` capability depends on the host having Docker installed — it disappears when the agent migrates to a host without it.

Rather than splitting capabilities into a separate "environment" layer (which would need to be re-populated on every deployment and conflates portable agent state with ephemeral host configuration), each capability carries a `portability` annotation:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `portability` | string | No | `intrinsic` (LLM-native, always available regardless of host) or `host_dependent` (requires specific host environment). Default: `intrinsic`. |
| `host_requirements` | string | No | For `host_dependent` capabilities, a brief description of what the host must provide (e.g., "Docker Engine accessible via CLI", "macOS with AppleScript support"). Informational — helps the importing adapter and user understand what's needed. |
| `credential_ids` | array of string (UUIDs) | No | IDs of credentials (from Layer 4, §3.4.2) required by this capability. The reverse linkage — credential to capability — is on the credential schema via `capabilities_granted`. Both directions are optional and informational. |

On migration, the identity doesn't change — the agent still *knows about* Docker. The importing adapter checks host-dependent capabilities against the target environment and reports which ones are unavailable. The runtime can then mark them as inactive so the agent doesn't attempt to use them, but the knowledge is preserved for future environments that do support them. Similarly, `credential_ids` tells the adapter which credentials to wire up for each capability — if a required credential is missing or expired, the adapter can report the gap.

**Passed through via `aieos_extensions`** (stored for lossless round-trip, not given first-class fields):

| AIEOS Field | Reason for Passthrough |
|------------|----------------------|
| `psychology.traits.mbti` | MBTI is widely regarded as lacking psychometric validity by personality researchers. OCEAN covers the same ground with better scientific basis. Promoting MBTI to a named field implicitly endorses it as a meaningful personality measure. |
| `psychology.moral_compass.alignment` (D&D-style, e.g., "Chaotic Good") | Game terminology, not a serious ethical framework. For real agents, the prose layer (SOUL.md) captures ethical boundaries and values far more richly. |
| `identity.physicality` (somatotype, facial features, aesthetic archetypes) | Designed for virtual avatars and character roleplay. Most practical agents have no physical presence. Not relevant to the backup/migration use case. |
| `identity.bio` (gender, age_biological) | Fictional character metadata. Harmless as passthrough data, but not warranting first-class schema structure. |
| `identity.origin` (nationality, birthplace) | Same — character backstory, not functional agent state. |

**How this works in practice:**

- The **ZeroClaw adapter** maps promoted fields bidirectionally between ALF's `identity.structured` and AIEOS JSON. Non-promoted fields are written to/from `aieos_extensions` for lossless round-trip.
- The **OpenClaw adapter** uses the prose representations. If importing an agent that only has structured identity (AIEOS-origin, no prose), the adapter MAY synthesize prose from the structured fields.
- If AIEOS evolves to add fields that map to real agent behavior, they can be promoted in a future ALF minor version without breaking existing snapshots (the new fields simply didn't exist before).

### 3.3 Layer 3 — Principals and User Context

#### 3.3.1 Principals Model

An agent may serve multiple **principals** — entities it takes direction from and maintains relationship context for. A principal can be a human user or another agent (in the case of sub-agents serving a managing agent). This generalizes the original single-user model to support agent-to-agent relationships.

Each principal has its own profile and preference history. The agent's accumulated experience with each principal (communication style learned, preferences observed, interaction history) is preserved independently.

```json
{
  "principals": [
    {
      "id": "uuid",
      "principal_type": "human",
      "agent_id": null,
      "profile": { "...see §3.3.3..." }
    },
    {
      "id": "uuid",
      "principal_type": "agent",
      "agent_id": "uuid-of-managing-agent",
      "profile": { "...see §3.3.3..." }
    }
  ]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string (UUID) | Yes | Unique identifier for this principal relationship. |
| `principal_type` | enum | Yes | `human` or `agent`. |
| `agent_id` | string (UUID) or null | No | For agent-type principals, the managing agent's ID (resolvable to another `.alf` if it exists). `null` for human principals. |
| `profile` | object | Yes | The principal's profile. Same schema as §3.3.3. |

#### 3.3.2 Separation from Memory

Principal context is stored as a **distinct, queryable namespace** within the memory layer (`namespace: "principal_context:{principal_id}"`), plus the dedicated profile document per principal. This dual approach accommodates:

- Frameworks that separate user context (OpenClaw's `USER.md`) — the primary human principal's profile maps directly.
- Frameworks that merge user context into memory — individual preference memories in the principal's namespace are indistinguishable from regular memories.
- Agent-to-agent relationships — a sub-agent's memories about its managing agent live in that principal's namespace.

For backwards compatibility and simplicity, a single-user agent (the common case) has exactly one principal of type `human`. Adapters for frameworks that have no multi-principal concept import/export only the primary human principal.

#### 3.3.3 Principal Profile Schema

```json
{
  "id": "uuid",
  "agent_id": "uuid",
  "principal_id": "uuid",
  "version": 5,
  "updated_at": "2026-02-20T11:00:00Z",

  "structured": {
    "name": "Alex",
    "principal_type": "human",
    "timezone": "America/Los_Angeles",
    "locale": "en-US",
    "communication_preferences": {
      "tone": "casual",
      "response_length": "concise",
      "formatting": "minimal markdown"
    },
    "work_context": {
      "role": "Software engineer",
      "company": "Acme Corp",
      "projects": ["Project Alpha", "Backend migration"]
    },
    "relationships": [],
    "custom_fields": {}
  },

  "prose": {
    "user_profile": "Full content of USER.md or equivalent..."
  },

  "source_format": "openclaw",
  "raw_source": {}
}
```

For agent-type principals, the `work_context` and `communication_preferences` fields capture how the managing agent prefers to interact — what formats it expects results in, how it delegates tasks, its escalation preferences.

#### 3.3.4 Versioning

Same versioning rules as identity (§3.2.3). Each principal's profile is versioned independently.

### 3.4 Layer 4 — Accounts and Credentials

This layer is security-critical and uses a zero-knowledge architecture.

#### 3.4.1 Encryption Requirements

- Credentials MUST be stored as **encrypted blobs**. The sync service stores only ciphertext it cannot decrypt.
- Encryption uses **user-controlled keys**. The key material never leaves the user's devices or is transmitted to the sync service.
- Recommended cipher: XChaCha20-Poly1305 (consistent with ZeroClaw's existing ChaCha20-Poly1305 choice, extended nonce for safety at scale).
- Each credential blob includes an **encryption metadata header** (key derivation parameters, algorithm identifier) stored in plaintext alongside the ciphertext.

#### 3.4.2 Credential Record Schema

```json
{
  "credential": {
    "id": "uuid",
    "agent_id": "uuid",
    "service": "openai",
    "credential_type": "api_key",
    "label": "OpenAI Production Key",
    "capabilities_granted": ["web_search", "code_generation"],
    "encrypted_payload": "base64-encoded-ciphertext",
    "encryption": {
      "algorithm": "xchacha20-poly1305",
      "kdf": "argon2id",
      "kdf_params": {
        "memory_cost": 65536,
        "time_cost": 3,
        "parallelism": 4
      },
      "nonce": "base64-encoded-nonce"
    },
    "created_at": "2026-01-15T00:00:00Z",
    "updated_at": "2026-02-10T00:00:00Z",
    "last_rotated_at": "2026-02-10T00:00:00Z",
    "expires_at": null,
    "tags": ["production", "llm-provider"]
  }
}
```

The `capabilities_granted` field (optional) lists the capability names (from `identity.structured.capabilities`) that this credential enables. This tells the importing adapter which tools to wire this credential to. Many credentials are general-purpose (an OpenAI API key powers multiple capabilities), and some don't map to specific capabilities at all — the field is informational and omitted when the linkage is unclear.

The reverse linkage — capability to credential — is on the capability schema via `credential_ids` (see §3.2.6).

#### 3.4.3 Supported Credential Types

The `credential_type` field supports: `api_key`, `oauth_token` (with refresh token in the encrypted payload), `webhook_secret`, `session_token`, `ssh_key`, `certificate`, `custom`.

#### 3.4.4 Credential Lifecycle

- Adapters handle decryption and injection into the target runtime's native credential storage on import.
- On export, adapters encrypt credentials using the user's key before writing to the neutral format.
- The sync service MUST NOT log, index, or inspect credential payloads.

---

## 4. Container Format

### 4.1 File Format

A complete agent-life snapshot is a **single file** with the extension `.alf` (Agent Life Format). It is a standard **ZIP archive** containing:

```
agent-snapshot.alf
├── manifest.json          # Format version, agent metadata, layer inventory
├── identity.json          # Layer 2: Identity and persona
├── principals.json        # Layer 3: Principals (users and agent relationships)
├── credentials.json       # Layer 4: Encrypted credential records
├── memory/
│   ├── index.json         # Partition inventory and memory-level metadata
│   ├── partitions/
│   │   ├── 2025-Q3.jsonl  # Sealed partition: Jul–Sep 2025 (immutable)
│   │   ├── 2025-Q4.jsonl  # Sealed partition: Oct–Dec 2025 (immutable)
│   │   └── 2026-Q1.jsonl  # Current partition (mutable until sealed)
│   └── embeddings/        # Optional: pre-computed embedding vectors
│       └── vectors.bin    # Binary format for compactness
├── attachments.json       # Workspace artifact index (Tier 2 included, Tier 3 reference-only)
├── artifacts/             # Portable workspace artifacts under size threshold (Tier 2)
│   ├── shares_tracker.csv
│   └── deploy_script.sh
└── raw/                   # Lossless originals from source runtime (Tier 1)
│   ├── openclaw/
│   │   ├── SOUL.md
│   │   ├── AGENTS.md
│   │   ├── USER.md
│   │   ├── MEMORY.md
│   │   └── memory/
│   │       └── 2026-02-10.md
│   └── zeroclaw/
│       ├── config.toml    # With credentials stripped/encrypted
│       └── memories.db    # SQLite database copy
```

#### 4.1.1 Time-Based Memory Partitioning

Memory records are partitioned by time period. The default partition granularity is **quarterly** (e.g., `2025-Q3.jsonl`), but adapters MAY use monthly granularity for high-activity agents. Each partition is a JSONL file containing all memory records whose `temporal.created_at` falls within that period.

**Sealed vs. unsealed partitions:**

- A **sealed** partition covers a completed time period. Its contents are immutable — new records from that period cannot be added (in practice, all records for a past quarter already exist). Sealed partitions can be cached aggressively by clients and the sync service.
- The **current (unsealed)** partition covers the ongoing time period. It is the only partition that changes during normal operation. New memory records are appended here; status changes to existing records (supersession, deletion) create new records in the current partition that reference the old record's ID.

**Why this matters:** For a years-old agent, a full export is still one ZIP, but most of it is sealed partitions that haven't changed since they were first written. A client that already has last quarter's sealed partitions only needs to download the current partition and any newly sealed ones. This directly maps to efficient cloud storage (§7.2) — sealed partitions are written once to object storage and never touched again.

### 4.2 Manifest Schema

```json
{
  "alf_version": "1.0.0",
  "created_at": "2026-02-24T12:00:00Z",
  "agent": {
    "id": "uuid",
    "name": "Nova",
    "source_runtime": "openclaw",
    "source_runtime_version": "1.2.0"
  },
  "runtime_hints": {
    "primary_model": "anthropic/claude-opus-4-5",
    "last_model": "anthropic/claude-opus-4-5",
    "provider": "anthropic",
    "context_window": 200000,
    "notes": null
  },
  "sync": {
    "last_sequence": 1247,
    "last_sync_at": "2026-02-24T12:00:00Z"
  },
  "layers": {
    "identity": { "version": 3, "file": "identity.json" },
    "principals": { "count": 1, "file": "principals.json" },
    "credentials": { "count": 4, "file": "credentials.json" },
    "memory": {
      "record_count": 1247,
      "index_file": "memory/index.json",
      "has_embeddings": true,
      "has_raw_source": true,
      "partitions": [
        { "file": "memory/partitions/2025-Q3.jsonl", "from": "2025-07-01", "to": "2025-09-30", "record_count": 342, "sealed": true },
        { "file": "memory/partitions/2025-Q4.jsonl", "from": "2025-10-01", "to": "2025-12-31", "record_count": 518, "sealed": true },
        { "file": "memory/partitions/2026-Q1.jsonl", "from": "2026-01-01", "to": null, "record_count": 387, "sealed": false }
      ]
    },
    "attachments": {
      "count": 12,
      "included_count": 3,
      "included_size_bytes": 42800,
      "referenced_count": 9,
      "referenced_size_bytes": 48248234,
      "file": "attachments.json"
    }
  },
  "raw_sources": ["openclaw"],
  "checksum": "sha256:abcdef..."
}
```

### 4.3 Incremental Sync Format

For incremental syncs (not full snapshots), the format uses a **delta bundle** — a smaller ZIP containing only changed records since a specified sync point:

```
delta-seq-1248.alf-delta
├── manifest.json          # Includes sync cursor (sequence number + timestamp)
├── identity.json          # Only present if identity changed
├── principals.json        # Only present if principals changed
├── memory/
│   └── delta.jsonl        # Only new/modified/deleted memory records
└── credentials.json       # Only present if credentials changed
```

Each delta record includes an `operation` field: `create`, `update`, `delete` (soft-delete with tombstone).

#### 4.3.1 Sync Cursors — Sequence Numbers

The sync protocol uses **monotonic sequence numbers** as the primary cursor for delta operations, not timestamps. The server assigns a sequence number to each delta on receipt.

```json
{
  "sync": {
    "base_sequence": 1247,
    "new_sequence": 1253,
    "base_timestamp": "2026-02-24T12:00:00Z",
    "new_timestamp": "2026-02-24T18:30:00Z"
  }
}
```

**Why sequence numbers over timestamps:**

- No clock skew ambiguity between client and server.
- No "at vs. after" boundary confusion — sequence 1248 unambiguously means "the first change after 1247."
- The server assigns sequence numbers on receipt, so ordering is always authoritative.
- Timestamps are preserved as data on each record (they're meaningful for memory temporal metadata), but the sync cursor is a sequence number.

**Client sync flow:** The client reports "I last synced at sequence 1247. Here are my changes since then." The server assigns them sequence numbers 1248–1253, stores them, and returns the new cursor. To catch up, the client requests "give me everything after sequence 1247" and receives deltas 1248+.

The client lumps all local changes since last sync into a single delta. This is the simplest correct approach and matches how every incremental backup tool works. The delta is already small for typical usage patterns (under 100 KB after a session).

---

## 5. Design Decisions and Rationale

### 5.1 Why JSON (not SQLite, not Protobuf)?

- **Readability and debuggability** — JSON is human-inspectable, critical for an open specification. Developers can examine and hand-edit agent state.
- **Ecosystem compatibility** — every language and tool speaks JSON. No special readers needed.
- **JSONL for memory** — streaming-friendly for large memory stores. Each line is a self-contained record.
- **Binary embeddings as opt-in** — vectors are stored in a separate binary file to avoid bloating JSON. They are always re-derivable from content.

SQLite is used **within runtimes** (ZeroClaw, AgentCore) but is not suitable as an interchange format because it's opaque, platform-dependent at the byte level, and not streaming-friendly.

### 5.2 Why ZIP?

- Standard, universally supported, tool-friendly.
- Supports per-file access without full extraction (important for the sync service to read manifests without decompressing all memory).
- Compression is free.

### 5.3 Why UUID v7 for Memory IDs?

UUID v7 is time-ordered, which means records sort naturally by creation time without a separate index. This is valuable for incremental sync (fetch all records after timestamp X) and for conflict resolution (later timestamp wins by default).

### 5.4 Why Dual Representation for Identity?

The prose vs. structured tension is real and cannot be resolved by choosing one side. OpenClaw's SOUL.md contains nuanced behavioral instructions that lose meaning when decomposed into OCEAN scores. ZeroClaw's AIEOS profiles contain precise numeric parameters that lose precision when rendered as prose. The neutral format preserves both and lets each adapter use whichever it supports.

### 5.5 Raw Source Preservation

The `raw/` directory contains unmodified copies of the source runtime's native files. This guarantees lossless round-trip: if you export from OpenClaw and later import back to OpenClaw, the adapter can use the raw files directly instead of synthesizing from the neutral representation. This is a safety net against information loss in the structured translation.

---

## 6. Adapter Interface Requirements

Each framework adapter implements the following operations:

### 6.1 Export (Runtime → ALF)

```
export(workspace_path, options) → alf_snapshot
```

The adapter reads the runtime's native state and produces a complete ALF snapshot. Requirements:

- MUST populate all four layers if data exists in the runtime.
- MUST preserve raw source files in `raw/`.
- MUST map runtime-specific memory categories to the neutral `memory_type` taxonomy (§3.1.2). Unmappable entries use the `category` free-form field for fidelity.
- MUST encrypt credentials before including them in the snapshot.
- SHOULD extract entities from memory content where feasible.
- SHOULD include existing embeddings from the runtime, tagged by model (see §3.1.6). Do not compute new embeddings on export — store only what the runtime already has.
- MUST classify workspace files into the three-tier model (§3.1.9): Tier 1 (managed state) to `raw/`, Tier 2 (portable artifacts under size threshold) to `artifacts/`, Tier 3 (large artifacts) as reference-only in `attachments.json`. SHOULD report the artifact inventory to the user, distinguishing included from reference-only.
- SHOULD stamp the current identity version onto each memory record's `source.identity_version` field (see §3.1.10). This is a low-cost operation — the adapter already knows the current identity version.

### 6.2 Import (ALF → Runtime)

```
import(alf_snapshot, workspace_path, options) → result
```

The adapter reads an ALF snapshot and writes it into the target runtime's native format. Requirements:

- MUST check for and prefer `raw/` files matching the target runtime (lossless shortcut).
- MUST translate between identity representations (prose ↔ structured) if the raw source is from a different runtime. MAY use an LLM for prose synthesis.
- MUST inject credentials into the target runtime's native credential storage after decryption.
- MUST re-index memory into the target runtime's search system (rebuild FTS indexes, etc.). For vector indexes, the adapter SHOULD reuse embeddings matching its preferred model from the `embeddings` array (§3.1.6) and only re-embed from `content` when no compatible embedding is available.
- MUST handle memory type mapping gracefully: unmapped types are preserved as `category` metadata.
- MUST extract Tier 2 artifacts from `artifacts/` and place them in the target workspace. For Tier 3 (reference-only) artifacts, the adapter MUST present the index so the user knows which files are not included (§3.1.9). Memory records with unresolved artifact references are imported normally.
- SHOULD check `host_dependent` capabilities against the target environment and report which ones are unavailable (§3.2.6). Unavailable capabilities are preserved in the identity (the agent retains the knowledge) but the runtime SHOULD mark them as inactive so the agent does not attempt to use them.
- SHOULD support a dry-run mode that reports what would change without writing.

### 6.3 Delta Export (for Incremental Sync)

```
export_delta(workspace_path, since_sequence, options) → alf_delta
```

Produces a delta bundle containing only changes since the last sync point, identified by sequence number (§4.3.1). The adapter tracks the last-synced sequence locally and includes all changes since then in a single delta.

### 6.4 Purge (for Hard Deletion)

```
purge(alf_path, record_ids, options) → purged_alf
```

Rewrites a local `.alf` file excluding the specified record IDs. Required for self-hosted deployments that need GDPR/CCPA compliance without the sync service. See §3.1.11.

- MUST remove the specified records from all memory partitions.
- MUST update the manifest with new partition checksums and record counts.
- MUST produce a valid ALF archive as output.
- MUST emit an audit record (to stdout or a separate file) listing purged record IDs, affected partitions, and timestamp — without including any purged content.
- SHOULD support a dry-run mode that reports which partitions would be affected and how many records would be removed.

---

## 7. Sync Protocol Requirements

### 7.1 Sync Cadence Options

Users choose their cadence: `after_interaction`, `after_session`, `scheduled` (cron), `manual`. The format and protocol must support all modes.

### 7.2 Storage and Transport Considerations

The format and protocol must support an efficient cloud storage model. Detailed service architecture is deferred to a separate design document, but the following high-level constraints shape format-level decisions and MUST NOT be precluded by the schema or protocol.

#### 7.2.1 Event-Sourcing Storage Model

The recommended cloud storage pattern treats **deltas as events** and **snapshots as projections**:

- **Deltas** (small, frequent) are appended to an ordered event log. Each delta is a small object in blob storage (S3, R2, GCS), typically under 100 KB. Storage cost is negligible — pennies per month for even an active agent. The event log is append-only; deltas are never modified after they are written.
- **Snapshots** (large, infrequent) are produced by **compaction** — the service periodically merges all deltas since the last snapshot into a new full snapshot. This is a background batch job (weekly or monthly, depending on activity), not on the hot path of any client request.
- **The metadata index** is a lightweight database (Postgres for multi-tenant SaaS, SQLite for self-hosted) that tracks: which deltas exist per agent, their sequence numbers, which snapshot they compact into, and partition seal status. The actual bulk data lives in object storage, which is 10–100x cheaper per GB than database storage.

**Read path:** A client requesting a full export receives the latest compacted snapshot plus any deltas after it. A client requesting "what changed since sequence X" gets the relevant deltas directly from the event log. Both operations are index lookups + blob fetches — no server-side parsing or merging required.

**Write path:** A client pushes a delta. The server assigns a sequence number, stores the delta as an immutable blob, and updates the metadata index. No rewriting of existing data.

#### 7.2.2 Partition-Aligned Object Storage

Time-based memory partitions (§4.1.1) align directly with the cloud storage model:

- **Sealed partitions** are written once to object storage and never touched again under normal operation. They can be served with long cache lifetimes and replicated cheaply across regions. The sole exception is a hard purge for compliance (§3.1.11), which replaces the partition rather than modifying it.
- **The current (unsealed) partition** is the only object that changes. On each compaction, the service rewrites the current partition by applying recent deltas. Once a time period ends, the partition is sealed and becomes immutable.
- **Full snapshot cost** is dominated by the initial upload. Subsequent syncs only transfer the current partition's changes (via deltas), making the marginal cost proportional to activity, not total agent size.

This means a 3-year-old agent with 50 MB of total memory doesn't cost more to sync incrementally than a 1-month-old agent — the sealed partitions are already in storage and never re-transferred.

#### 7.2.3 Sequence-Based Delta Retrieval

The sync service indexes deltas by agent ID and sequence number. The primary query patterns are:

- **Push delta:** `POST /agents/{id}/deltas` → server assigns sequence, stores blob, returns new cursor.
- **Pull deltas:** `GET /agents/{id}/deltas?since={sequence}` → returns ordered list of delta blobs after the given sequence.
- **Pull snapshot:** `GET /agents/{id}/snapshot` → returns the latest compacted snapshot (full `.alf` file).
- **Pull snapshot + deltas:** `GET /agents/{id}/snapshot?include_deltas=true` → returns the latest snapshot plus any deltas not yet compacted into it. This is the "full restore" path.

Detailed API design (authentication, pagination, rate limiting, webhook notifications for sync events) is deferred to the sync protocol design document.

#### 7.2.4 Cost Model Implications

The format and protocol are designed so that the primary cost drivers for the sync service are:

| Cost Driver | Scales With | Mitigation |
|-------------|-------------|------------|
| Object storage (bulk data) | Total agent size | Sealed partitions written once, never rewritten. Large artifacts excluded by default (§3.1.9). |
| Metadata index (database) | Number of deltas per agent | Compaction collapses deltas into snapshots. Index rows are small (ID, sequence, blob pointer). |
| Bandwidth (transfer) | Sync frequency × delta size | Deltas are small (< 100 KB typical). Sealed partitions are cached and never re-transferred. |
| Compute (compaction) | Sync frequency × agent count | Background batch job. JSONL append — no parsing or merging required. |

The service should be viable at low cost per agent, enabling a free tier for personal use and paid tiers for high-frequency sync or multi-device scenarios.

### 7.3 Conflict Resolution (Initial Release)

Initial releases support **single-writer** only (one runtime syncing to one agent-life store). Conflict resolution is last-write-wins by sequence number.

### 7.4 Conflict Resolution (Future: Multi-Writer)

The schema includes fields to support future multi-writer scenarios:

- `source.runtime` and `source.session_id` on every record enable per-source tracking.
- `supersedes` chains enable merge-and-audit.
- Soft deletes enable tombstone-based conflict resolution.
- Sequence numbers assigned by the server provide a total ordering across writers.
- The specific merge strategy for concurrent multi-framework operation is deferred to a future specification.

---

## 8. Non-Functional Requirements

### 8.1 Size and Performance

- A typical agent with 6 months of daily use should produce a full snapshot under 50 MB (compressed). Large workspace artifacts are excluded by default (§3.1.9); only small portable artifacts (Tier 2) are included.
- Incremental deltas after a single session should typically be under 100 KB.
- The sync service must accept delta uploads within 5 seconds for typical deltas.
- Memory retrieval from the sync service (future cloud runtime use case) should target <500ms for semantic search over 10K records.

### 8.2 Compatibility and Schema Evolution

- The format MUST use semantic versioning (`alf_version` in manifest).
- Readers MUST be forward-compatible within a major version: ignore unknown fields, process known fields.
- Breaking changes (removed or retyped fields) require a major version bump.

**Unknown enum handling:** When a reader encounters an unrecognized value for an enum field, it MUST NOT reject the record. Instead, it applies a safe default interpretation:

| Enum Field | Safe Default for Unknown Values |
|------------|-------------------------------|
| `memory_type` | Treat as `semantic` (general fact). This is the safest interpretation — the memory is preserved and queryable, just not categorized into a specific cognitive type. |
| `principal_type` | Treat as `human` (the common case). |
| `credential_type` | Treat as `custom` (opaque, no special handling). |
| `status` | Treat as `active` (err on the side of visibility). |
| `source` (embedding) | Treat as `runtime`. |

This rule ensures that a snapshot produced by a newer version of the format can still be consumed by an older reader with graceful degradation rather than failure. The unrecognized value MUST be preserved on round-trip — the reader stores what it received, even if it didn't understand it, so that a future export doesn't lose the original type information.

### 8.3 Security

- No plaintext credentials in any file within the ALF archive.
- The sync service operates on a zero-knowledge basis for Layer 4 (credentials).
- Snapshots SHOULD support optional full-archive encryption (encrypt the entire ZIP) for at-rest protection on untrusted storage.
- The format MUST NOT include information that would allow credential recovery without the user's key.
- For detailed data privacy requirements including transport encryption, at-rest encryption tiers, and defense-in-depth measures, see §8.5.

### 8.4 Licensing

- The format specification is released under the **MIT License**.
- Reference adapter implementations are open source (MIT License).
- The sync service API specification is open. Service implementations may use any license.

### 8.5 Data Privacy and Encryption

Agents have access to a user's most intimate data — financial records, health information, private messages, personal preferences. The encryption model must reflect this sensitivity.

#### 8.5.1 Threat Model

The encryption architecture addresses three distinct threat categories:

| Threat | Example | Mitigation |
|--------|---------|------------|
| **Wire interception** | Man-in-the-middle on client-to-service traffic | Transport encryption (§8.5.2) |
| **Storage-layer breach** | Stolen disk, misconfigured bucket, compromised backup, leaked database replica | At-rest encryption with per-tenant keys (§8.5.3) |
| **Compromised service process** | Attacker gains access to a running service instance that holds decrypted data in memory | Access controls, audit logging, data residency minimization (§8.5.4) |

**What at-rest encryption does and does not do:** At-rest encryption protects against infrastructure-level threats — an attacker who gains access to the storage medium but not to the running service. It does not protect against a compromised service process, which necessarily holds decrypted data in memory while indexing, searching, and compacting. The spec names this boundary explicitly rather than implying encryption alone is sufficient.

#### 8.5.2 Transport Encryption

- All client-to-service communication MUST use **TLS 1.3** (or higher). Connections using TLS 1.2 or below MUST be rejected.
- All service-internal communication (e.g., between API servers and object storage, between API servers and the metadata index) SHOULD use **mutual TLS (mTLS)** with short-lived certificates.
- Delta and snapshot payloads are not additionally encrypted in transit — TLS provides confidentiality and integrity on the wire.

#### 8.5.3 At-Rest Encryption Tiers

The service defines three encryption tiers. The tier applies to all agent data within a tenant unless per-agent key configuration is enabled (see §8.5.3.2).

**Tier 1 — Per-Tenant Service-Managed Keys (Standard)**

Every tenant (user account) receives a unique encryption key, generated and managed by the service via a KMS (AWS KMS, GCP Cloud KMS, HashiCorp Vault, or equivalent). All agent data for that tenant — memory partitions, identity, principals, delta blobs, compacted snapshots — is encrypted with that tenant's key before being written to object storage or the metadata index.

Properties:

- A storage-layer breach (stolen backup, misconfigured bucket, compromised replica) exposes only ciphertext. Without the KMS, the data is unreadable.
- A single key compromise exposes one tenant's data, not all tenants. The blast radius is contained.
- The service decrypts transparently when processing — indexing, searching, compacting, and serving all operate on plaintext in memory.
- Key rotation is managed by the KMS. Old data is re-encrypted on the next compaction cycle — sealed partitions are re-encrypted lazily, since they are read-only and the old key remains valid until rotation completes.

This is the launch tier and the right default for most users.

**Tier 2 — Customer-Managed Keys (BYOK)**

The tenant provides and controls their own encryption key via their own KMS. The service holds a key reference (ARN, resource ID) rather than the key material itself. The customer controls rotation and can revoke the service's access to the key at any time.

Properties:

- Same protection as Tier 1, plus the customer retains ultimate control over data access.
- Revoking the key renders all of that tenant's data unreadable by the service — effectively a cryptographic delete.
- Required for enterprise customers in regulated industries (healthcare, finance) who must demonstrate key custody in compliance audits.
- Higher operational complexity for the customer — they manage key availability, rotation, and access policies.

This tier is a future addition. The architecture MUST NOT preclude it — the encryption layer should be key-source-agnostic from day one.

**Tier 3 — Layer 4 Zero-Knowledge (Credentials Only)**

Credentials (Layer 4) remain fully client-side encrypted with user-controlled keys, as specified in §3.4 and §8.3. The service stores only ciphertext and never possesses the decryption key. This is unchanged from the existing spec. It applies regardless of which at-rest tier is active for Layers 1–3.

**Why not zero-knowledge for all layers?** The service must read Layers 1–3 to provide its core functionality: re-indexing memory for search, compacting deltas into snapshots, and serving query results. Zero-knowledge encryption for these layers would reduce the service to dumb blob storage with no ability to index or search — the user would have to download, decrypt, and search locally, eliminating the value of a sync service.

#### 8.5.3.1 Why Not a Service-Wide Key?

A single key encrypting all tenant data is the "we encrypted the S3 bucket" checkbox. It satisfies basic compliance requirements but doesn't meaningfully contain breaches — a single key compromise exposes every user's data. For data as sensitive as agent memory (which may contain health records, financial details, and private messages), this is not acceptable.

#### 8.5.3.2 Key Lookup Architecture

Keys are looked up by a **scope identifier** — by default, the tenant ID. This means per-agent keys are a deployment-time configuration choice, not a schema change: if the scope identifier is set to `agent_id` instead of `tenant_id`, each agent within a tenant gets its own key. This adds value for power users running multiple agents with different sensitivity levels (e.g., a personal assistant with health data vs. a coding assistant with no sensitive data). Per-agent keys are not the default because for most users (one tenant, one agent) they are identical to per-tenant keys, and the additional key management complexity is not justified.

#### 8.5.4 Defense in Depth Beyond Encryption

At-rest encryption is necessary but not sufficient. The sync service MUST also implement:

- **Access controls:** Tenant data is accessible only to authenticated requests from that tenant. Service-internal access (compaction jobs, indexing workers) uses scoped, short-lived credentials with least-privilege policies.
- **Audit logging:** All data access — reads, writes, key usage, admin operations — is logged with tenant ID, timestamp, and operation type. Logs are immutable and retained for a configurable period.
- **Data residency minimization:** Decrypted data exists in memory only for the duration of a processing operation (index, compact, serve). It is not written to disk in plaintext, not cached beyond the operation, and not logged in cleartext.
- **Tenant isolation:** Multi-tenant deployments MUST enforce logical isolation at the storage and query layers. A query for tenant A's data MUST NOT be able to return tenant B's records, even in the event of a bug. The metadata index enforces tenant-scoped queries at the database level (row-level security or equivalent), not just at the application level.
- **Hard deletion:** The service MUST support both full agent deletion and surgical record purge to comply with GDPR, CCPA, and equivalent data privacy regulations. Soft deletes are insufficient for compliance — physically removing data from storage is required. See §3.1.11 for the purge operation and audit trail requirements.

---

## 9. Coverage Matrix — No Information Loss

This matrix demonstrates that the neutral format captures a superset of data from both target runtimes.

| Data Item | OpenClaw Source | ZeroClaw Source | ALF Location |
|-----------|----------------|-----------------|--------------|
| Agent name / role | IDENTITY.md | AIEOS `identity.names` | `identity.structured.names` + `identity.prose.identity_profile` |
| Personality / tone | SOUL.md | AIEOS `psychology`, `linguistics` | `identity.structured.psychology` + `identity.prose.soul` |
| Operating rules | AGENTS.md | Workspace AGENTS.md | `identity.prose.operating_instructions` |
| User preferences | USER.md | Workspace USER.md / memory | Primary human principal profile + `namespace: "principal_context:{id}"` memories |
| Agent-to-agent relationships | N/A | N/A | `principals.json` with `principal_type: "agent"` (§3.3.1) |
| Sub-agent roster | (in AGENTS.md or memory) | (in memory) | `identity.structured.sub_agents` (§3.2.4) |
| Model used | (not stored) | config.toml `default_model` | `manifest.runtime_hints` (§3.2.5) + sub-agent `model_hints` |
| Long-term memory | MEMORY.md | SQLite `core` category | Memory records with `memory_type: "summary"` |
| Daily logs | memory/YYYY-MM-DD.md | SQLite `daily` category / Markdown daily files | Memory records with `category: "daily_log"` |
| Conversation memories | (in daily logs) | SQLite `conversation` category | Memory records with `memory_type: "episodic"` |
| Startup checklist | BOOT.md | (not standard) | `identity.prose.custom_blocks.boot_checklist` |
| Heartbeat checklist | HEARTBEAT.md | Workspace HEARTBEAT.md | `identity.prose.custom_blocks.heartbeat_checklist` |
| Tools guidance | TOOLS.md | Workspace TOOLS.md | `identity.prose.custom_blocks.tools_guidance` |
| Capabilities | (in AGENTS.md) | AIEOS `capabilities` | `identity.structured.capabilities` |
| API keys | ~/.openclaw/ env vars | config.toml `[secrets]` | `credentials.json` (encrypted) |
| OAuth tokens | (not standard) | `.zeroclaw/state/auth_profiles.json` | `credentials.json` (encrypted) |
| Vector search weights | N/A (hardcoded 70/30) | config.toml `vector_weight`, `keyword_weight` | `manifest.json` adapter hints |
| AIEOS neural matrix | N/A | AIEOS `psychology.neural_matrix` | `identity.structured.psychology.neural_matrix` |
| AIEOS linguistics | N/A | AIEOS `linguistics` | `identity.structured.linguistics` |
| Raw native files | All workspace .md files | config.toml, memories.db | `raw/openclaw/`, `raw/zeroclaw/` |
| Workspace artifacts | Workspace non-runtime files | Workspace non-runtime files | `artifacts/` (Tier 2, included) + `attachments.json` (Tier 3, reference-only) (§3.1.9) |

---

## 10. Testing Strategy

### 10.1 Schema Validation Tests

- JSON Schema definitions for every layer, validated against the spec examples.
- Reject invalid records (missing required fields, wrong types, invalid enums).
- Accept records with unknown extension fields (forward compatibility).

### 10.2 Round-Trip Tests

For each supported runtime:

1. Set up a runtime with a rich agent state (identity, 100+ memories, 5+ credentials, user context).
2. Export to ALF.
3. Import back to the same runtime type.
4. Assert byte-for-byte equality of raw source files.
5. Assert semantic equality of structured fields.
6. Assert `source.identity_version` is preserved on all memory records that had it.
7. Assert `related_records` arrays are preserved on all memory records that had them, with relation types and target IDs intact.

### 10.2.1 Memory-to-Identity Lineage Tests

1. Create an agent with identity version 1. Generate several memories. Export. Verify all memory records have `source.identity_version: 1`.
2. Update the identity (producing version 2). Generate more memories. Export. Verify new memories have `source.identity_version: 2` while old memories retain version 1.
3. Filter memories by `source.identity_version`. Verify the filter produces correct, disjoint sets.
4. Import a snapshot containing memories with mixed identity versions. Verify all version values survive the round-trip.
5. Import a snapshot containing legacy memories without `source.identity_version` (migrated data). Verify these import without error and the field remains absent (not backfilled with a guess).

### 10.2.2 Memory Relational Link Tests

1. Create memories with `related_records` links using well-known relation types (`caused_by`, `contradicts`, `elaborates_on`). Export and re-import. Verify all links survive the round-trip with correct target IDs and relation types.
2. Create memories with custom (runtime-defined) relation types. Verify these are preserved through round-trip without modification.
3. Create a bidirectional relationship (record A `contradicts` record B, record B `contradicts` record A). Verify both directions are preserved independently.
4. Surgically purge a memory that is the target of a relational link from another record. Verify the linking record imports without error (dangling reference handled gracefully).
5. Import a snapshot with relational links into a runtime that does not support graph relationships. Verify the import succeeds and the links are preserved in the exported ALF for future use (even if the runtime ignores them).

### 10.3 Cross-Runtime Migration Tests

1. Export from Runtime A (e.g., OpenClaw).
2. Import to Runtime B (e.g., ZeroClaw).
3. Assert no data loss: every memory is present, identity is recognizable, credentials are functional.
4. Export from Runtime B.
5. Import back to Runtime A.
6. Assert equivalence with the original (allowing for representation differences in prose↔structured translation).

### 10.4 Incremental Sync Tests

1. Take a full snapshot at sequence S0.
2. Make changes (add memories, update identity, rotate a credential).
3. Export delta since S0. Verify the delta manifest contains `base_sequence: S0` and a `new_sequence` > S0.
4. Apply delta to a copy of the S0 snapshot.
5. Assert the result equals a full snapshot taken at the new sequence.
6. Verify sequence numbers are monotonically increasing across consecutive deltas.
7. Verify a client requesting deltas "since S0" receives exactly the changes made in step 2.

### 10.5 Credential Security Tests

- Verify no plaintext credentials appear anywhere in the ALF archive (binary scan).
- Verify credentials decrypt correctly with the correct key.
- Verify credentials fail to decrypt with an incorrect key.
- Verify the sync service never receives or logs the decryption key.

### 10.5.1 Credential-to-Capability Linkage Tests

1. **Bidirectional linkage round-trip:** Create a credential with `capabilities_granted: ["github_repo_reader"]` and a capability with `credential_ids: ["credential-uuid"]`. Export and re-import. Verify both directions of the linkage survive.
2. **Capability lookup from credential:** Given a credential with `capabilities_granted`, verify the listed capability names resolve to entries in `identity.structured.capabilities`.
3. **Credential lookup from capability:** Given a capability with `credential_ids`, verify the listed UUIDs resolve to entries in `credentials.json`.
4. **Missing credential on import:** Import a snapshot where a capability references a `credential_id` that doesn't exist in `credentials.json` (e.g., credential was purged). Verify the import succeeds with a warning, not an error.
5. **General-purpose credential:** Create a credential with no `capabilities_granted` (e.g., a general-purpose OpenAI key). Verify it exports and imports cleanly with the field absent.
6. **Intrinsic capability without credentials:** Verify capabilities with `portability: "intrinsic"` and no `credential_ids` export cleanly.

### 10.6 Workspace Artifact Tests

1. **Tier classification:** Export from a workspace containing a mix of runtime files (SOUL.md), small artifacts (4 KB CSV, 20 KB script), and large artifacts (500 KB image, 2 MB PDF). Verify: Tier 1 files go to `raw/`, Tier 2 files go to `artifacts/` with `archive_path` set, Tier 3 files are reference-only with `archive_path: null`.
2. **Size threshold:** Export with the default 100 KB threshold. Verify files under 100 KB are in `artifacts/`, files over 100 KB are reference-only. Re-export with a custom threshold of 10 KB. Verify the classification changes accordingly and `artifact_size_threshold` in `attachments.json` reflects the configured value.
3. **Tier 2 round-trip:** Export, then import to a new workspace. Verify all Tier 2 artifacts from `artifacts/` are extracted and placed in the target workspace.
4. **Tier 3 reporting:** On import, verify the adapter reports Tier 3 (reference-only) artifacts to the user with filenames and sizes.
5. **Hash integrity:** Verify all entries in `attachments.json` have correct SHA-256 hashes matching the source files. For Tier 2, also verify the hash matches the file in `artifacts/`.
6. **Referenced_by linkage:** Verify `referenced_by` correctly connects artifacts to the memory records that reference them.
7. **Mixed media types:** Verify the index correctly catalogs text artifacts (CSV, Python, JSON) alongside binary artifacts (PNG, PDF) with correct MIME types.
8. **Memory import with unresolved artifacts:** Verify memory records referencing Tier 3 (absent) artifacts import successfully without errors.
9. **Manifest counts:** Verify the manifest's `included_count` and `referenced_count` match the actual Tier 2 and Tier 3 counts in `attachments.json`.

### 10.7 Principal and Relationship Tests

1. Export from a runtime with a single human user. Verify `principals.json` contains one entry with `principal_type: "human"`.
2. Import a single-principal snapshot into a framework that has no multi-principal concept. Verify the primary human principal's profile maps correctly to the native user context (e.g., OpenClaw's `USER.md`).
3. Construct a snapshot with multiple principals (one human, one agent-type). Verify each has an independent profile, independent versioning, and separate memory namespaces.
4. Verify agent-type principals have a valid `agent_id` or explicit null.

### 10.8 Sub-Agent Roster Tests

1. Export from a managing agent with sub-agents. Verify `identity.structured.sub_agents` contains entries with capabilities, model hints, and routing hints.
2. Import a managing agent snapshot to a new host. Verify the roster is preserved and accessible to the runtime for future sub-agent recreation.
3. Verify sub-agents with `agent_id: null` (ephemeral) import cleanly alongside sub-agents with valid `agent_id` references.
4. Verify model hints (`primary_model`, `last_model`) are preserved through round-trip export/import.

### 10.9 Capability Portability Tests

1. **Portability annotation round-trip:** Export an agent with both `intrinsic` and `host_dependent` capabilities. Verify `portability` and `host_requirements` fields are preserved through export/import.
2. **Host-dependent reporting on import:** Import a snapshot containing a `host_dependent` capability (e.g., `docker_management` requiring Docker CLI) into an environment without Docker. Verify the adapter reports the unavailable capability to the user.
3. **Capability preservation:** After importing to a host that lacks Docker, verify the `docker_management` capability is still present in `identity.structured.capabilities` — the agent retains the knowledge even though the capability is currently unavailable.
4. **Default portability:** Export an agent with capabilities that have no explicit `portability` field. Verify the importer treats them as `intrinsic` (the default).
5. **Sub-agent capability portability:** Verify that capabilities in the sub-agent roster also support `portability` and `host_requirements` and are preserved through round-trip.
6. **Re-migration restoration:** Import an agent (with an unavailable host-dependent capability) to a second host that does have Docker. Verify the capability is available again without any identity changes — the agent didn't "forget" it.

### 10.10 Scale Tests

- Generate a synthetic agent with 50K memory records.
- Verify full snapshot size stays under target (50 MB compressed).
- Verify incremental delta generation completes in under 10 seconds.
- Verify memory search over 50K records meets latency targets.

### 10.11 Partitioning Tests

1. Export an agent with memories spanning multiple quarters. Verify the snapshot contains one JSONL file per quarter under `memory/partitions/`.
2. Verify all partitions except the current one have `sealed: true` in the manifest.
3. Verify each record's `temporal.created_at` falls within its partition's `from`/`to` range.
4. Re-export the same agent after adding new memories. Verify sealed partition files are byte-identical to the previous export — only the current partition changed.
5. Verify that a status change (supersession or deletion) of a record in a sealed partition creates a new record in the current partition (sealed partitions are never rewritten).
6. Generate a high-activity synthetic agent. Verify the adapter can switch to monthly partitioning and that the manifest correctly reflects monthly `from`/`to` ranges.

### 10.12 Storage and Transport Tests

1. Simulate the event-sourcing write path: push three consecutive deltas, verify each receives a monotonically increasing sequence number.
2. Simulate the delta retrieval path: request deltas since a given sequence, verify only deltas after that sequence are returned, in order.
3. Simulate compaction: given a snapshot at S0 and deltas S1–S5, compact into a new snapshot. Verify the compacted snapshot equals a full export at S5.
4. Verify sealed partitions can be served from a cache (blob hash matches across repeated retrievals).
5. Verify full restore path: snapshot + uncompacted deltas produces the same result as a fresh full export.

### 10.13 Data Privacy and Encryption Tests

1. **Per-tenant key isolation:** Create two tenants. Store data for both. Verify tenant A's key cannot decrypt tenant B's data at the storage layer.
2. **Encryption at rest:** Write a memory partition to object storage. Read the raw bytes from the storage backend (bypassing the service). Verify the content is ciphertext, not readable JSONL.
3. **Key rotation:** Rotate a tenant's key. Trigger a compaction cycle. Verify all data (including sealed partitions) is re-encrypted with the new key and the old key no longer decrypts it.
4. **Per-agent key scope:** Configure per-agent key lookup. Create two agents under one tenant. Verify each agent's data is encrypted with a different key.
5. **BYOK key revocation:** Configure a customer-managed key. Revoke the service's access to the key. Verify all reads for that tenant fail with an appropriate error (cryptographic delete).
6. **Transport encryption:** Attempt a connection without TLS (or with TLS < 1.3). Verify the connection is rejected.
7. **Data residency minimization:** After a search or compaction operation completes, verify no decrypted data remains on disk (temp files, swap, logs). Decrypted data should only exist in memory for the duration of the operation.
8. **Tenant isolation at query layer:** Issue a metadata index query scoped to tenant A that attempts to reference tenant B's records (e.g., via a crafted agent ID). Verify the query returns no results from tenant B.

### 10.14 Hard Deletion Tests

1. **Full agent deletion:** Create an agent with memories across multiple sealed partitions, identity, principals, and credentials. Issue a full agent deletion. Verify all partition blobs, identity, principals, credentials, deltas, metadata index entries, and compacted snapshots are destroyed. Verify an audit record exists with `scope: "full_agent"` and does not contain any purged content.
2. **Surgical record purge:** Create an agent with a sealed partition containing 100 records. Purge 3 specific records by ID. Verify: (a) a new partition exists with 97 records, (b) the old partition blob is destroyed from object storage, (c) the manifest references the new partition with updated record count and checksum, (d) the purged record IDs do not appear anywhere in the new partition, (e) an audit record exists listing the purged record IDs.
3. **Purge from multiple partitions:** Purge records spanning two sealed partitions. Verify both partitions are replaced independently and both old blobs are destroyed.
4. **Purge from current (unsealed) partition:** Purge a record from the current partition. Verify the record is removed and the partition is rewritten (same operation, just on the unsealed partition).
5. **Delta cleanup:** After a surgical purge, verify any delta blobs containing the purged records are also destroyed or rewritten to exclude them.
6. **Search index cleanup:** After a purge, query the search/retrieval index for the purged record's content. Verify no results are returned.
7. **Cache invalidation:** After a partition replacement, verify that clients caching the old partition hash receive the new partition on next sync (hash mismatch forces re-download).
8. **Local `.alf` purge:** Use the adapter `purge` command on a local `.alf` file. Verify the rewritten archive excludes the target records and the file is a valid ALF archive.
9. **Audit record integrity:** Verify the audit record for a purge contains record IDs, partition paths, reason, and timestamps — but does NOT contain any of the purged content or metadata that could reconstruct it.

---

## 11. Design Decision Log

This section documents key design decisions with their rationale. Each entry records a design question that was considered during specification development and the resolution adopted. Detailed technical context lives in the referenced sections.

1. **Embedding portability.** The format stores embeddings as-is from the source runtime, tagged by model (see §3.1.6). No neutral embedding model is mandated. Re-embedding happens at the edges (adapters, sync service) only when needed. This avoids vendor lock-in, eliminates bulk re-embedding costs, and allows future on-the-fly conversion when a client requests an import with a different embedding spec.

2. **Workspace artifact handling.** Three-tier model (§3.1.9). Tier 1: managed agent state (SOUL.md, config files) goes in `raw/`. Tier 2: small portable artifacts under 100 KB (scripts, small data files) are included in `artifacts/` with `archive_path` set. Tier 3: large artifacts are cataloged by reference only (`archive_path: null`, `remote_ref: null` with placeholder for future online storage). The classification is based on managed-state-vs-artifact and size, not binary-vs-text — a 10K-line CSV and a large PNG are both Tier 3. The `referenced_by` linkage connects artifacts to memory records. On import, Tier 2 artifacts are extracted to the workspace; Tier 3 artifacts are reported to the user.

3. **Multi-agent containers.** One agent per `.alf` file. Inter-agent relationships are modeled as references, not containment. A managing agent records its sub-agents via a structured roster in the identity layer (§3.2.4) with capabilities, model hints, and routing guidance. A sub-agent records its managing agents as principals (§3.3.1) with `principal_type: "agent"`. On restore, sub-agents won't exist on the new host — the roster gives the managing agent the information it needs to recreate them on request. Each sub-agent's own state, if persistent, lives in its own `.alf` file referenced by `agent_id`.

4. **Schema evolution.** Unknown enum values are handled via safe defaults: unknown `memory_type` → `semantic`, unknown `principal_type` → `human`, unknown `credential_type` → `custom`, unknown `status` → `active` (see §8.2). Unrecognized values MUST be preserved on round-trip so future exports don't lose type information. Unknown fields are ignored by readers but retained in storage. Breaking changes require a major version bump.

5. **AIEOS alignment.** Compatible superset with opinionated field promotion (see §3.2.6). Fields that map to real agent behavior (names, OCEAN traits, neural matrix, linguistics, capabilities, motivations) are promoted to first-class structured fields. Fields rooted in fictional character design (MBTI, D&D moral alignment, physicality/somatotype, bio metadata) are preserved for lossless round-trip via `aieos_extensions` but not promoted. This provides full ZeroClaw/AIEOS round-trip fidelity without elevating design choices that lack broad applicability as schema primitives.

6. **Storage, transport, and partitioning.** Memory is partitioned by time period (quarterly by default) with sealed/unsealed semantics (§4.1.1). Sealed partitions are immutable and cache-friendly. The sync protocol uses monotonic sequence numbers as the primary cursor (§4.3.1), not timestamps — avoiding clock skew and boundary ambiguity. The cloud storage model follows an event-sourcing pattern (§7.2): deltas as append-only events in object storage, snapshots as periodic compacted projections, metadata index in a lightweight database. This keeps costs proportional to activity rather than total agent size, and ensures all operations (push delta, pull deltas, full restore) are index lookups + blob fetches with no server-side parsing.

7. **Memory-to-identity lineage.** Each memory record's `source` object includes an optional `identity_version` integer (§3.1.10) linking what the agent did to who the agent was at the time. This makes lineage explicit and directly queryable rather than requiring timestamp correlation between the identity version history and memory creation times. The field reuses the identity layer's existing version counter (§3.2.3) — no new versioning mechanism needed. Adapters SHOULD stamp the current version on export. Records without the field (migrated data) have unknown lineage, preserving backward compatibility.

8. **Data privacy and at-rest encryption.** Three-tier encryption model (§8.5.3). Tier 1 (launch default): per-tenant service-managed keys via KMS — a storage-layer breach exposes only ciphertext, and a single key compromise affects only one tenant. Tier 2 (future): customer-managed keys (BYOK) for regulated industries — customer retains key custody and can revoke service access. Tier 3: Layer 4 credentials remain zero-knowledge client-side encrypted regardless of tier. Service-wide keys rejected as insufficient for agent-sensitivity data. Per-agent keys supported as a deployment-time configuration via scope identifier (§8.5.3.2) but not the default. Transport: TLS 1.3 required, mTLS for service-internal. Defense-in-depth measures (access controls, audit logging, data residency minimization, tenant isolation) specified in §8.5.4 because at-rest encryption alone doesn't protect against a compromised service process.

9. **GDPR/CCPA hard deletion vs. immutable partitions.** Two deletion scopes (§3.1.11). Full agent deletion (the common compliance request) destroys all objects wholesale — no partition rewriting needed. Surgical record purge replaces affected sealed partitions rather than modifying them in place, preserving the mental model that any given partition is immutable. The old partition is destroyed, the replacement has a different hash so caches invalidate naturally. Both scopes produce audit records proving deletion without revealing purged content. Soft deletes (§3.1.8) remain the default for normal operation; hard purge is an exceptional compliance operation. Adapters must support a local `purge` command for self-hosted deployments.

10. **Intrinsic vs. host-dependent capabilities.** Capabilities remain in the identity layer (correct for task routing), but each carries a `portability` annotation: `intrinsic` (LLM-native, always available) or `host_dependent` (requires specific environment) with an optional `host_requirements` description (§3.2.6). No new "Environment" or "Host Context" layer — that would conflate portable agent state with ephemeral deployment configuration. On migration, the identity doesn't change; the importing adapter checks host-dependent capabilities against the target environment and reports unavailable ones. The agent retains knowledge of all capabilities for future environments that support them.

11. **Memory graph / relational linkages.** Optional `related_records` array on memory records (§3.1.12) with typed, one-directional links. Relation types are free-form strings with well-known conventions (`caused_by`, `contradicts`, `elaborates_on`, `derived_from`, `caused`, `related_to`). `supersedes` remains a separate top-level field because it has lifecycle semantics (changes record status, used by compaction). Relational links are informational/navigational — they don't affect record lifecycle. Runtimes that don't use graph relationships ignore the field. Dangling references (from purged targets) are tolerated.

12. **Execution state / workflow checkpoints.** Explicitly out of scope (§1). ALF stores knowledge state, identity state, relationship state, and credentials — the durable, portable layers. Active execution state (in-flight tool calls, pending workflow steps, loop counters, LangGraph checkpoints, call stacks) is excluded because it is inherently runtime-specific, ephemeral, and not part of the agent's durable identity or knowledge. Migrating mid-task aborts the in-progress task; the agent retains all accumulated knowledge but unfinished workflows are lost. This is the correct tradeoff — execution state is cheap to recreate (re-run the task), knowledge and identity are expensive to recreate (months of accumulated context).

13. **Credential-to-capability linkage.** Bidirectional optional linkage (§3.4.2, §3.2.6). Credentials carry `capabilities_granted` (names of capabilities they enable). Capabilities carry `credential_ids` (UUIDs of credentials they require). Both are informational and optional — many credentials are general-purpose (one API key powering multiple capabilities) and many capabilities need no credentials (intrinsic LLM abilities). The linkage helps importers wire up auth automatically and report missing credentials as warnings.

## 12. Roadmap

1. **Publish JSON Schema definitions** — formalize the schemas in §3 as JSON Schema files (see `schemas/` directory).
2. **Build reference adapters** — OpenClaw adapter first, then ZeroClaw.
3. **Publish sync service API specification** — detailed design covering the event-sourcing storage model (§7.2), REST API (endpoints, auth, pagination, rate limiting), compaction scheduling, and cost projections.
4. **Implement round-trip test suite** — the primary quality gate before any adapter release.
5. **Community feedback** — solicit input from agent framework developers on coverage gaps and interoperability concerns.
