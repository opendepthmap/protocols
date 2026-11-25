# OpenDepthMap Public Protocol Extensions (Phase 1.5 Roadmap)

This document collects public external API contracts, input/output formats, and optional privacy/ecological extensions reserved for **Phase 1.5** (after Day-1 GA launch). Machine-readable draft endpoint and schema definitions live in [openapi-v1.5-draft.yaml](../schemas/openapi-v1.5-draft.yaml).

---

## 1. Extended Open Ingestion Formats & Proprietary Binary Parsers
Day-1 GA accepts `GPX`, `GEOJSON`, `CSV`, and `NMEA_0183`. Phase 1.5 adds:

| Format Code | File Extensions | Category & Specification |
| :--- | :--- | :--- |
| `NMEA_2000` | `.n2k`, `.can`, `.log` | CANboat/Actisense CSV PGN logs and SocketCAN `candump` logs carrying PGNs `128267` (Water Depth), `129025` (Position Rapid), `129029` (GNSS Position Data), and `130310` (Environmental Parameters). Reference fixture: `docs/samples/synthetic-nmea-2000.n2k`. |
| `SUBSURFACE_SSRF` | `.ssrf`, `.xml` | Open dive log XML (`track_modality=DIVE_PROFILE`, excluded from seafloor bathymetry, contours, and IHO B-12 CSB); DTD disabled, max 500 MiB expanded. Reference fixture: `docs/samples/synthetic-dive-profile.ssrf`. |
| `SIGNALK_DELTA` | `.json`, `.ndjson` | Batched Signal K delta JSON/NDJSON log files uploaded via `/api/v1/tracks/upload-url`. |
| `LOWRANCE_SL2` | `.sl2` | Proprietary binary sonar; magic-byte verified (implemented in private ingestion module). |
| `LOWRANCE_SL3` | `.sl3` | Proprietary binary sonar; magic-byte verified (implemented in private ingestion module). |
| `HUMMINBIRD_ZIP` | `.zip` (`.dat` + `.son`) | Proprietary sonar bundle pairing `.dat` UTC header and `.son` frames; max 500 MiB expanded. |
| `GARMIN_RSD` | `.rsd` | Proprietary binary sonar (Beta); magic-byte verified. |

---

## 2. Gridded Bathymetric Raster Exports & Zenodo DOI Releases
Day-1 GA exports full-resolution GeoParquet 1.1, IHO B-12 CSB (`CSV` and `GeoJSON`), OGC API - Features, and `.pmtiles` offline chart packs. Phase 1.5 adds:
- **CF-1.10 NetCDF (`CF_NETCDF`):** `Conventions = "CF-1.10"`, coordinate variables `lon` (`degrees_east`), `lat` (`degrees_north`), data variables `depth` (`units = "m"`, `positive = "down"`), `depth_uncertainty` (`tvu_m`), `sounding_count`, plus global provenance attributes (`vertical_datum`, `gridding_method`, `grid_resolution_m`, `algorithm_version`, `doi`, `license`, `disclaimer`). Unmapped cells beyond `max_search_radius_m` remain `NODATA` (`NaN`); capped at `<= 25,000,000` pixels; requires authentication.
- **Cloud-Optimized GeoTIFF (`COG`):** Single-band or multi-band float32 GeoTIFF (`Band 1: depth_m`, `Band 2: tvu_m`, `Band 3: sounding_count`), `EPSG:4326` or `EPSG:3857`, `512x512` internal tiling, `DEFLATE` compression, internal overviews, and matching provenance tags.
- **DOI Dataset Releases (`GET /api/v1/releases`):** Curated, citable CC0-1.0 regional GeoParquet snapshots linked to Zenodo DOI deposits.

---

## 3. Advanced Identity & Headless Hardware Pairing
Day-1 GA supports Google and GitHub browser sessions plus Personal (`odm_usr_`) and write-only Device (`odm_dev_`) API keys. Phase 1.5 adds:
- **OAuth2 / OIDC Authorization Server (`/oauth2/authorize`, `/oauth2/token`, `/.well-known/openid-configuration`)** for third-party GIS clients and apps.
- **RFC 8628 Device Authorization Grant (`POST /api/v1/auth/device/code`, `POST /api/v1/auth/device/token`, `GET/POST /api/v1/auth/device/verify`)** ahead of Phase 2 autonomous survey buoy hardware.
- **WebAuthn / FIDO2 Passkeys (`/api/v1/auth/passkeys/*`)** plus Email + Password, Sign in with Apple, and OpenStreetMap account linking.

---

## 4. Waypoint Media, Ecological Taxonomies & Extended Privacy Governance
- **Waypoint Photo Uploads (`POST /api/v1/waypoints/{id}/media/upload-url`, `GET /api/v1/waypoints/{id}/media/{mediaId}`):** Single-use uploads (`<= 15 MiB`, JPEG/PNG/WebP), server-side EXIF capture-timestamp extraction for track auto-correlation, and full EXIF PII stripping (camera GPS, serial numbers) prior to auth-checked serving.
- **CMECS & Darwin Core Waypoint Enrichment:** Standardized NOAA CMECS substrate/biotic codes and Darwin Core taxon identifiers on waypoints.
- **Opt-In H3 Shoreline Blurring & Grid Binning:** Optional `fuzz_endpoints = true` (`fuzz_radius_m` `25-500 m` -> enclosing H3 Resolution 8 centroid, `is_fuzzed = true`) and optional regional H3 Resolution 9/10 cell binning on `POST /api/v1/tracks/{id}/publish`.
- **Site Steward Cultural Takedown Portal (`GET/POST /api/v1/privacy/takedowns`):** Self-service takedown polygon workflow for verified marine archaeologists and tribal/heritage stewards.
- **SHA-256 Hash-Chained Deletion Audit Receipts (`GET /api/v1/privacy/deletion-receipts/{receiptId}`):** Downloadable cryptographic deletion verification receipts.
