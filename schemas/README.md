# ALF JSON Schemas

JSON Schema definitions for the Agent Life Format (ALF) v1.0.0.

These schemas formalize the data layer specifications in the [ALF Specification](../SPECIFICATION.md). They use [JSON Schema Draft 2020-12](https://json-schema.org/draft/2020-12/schema).

## Schemas

| Schema | File in `.alf` archive | Description |
|--------|----------------------|-------------|
| [manifest.schema.json](manifest.schema.json) | `manifest.json` | Snapshot manifest — format version, agent metadata, runtime hints, sync cursor, layer inventory with partition details (§4.2) |
| [memory-record.schema.json](memory-record.schema.json) | `memory/partitions/*.jsonl` | Single memory record with source provenance, temporal metadata, entities, and embeddings (§3.1) |
| [identity.schema.json](identity.schema.json) | `identity.json` | Identity and persona — structured fields, prose blocks, sub-agent roster, AIEOS extensions (§3.2) |
| [principals.schema.json](principals.schema.json) | `principals.json` | Principals — human users and managing agents with profiles and preferences (§3.3) |
| [credentials.schema.json](credentials.schema.json) | `credentials.json` | Encrypted credentials — zero-knowledge architecture with encryption metadata (§3.4) |
| [attachments.schema.json](attachments.schema.json) | `attachments.json` | Workspace artifact index — Tier 2 (included in `artifacts/`) and Tier 3 (reference-only) artifacts with hashes, archive paths, and remote_ref placeholders (§3.1.9) |
| [delta-manifest.schema.json](delta-manifest.schema.json) | `manifest.json` (in `.alf-delta`) | Delta bundle manifest — sequence-based sync cursors and change inventory (§4.3) |

## Design Decisions

### Self-Contained Schemas

Each schema is self-contained — shared sub-objects (source provenance, temporal metadata, etc.) are defined in `$defs` within the schema that owns them, not in separate files. This avoids `$ref` resolution headaches for consumers and means each schema can be used independently.

### Enum Handling and Forward Compatibility

Enum fields (e.g., `memory_type`, `status`, `credential_type`) use JSON Schema's `enum` keyword to document known values and enable tooling autocomplete. However, **validators SHOULD be configured in warning mode for enum fields, not rejection mode**.

Per §8.2 of the specification, readers MUST NOT reject records with unknown enum values. Unknown values are handled via safe defaults:

| Enum Field | Safe Default for Unknown Values |
|------------|-------------------------------|
| `memory_type` | `semantic` |
| `principal_type` | `human` |
| `credential_type` | `custom` |
| `status` (memory) | `active` |
| `source` (embedding) | `runtime` |

Each enum field includes an `x-unknown-default` extension property documenting its safe default. Unknown values MUST be preserved on round-trip.

### Additional Properties

All objects set `additionalProperties: true` (or omit it, which defaults to true). This supports the round-trip preservation requirement — readers MUST store fields they don't recognize so future exports don't lose information.

## Validation

To validate an ALF file against these schemas:

```bash
# Validate the manifest
jsonschema --instance manifest.json manifest.schema.json

# Validate each line of a memory partition
while IFS= read -r line; do
  echo "$line" | jsonschema --instance /dev/stdin memory-record.schema.json
done < memory/partitions/2026-Q1.jsonl
```

Note: Use a validator that supports warning mode for enums, or pre-process to skip enum validation for forward compatibility.

## Relationship to the `.alf` Archive

```
agent-snapshot.alf (ZIP)
├── manifest.json              ← manifest.schema.json
├── identity.json              ← identity.schema.json
├── principals.json            ← principals.schema.json
├── credentials.json           ← credentials.schema.json
├── memory/
│   ├── index.json
│   └── partitions/
│       ├── 2025-Q3.jsonl      ← memory-record.schema.json (per line)
│       ├── 2025-Q4.jsonl      ← memory-record.schema.json (per line)
│       └── 2026-Q1.jsonl      ← memory-record.schema.json (per line)
├── attachments.json           ← attachments.schema.json
├── artifacts/                 (Tier 2 portable artifacts)
│   ├── shares_tracker.csv
│   └── deploy_script.sh
└── raw/                       (no schema — opaque runtime originals)

delta-seq-1248.alf-delta (ZIP)
├── manifest.json              ← delta-manifest.schema.json
├── memory/
│   └── delta.jsonl            ← memory-record.schema.json + operation field (per line)
└── ...                        (only changed layers present)
```