# OpenDepthMap Protocols Governance & Developer Guardrails

## Visibility Tier: TIER 3 - PUBLIC OPEN SOURCE (ORGANIZATION)
- Target remote: `github.com/opendepthmap/protocols` (Public).
- Licensing: CC0-1.0 for specifications, schemas, and sanitized sample datasets; MIT for code/tooling.

## Hard Boundary Rules
- Keep this repository strictly scoped to public specifications, open schemas, sanitized synthetic fixtures, and open tooling. Never commit closed-source services, internal infrastructure, private notes, or credentials.
- NEVER commit secrets, real user coordinates/tracks (all sample fixtures MUST be synthetic or fully anonymized), or private endpoints.
- Keep documentation and public interface descriptors free of em dashes.

## What Belongs Here
- Public problem statement and human-factors intent (`docs/intent.md`)
- External API, OGC API - Features, GeoParquet, and RS-485 wire protocol specifications (`docs/spec.md`, `docs/protocols/`)
- Machine-readable JSON/OpenAPI/IHO B-12 schemas (`schemas/`)
- Sanitized synthetic sample sonar/GPX fixtures (`docs/samples/`)
- Future public Python SDK / CLI tooling (MIT)

## Automatic Milestone Gate (Scope, Security & Clarity)
Before finalizing any schema, endpoint, or spec change:
1. **YAGNI / Scope Discipline:** Keep `schemas/openapi-v1.yaml` and `docs/spec.md` strictly scoped to active Day-1 GA contracts; place deferred formats or endpoints in `schemas/openapi-v1.5-draft.yaml` and `docs/phase-1.5-roadmap.md`.
2. **Security & Privacy Check:** Verify all endpoints specify exact auth/scope schemes, never expose private coordinates or raw device identifiers on public routes, and contain zero internal infrastructure details.
3. **Validation & Push Hygiene:** Validate all YAML/JSON `$ref`s and schemas before declaring done, and always ask the user explicitly before running `git push`.
