# Agent Life Format (ALF)

**An open, portable data format for AI agent state.**

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Spec Version](https://img.shields.io/badge/spec-1.0.0--rc.1-orange.svg)](SPECIFICATION.md)

---

ALF is a serialization format for the complete durable state of an AI agent — memories, identity, relationships, and credentials. It enables:

- **Backup** — snapshot your agent's accumulated knowledge and personality to a single portable file.
- **Migration** — move an agent between runtimes (e.g., OpenClaw → ZeroClaw) without losing context.
- **Incremental sync** — push lightweight deltas to cloud storage after each session, not full snapshots.

ALF is runtime-neutral by design. It captures the union of data models used by today's agent frameworks so that exporting from any supported runtime preserves all information. The format is decoupled from any specific LLM provider, embedding model, or execution engine.

A html version of the specification is hosted at [agent-life.ai](https://agent-life.ai)

## Quick Look

An `.alf` file is a standard ZIP archive:

```
nova-agent.alf
├── manifest.json          # Format version, agent metadata, sync cursor
├── identity.json          # Persona, capabilities, behavioral config
├── principals.json        # Users and agent relationships
├── credentials.json       # Encrypted credentials (zero-knowledge)
├── memory/
│   └── partitions/
│       ├── 2025-Q3.jsonl  # Sealed partition (immutable)
│       ├── 2025-Q4.jsonl  # Sealed partition (immutable)
│       └── 2026-Q1.jsonl  # Current partition
├── attachments.json       # Workspace artifact index
├── artifacts/             # Small portable files (<100KB)
└── raw/                   # Lossless runtime originals
    └── openclaw/
        ├── SOUL.md
        └── MEMORY.md
```

Each memory record is a self-contained JSON line:

```json
{
  "id": "01902f4a-5b6c-7d8e-9f0a-1b2c3d4e5f6a",
  "content": "User prefers concise answers with minimal formatting.",
  "memory_type": "preference",
  "source": {
    "runtime": "openclaw",
    "extraction_method": "agent_written",
    "identity_version": 3
  },
  "temporal": {
    "created_at": "2026-02-10T14:30:00Z",
    "observed_at": "2026-02-10T14:25:00Z"
  },
  "status": "active",
  "namespace": "default"
}
```

## Data Layers

ALF organizes agent state into four layers:

| Layer | Contents | Sensitivity |
|-------|----------|-------------|
| **1. Memory** | Facts, episodes, procedures, preferences, summaries — with embeddings, entities, temporal metadata, and relational links | Standard |
| **2. Identity** | Persona (structured fields + prose blocks), capabilities, sub-agent roster, AIEOS-compatible extensions | Standard |
| **3. Principals** | User profiles, agent-to-agent relationships, communication preferences | Standard |
| **4. Credentials** | API keys, OAuth tokens, SSH keys — zero-knowledge encrypted, never readable by the sync service | Critical |

## Key Design Choices

- **Dual identity representation** — both structured fields (machine-readable) and prose blocks (LLM-interpreted). Each runtime uses whichever it supports.
- **Store-and-tag embeddings** — vectors are preserved as-is from the source runtime, tagged by model. No mandated embedding model. Re-embedding happens at the edges.
- **Time-based partitioning** — memory is partitioned quarterly. Sealed partitions are immutable and cache-friendly. Only the current partition changes.
- **Sequence-based sync** — monotonic sequence numbers as the sync cursor, not timestamps. No clock skew, no boundary ambiguity.
- **Three-tier artifacts** — managed agent state in `raw/`, small portable files in `artifacts/`, large files as reference-only. Classification by purpose and size, not binary vs. text.
- **Forward compatibility** — unknown fields are preserved on round-trip. Unknown enum values get safe defaults. No data is silently dropped.

## Repository Structure

```
agent-life-data-format/
├── README.md              # This file
├── LICENSE                # MIT
├── SPECIFICATION.md       # The full format specification
├── CONTRIBUTING.md        # How to propose changes
└── schemas/
    ├── README.md          # Schema overview and usage
    ├── manifest.schema.json
    ├── memory-record.schema.json
    ├── identity.schema.json
    ├── principals.schema.json
    ├── credentials.schema.json
    ├── attachments.schema.json
    └── delta-manifest.schema.json
```

## Specification

The complete specification is in [SPECIFICATION.md](SPECIFICATION.md). It covers:

- §1 — Purpose, scope, and explicit exclusions (no execution state)
- §2 — Background on existing runtime data models
- §3 — Data layer specification (memory, identity, principals, credentials)
- §4 — Container format (`.alf` and `.alf-delta` archives)
- §5 — Design decisions and rationale
- §6 — Adapter interface requirements (export, import, delta, purge)
- §7 — Sync protocol requirements (event-sourcing, partitioning, conflict resolution)
- §8 — Non-functional requirements (size, compatibility, security, privacy, encryption)
- §9 — Coverage matrix (no-information-loss guarantee)
- §10 — Testing strategy
- §11 — Design decision log

## JSON Schemas

Machine-readable schemas for all ALF data structures are in the [`schemas/`](schemas/) directory. They use [JSON Schema Draft 2020-12](https://json-schema.org/draft/2020-12/schema) and are the authoritative definitions for the format.

## Status

This specification is at **Release Candidate** (`1.0.0-rc.1`). The data model is stable. We are soliciting feedback from agent framework developers before finalizing v1.0.0.

**What's next:**
- Reference adapter implementations (OpenClaw, ZeroClaw)
- Sync service API specification (separate repository)
- Round-trip test suite

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for how to propose changes to the specification.

## License

This specification and all associated schemas are released under the [MIT License](LICENSE).
