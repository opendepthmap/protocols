# OpenDepthMap Protocols

Public repository defining OpenDepthMap's vendor-neutral bathymetric data schemas, hardware wire protocols, external API contracts, and sanitized sample fixtures.

**Status:** Phase 1 public API contracts (`openapi-v1.yaml`), geospatial schemas (`geoparquet-sounding-v1.json`, `ogc-features-v1.json`, `iho-b12-csb-v1.json`), and sanitized synthetic fixtures (`docs/samples/`) are locked and ready for Phase 1 implementation. Hardware wire protocols (`docs/protocols/`) remain placeholders for Phases 2 and 3.

## Repository Structure

```text
protocols/
├── docs/
│   ├── protocols/    # Wire protocol and local hardware portal specs
│   └── samples/      # Sanitized synthetic sonar and GPX test fixtures
└── schemas/          # Machine-readable OpenAPI, OGC, GeoParquet, and IHO B-12 schemas
```

## Licensing & Attribution

- **Specifications, Schemas, Documentation & Sample Datasets:** Dedicated to the public domain under CC0 1.0 Universal (`CC0-1.0`).
- **Reference Tooling & SDKs:** Licensed under the MIT License (`MIT`).
- **Base Map Attribution:** Land and shoreline vectors derived from OpenStreetMap follow ODbL 1.0 ([Map data © OpenStreetMap contributors](https://www.openstreetmap.org/copyright)).

## Non-Navigational Disclaimer

Supplemental bathymetric reference only. Not certified for primary marine navigation; always consult official NOAA/IHO nautical charts.
