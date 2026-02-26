# Contributing to the Agent Life Format

Thank you for your interest in improving ALF. This document explains how to propose changes to the specification.

## Scope

This repository contains the **format specification and JSON schemas**. It does not contain runtime implementations, adapters, or the sync service — those live in separate repositories.

Changes to this repository affect what the data looks like, not how specific software operates on it.

## Types of Contributions

### Bug Reports

If you find an inconsistency in the specification (e.g., a section contradicts another, a JSON example doesn't match the schema, a cross-reference is broken), please [open an issue](https://github.com/agent-life/agent-life-data-format/issues/new) with:

- The section numbers involved
- What the spec says vs. what you believe it should say
- Any relevant context

### Minor Clarifications

For typos, wording improvements, or clarifications that don't change the meaning of the spec, open a pull request directly. Keep the PR description brief — explain what was unclear and how your change improves it.

### Substantive Changes (RFC Process)

For changes that add, remove, or modify fields, schemas, behavioral rules, or design decisions:

1. **Open a discussion issue** describing the problem, the proposed solution, and the tradeoffs. Use the title format: `RFC: <short description>` (e.g., `RFC: Add priority field to relational links`).

2. **Gather feedback** in the issue. The goal is to reach rough consensus before writing spec text.

3. **Submit a pull request** with the spec changes once the approach is agreed upon. The PR should include:
   - Updated `SPECIFICATION.md` sections
   - Updated JSON schemas (if applicable)
   - New or updated test cases in §10
   - A new entry in §11 (Design Decision Log)

4. **Review and merge.** Maintainers review for consistency with the spec's design philosophy (runtime neutrality, forward compatibility, no information loss).

## Design Philosophy

When proposing changes, keep these principles in mind:

- **Runtime neutrality** — the format must not favor or require any specific runtime, LLM provider, or execution engine.
- **No information loss** — exporting from a supported runtime and importing back must preserve all agent state.
- **Forward compatibility** — readers must handle unknown fields and enum values gracefully. Changes should not break older readers within the same major version.
- **Minimal schema** — add fields only when they serve a clear, documented purpose. Optional fields are preferred over required fields for new additions.
- **Testability** — every behavioral rule should be verifiable by a test case.

## Schema Changes

If your change modifies a JSON schema:

- Update the schema file in `schemas/`
- Ensure all JSON examples in `SPECIFICATION.md` still validate against the updated schema
- Keep `additionalProperties: true` on all objects (round-trip preservation)
- For new enum values, add them to the `enum` array and document the `x-unknown-default` behavior

## Versioning

The specification uses [Semantic Versioning](https://semver.org/):

- **Patch** (`1.0.x`) — typos, clarifications, non-normative changes
- **Minor** (`1.x.0`) — new optional fields, new enum values, new sections
- **Major** (`x.0.0`) — removed fields, changed field types, breaking behavioral changes

## Code of Conduct

Be respectful, constructive, and assume good intent. Technical disagreements are welcome; personal attacks are not.
