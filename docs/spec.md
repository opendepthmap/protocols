# OpenDepthMap External Protocols & Public Contract Specification (Phase 1)

- **Version:** `1.0.0`
- **Status:** Baseline Specification (Phase 1 Core System)
- **License:** CC0-1.0 Universal
- **Machine-Readable Schemas:**
  - OpenAPI 3.1 REST Contract: [openapi-v1.yaml](../schemas/openapi-v1.yaml)
  - GeoParquet 1.1 Sounding Schema: [geoparquet-sounding-v1.json](../schemas/geoparquet-sounding-v1.json)
  - IHO B-12 CSB Schema: [iho-b12-csb-v1.json](../schemas/iho-b12-csb-v1.json)
  - OGC API - Features Schema: [ogc-features-v1.json](../schemas/ogc-features-v1.json)

> **Non-Navigational Notice:** Supplemental bathymetric reference only. Not certified for primary marine navigation; always consult official NOAA/IHO nautical charts.

---

## 1. Domain & Base Endpoint Architecture

| Surface | Canonical Host | Purpose |
| :--- | :--- | :--- |
| Portal, REST, OAuth2/OIDC, OGC & Private MVT | `https://opendepthmap.org` | Dark-water web app plus same-origin `/api/v1/*`, `/oauth2/*`, `/.well-known/*`, `/ogc/*`, and `/tiles/private/*` (strictly first-party cookies, zero CORS preflights, single WebAuthn `rpId`) |
| Static Public Vector Tiles | `https://tiles.opendepthmap.org` | Cookie-less CDN host for zero-server byte-range `.pmtiles` archives and `/tiles/manifest.json` |
| Optional Brand Redirects | `https://opendepthmap.com` | Plain 301 redirect to `https://opendepthmap.org` |

All API payloads use UTF-8 JSON (`application/json`, `application/geo+json`, or `application/problem+json` per RFC 9457 for errors). All timestamps use RFC 3339 UTC (`YYYY-MM-DDTHH:MM:SSZ` or millisecond precision). All spatial coordinates use `OGC:CRS84` (WGS84 longitude, latitude order `[lon, lat]` in decimal degrees, `EPSG:4326` compatible).

---

## 2. Authentication, Scopes & Rate Limits

### 2.1 Public Zero-Login Access
Public map tiles (`.pmtiles`), public OGC API - Features queries, public dataset release metadata, and public contributor profiles require **zero authentication**.

### 2.2 Authentication Flows
1. **Browser & Third-Party OAuth2 / OIDC:**
   - Standard Authorization Code + PKCE flow via `/.well-known/openid-configuration`, `/oauth2/authorize`, and `/oauth2/token`.
   - Day-1 contributor sign-in uses **Google** and **GitHub** OAuth2/OIDC plus **WebAuthn / FIDO2 Passkeys** (Email + Password, Apple, and OpenStreetMap linking deferred to Phase 1.5+).
2. **Headless Field Device Pairing (OAuth2 Device Authorization Grant, RFC 8628):**
   - `POST /api/v1/auth/device/code` initiates pairing and returns `device_code`, `user_code`, `verification_uri`, `expires_in`, and `interval`.
   - `POST /api/v1/auth/device/token` polls until the user approves pairing in the web portal and returns a device bearer token and/or provisioned Device API Key.
3. **Scoped API Keys (`X-API-Key` Header):**
   - **Personal User API Keys** (`odm_usr_` prefix): Created by authenticated users for Python/DuckDB scripts, QGIS plugins, and elevated quotas. Supports expiration, instant revocation, and explicit scopes.
   - **Device / Vessel API Keys** (`odm_dev_` prefix): Bound strictly to a single registered device/vessel (`device_id`). Permanently restricted to `write:tracks` scope only. A Device API Key MUST NEVER be granted `read:private_vault`, `write:waypoints`, or account-management scopes.

### 2.3 OAuth2 & API Key Scopes

| Scope | Allowed Credential Types | Capabilities |
| :--- | :--- | :--- |
| `read:public` | Public (anonymous), User Token, User API Key | Read public datasets, OGC features, public releases at authenticated quota |
| `read:prefs` | User Token, User API Key | Read contributor profile, preferences, and registered device metadata |
| `write:tracks` | User Token, User API Key, Device API Key | Request upload URLs for owned devices, upload raw logs, query own job status |
| `write:waypoints` | User Token, User API Key | Create, enrich (CMECS/Darwin Core), and update dive waypoints |
| `read:private_vault` | User Token, User API Key | Read unredacted private tracks, private GeoParquet downloads, and private MVT tiles |

### 2.4 Sliding-Window Rate Limits
Standard HTTP headers `X-RateLimit-Limit`, `X-RateLimit-Remaining`, and `Retry-After` are returned on throttled responses (`HTTP 429 Too Many Requests`):
- **Anonymous Public IP:** `60 requests / minute` (OGC API and REST metadata; static `.pmtiles` CDN byte-range requests are cached at edge).
- **Device / Vessel API Key:** `120 requests / minute`.
- **Personal User API Key / OAuth2 Bearer:** `300 requests / minute`.

---

## 3. Two-Step Direct Upload Contract & Ingestion Lifecycle

### 3.1 Mandatory Pre-Upload Device Registration & Effective-Dated Calibrations
Every track upload MUST reference an active registered `device_id` owned by the authenticated user (created via `POST /api/v1/devices` or OAuth2 Device Flow). Because sensor rigs evolve over time (remounted transducers, changed draft arms, swapped GNSS masts), geometry offsets are stored as **non-overlapping effective-dated calibration revisions** (`POST /api/v1/devices/{deviceId}/calibrations`):
- `effective_from`, `effective_to` (RFC 3339 UTC; `effective_to = null` indicates the currently active revision)
- `depth_reference`: `BELOW_TRANSDUCER` (default; applies `draft_offset_m`) or `BELOW_WATERLINE` (prevents double-counting when chartplotters already bake draft into exported depth)
- `device_category`: `SURFACE_BUOY`, `RECREATIONAL_VESSEL`, `DIVER_COMPUTER`, `USV_AUTONOMOUS`, `THIRD_PARTY_PLOTTER`
- `draft_offset_m` (meters below waterline, default `0.00`)
- `antenna_offset_x_m`, `antenna_offset_y_m`, `antenna_offset_z_m` (meters in vessel NED-body frame: `+X` forward, `+Y` starboard, `+Z` down)
- `transducer_freq_khz` (integer kHz, optional)
- `beamwidth_deg` (degrees, optional)
- `assumed_sound_speed_ms` (m/s, default `1500.00`)

At ingestion time, the server resolves the calibration revision active at the track's actual UTC timestamp (`started_at`) and snapshots it immutably onto the track. Updating future or past revisions never silently mutates existing tracks; owners may explicitly trigger `POST /api/v1/tracks/{trackId}/recalibrate` to re-run calibration over immutable raw soundings.

### 3.2 Upload Flow
1. Client sends `POST /api/v1/tracks/upload-url` with `write:tracks` scope containing `device_id`, `filename`, `byte_size` (max `262144000` bytes / 250 MiB), `source_format`, and optional track metadata (`title`, `water_body_type`, `depth_unit` for CSV, `tide_station_id_override`, `manual_water_level_offset_m`). When authenticated with a Device API Key (`odm_dev_`), `device_id` MUST match the key's bound device.
2. Server validates ownership, extension/format compatibility, and size cap, creates an ingestion job in status `PENDING_UPLOAD`, and returns:
   - `job_id` (UUID)
   - `upload_url` (single-use signed HTTP `PUT` URL valid for 15 minutes)
   - `required_headers` (`Content-Type` plus any server-supplied single-use upload precondition headers; never requires browser-forbidden `Content-Length`)
   - `expires_at` (RFC 3339 UTC)
3. Client executes `HTTP PUT` of the raw file bytes directly to `upload_url` with exact `required_headers`.
4. Upon upload completion, backend ingestion transitions asynchronously through deterministic states:
   `PENDING_UPLOAD` -> `QUEUED` -> `PARSING` -> `QC_RUNNING` -> `TIDE_REDUCING` -> `COMPLETED` (or `FAILED`; abandoned uploads past expiration transition to `EXPIRED`).
5. Client polls `GET /api/v1/jobs/{jobId}` for `status`, `progress_pct` (`0` to `100`), `track_id` (once created), and structured `error_code` / `error_detail` on failure.

### 3.3 Supported Input Formats (Day-1 GA vs. Phase 1.5 Proprietary Pack)

| Format Code | File Extensions | Availability | Category & Hardening Limits |
| :--- | :--- | :--- | :--- |
| `GPX` | `.gpx` | **Day-1 GA** | Open XML (Garmin/depth extensions); DTD disabled, max 500 MiB expanded |
| `GEOJSON` | `.geojson`, `.json` | **Day-1 GA** | Open JSON (RFC 7946 `FeatureCollection`, `LineString`, or `Point`; `feature_type=waypoint` extracted) |
| `CSV` | `.csv` | **Day-1 GA** | Open tabular requiring `lon`, `lat`, `recorded_at_utc`, and exactly one depth column (`depth_m`, `depth_ft`, `depth_fathoms`, or `depth`); unit-header mismatch rejected (`UNIT_HEADER_MISMATCH`) |
| `NMEA_0183` | `.nmea`, `.txt`, `.log` | **Day-1 GA** | Text marine stream; strict checksum validation (`GGA`, `RMC`, `DBT`, `DPT`, `MTW`); `$SDDPT` takes precedence over `$SDDBT` |
| `NMEA_2000` | `.n2k`, `.can`, `.log` | **Day-1 GA** | CANboat/Actisense CSV PGN logs and SocketCAN `candump` logs carrying PGNs `128267` (Water Depth), `129025` (Position Rapid), `129029` (GNSS Position Data), and `130310` (Environmental Parameters) |
| `SUBSURFACE_SSRF` | `.ssrf`, `.xml` | **Day-1 GA** | Open dive log XML (`track_modality=DIVE_PROFILE`, excluded from seafloor bathymetry); DTD disabled, max 500 MiB expanded |
| `SIGNALK_DELTA` | `.json`, `.ndjson` | **Day-1 GA** | Batched Signal K delta JSON/NDJSON log files uploaded via `/api/v1/tracks/upload-url` |
| `LOWRANCE_SL2` | `.sl2` | Phase 1.5 | Proprietary binary sonar; magic-byte verified |
| `LOWRANCE_SL3` | `.sl3` | Phase 1.5 | Proprietary binary sonar; magic-byte verified |
| `HUMMINBIRD_ZIP` | `.zip` (`.dat` + `.son`) | Phase 1.5 | Proprietary sonar bundle pairing `.dat` UTC header and `.son` frames; max 500 MiB expanded |
| `GARMIN_RSD` | `.rsd` | Phase 1.5 (Beta) | Proprietary binary sonar; magic-byte verified |

---

## 4. Canonical REST API Summary

Full request/response payloads, status codes, and RFC 9457 error objects are specified in [openapi-v1.yaml](../schemas/openapi-v1.yaml).

| Group | Endpoint | Auth / Scope | Purpose |
| :--- | :--- | :--- | :--- |
| Auth (Session) | `POST /api/v1/auth/logout`, `GET/PATCH /api/v1/auth/me`, `POST /api/v1/auth/accept-terms`, `DELETE /api/v1/auth/account` | `SessionCookie` | Interactive session, profile updates, Contributor Terms, and account hard-deletion |
| Auth (Passkeys/OAuth) | `GET /api/v1/auth/passkeys`, `PATCH/DELETE /api/v1/auth/passkeys/{id}`, `POST /api/v1/auth/passkeys/register\|login/options\|verify`, `GET /api/v1/auth/external/{provider}/start\|callback`, `DELETE /api/v1/auth/external/{provider}` | Public / `SessionCookie` | FIDO2 Passkey management (including Admin step-up), Google/GitHub login/linking (rejects removing last auth method with `409`) |
| Auth (Device) | `POST /api/v1/auth/device/code`, `POST /api/v1/auth/device/token`, `GET/POST /api/v1/auth/device/verify` | Public / `SessionCookie` | RFC 8628 headless device pairing initiation, polling, pending-code lookup, and user approval |
| User Keys | `GET/POST /api/v1/api-keys`, `DELETE /api/v1/api-keys/{id}` | `SessionCookie` | Manage Personal User API Keys (`odm_usr_`) |
| Devices | `GET/POST /api/v1/devices`, `PATCH/DELETE /api/v1/devices/{id}`, `POST /api/v1/devices/{id}/rotate-key` | Session / Bearer / User Key | Register vessels/sensors, update device name/category, rotate or soft-deactivate Device API Keys (`odm_dev_`) |
| Calibrations | `GET/POST /api/v1/devices/{id}/calibrations`, `PATCH /api/v1/devices/{id}/calibrations/{revisionId}` | Session / Bearer / User Key | Manage non-overlapping effective-dated sensor rig geometry revisions |
| Ingestion | `POST /api/v1/tracks/upload-url` | `write:tracks` (Device Key bound to own `device_id`) | Request 15-minute single-use signed direct upload URL |
| Jobs | `GET /api/v1/jobs`, `GET /api/v1/jobs/{jobId}` | Owner (Device Key restricted to own `INGESTION` jobs) | List and inspect ingestion, recalibration, publish, export, or offline pack jobs |
| Tracks | `GET /api/v1/tracks` (`q`, `device_id`, `modality`, `is_public`, `from`/`to`, `bbox`, `sort`), `GET/PATCH/DELETE /api/v1/tracks/{id}`, `GET /api/v1/tracks/{id}/privacy-settings`, `GET /api/v1/tracks/{id}/profile`, `GET /api/v1/tracks/{id}/nearby-tide-stations` | Owner (`read:private_vault`) or Public (redacted/trimmed only) | Search, filter, inspect, edit, or hard-delete tracks; read saved privacy trims, depth profiles, and nearby NOAA stations |
| Tracks | `POST /api/v1/tracks/{id}/recalibrate`, `POST /api/v1/tracks/{id}/publish`, `POST /api/v1/tracks/{id}/unpublish` | Owner Session / Bearer | Re-run calibration over immutable raw soundings, trim/redact and publish publicly, or pull a published track back to private |
| Search & Marine Stations | `GET /api/v1/search`, `GET /api/v1/tides/stations` | Public | Unified map search (published sites/tracks, tide stations, cached OSM places) and cached NOAA CO-OPS tide & NDBC buoy stations |
| Privacy | `GET/POST /api/v1/privacy/redaction-zones`, `PATCH/DELETE /api/v1/privacy/redaction-zones/{id}`, `GET/POST /api/v1/privacy/takedowns`, `GET /api/v1/privacy/deletion-receipts/{id}` | Owner / `SiteSteward` | Personal spatial exclusion polygons, cultural takedowns, and deletion receipts |
| Waypoints | `GET/POST /api/v1/waypoints`, `PATCH/DELETE /api/v1/waypoints/{id}`, `POST /api/v1/waypoints/{id}/media/upload-url`, `GET /api/v1/waypoints/{id}/media/{mediaId}` | Public (if `is_public`) or Owner (`write:waypoints`) | Manage waypoints (CMECS/Darwin Core) and EXIF-stripped photo uploads (`<= 15 MiB`, served via auth-checked endpoint) |
| Routes | `GET/POST /api/v1/routes`, `PATCH/DELETE /api/v1/routes/{id}`, `GET /api/v1/routes/{id}/compare/{trackId}` | Owner Session / Bearer | Planned water routes & survey transects (dive, kayak, boat, survey), leg magnetic bearings, and planned-vs-actual drift comparison |
| Offline Packs | `POST /api/v1/packs/offline` | Session / Bearer / User Key | Queue regional `.pmtiles` offline chart pack build (`<= 10,000 km^2`) |
| Exports | `POST /api/v1/exports`, `GET /api/v1/exports/{jobId}`, `GET /api/v1/releases` | Public / Scoped Key (`CF_NETCDF`/`COG` require auth) | GeoParquet, IHO B-12, CF-NetCDF, COG exports, anonymous public export polling, and DOI dataset releases |
| Contributors | `GET /api/v1/contributors/{displayName}` | Public | View public contributor profile and published traces |
| Tiles | `GET https://tiles.opendepthmap.org/tiles/manifest.json`, `GET /tiles/private/{z}/{x}/{y}.mvt` | Public (manifest) / `read:private_vault` (MVT) | Regional `.pmtiles` shard manifest and dynamic private MVT vector tiles |
| System & Stats | `GET /api/v1/stats/public`, `GET /api/v1/system/status`, `GET /health/live`, `GET /health/ready` | Public | Live coverage counters, active public feature/maintenance flags, and liveness/readiness probes |

---

## 5. OGC API - Features Contract

OpenDepthMap implements **OGC API - Features Part 1 (Core)** and **Part 2 (Coordinate Reference Systems by Reference)** under `/ogc`, defined formally in [ogc-features-v1.json](../schemas/ogc-features-v1.json):

- `GET /ogc`: Landing page with links (`self`, `service-desc`, `conformance`, `data`) and mandatory non-navigational notice.
- `GET /ogc/conformance`: Conformance classes (`core`, `oas30`, `geojson`, `crs`).
- `GET /ogc/collections`: Lists available public collections:
  - `public-tracks`: Published, post-redaction, post-fuzz-trimmed bathymetric track summaries (`MultiLineStringZ`, split across redacted polygon gaps; never raw tracks or mid-water `DIVE_PROFILE` logs).
  - `public-soundings`: Published, privacy-redacted bathymetric soundings (`Point`, with `is_fuzzed`, `vertical_datum`, `calibration_tier`, `qartod_qc_flag`, `tvu_m`, `thu_m`, `iho_s44_order`, `provenance`).
  - `public-waypoints`: Published subsea ecological and structural waypoints (`PointZ`, CMECS/Darwin Core fields).
- `GET /ogc/collections/{collectionId}/items`: Supports `bbox` (`minLon,minLat,maxLon,maxLat`), `bbox-crs`, `crs`, `datetime` (RFC 3339 instant or `start/end` interval), `limit` (`1` to `1000`, default `100`), cursor pagination (`next` link), and domain filters (`vertical_datum`, `calibration_tier`, `qartod_qc_flag`, `iho_s44_order`, `provenance`).

Every OGC FeatureCollection response includes a top-level `attribution` and `disclaimer` metadata object carrying OpenStreetMap attribution (when applicable) and the mandatory non-navigational warning.

---

## 6. GeoParquet 1.1 Sounding Data Contract

Full-resolution bathymetric archives conform to **GeoParquet 1.1.0** (`[geoparquet-sounding-v1.json](../schemas/geoparquet-sounding-v1.json)`):

- **Parquet File Footer (`geo` key):**
  - `version`: `"1.1.0"`
  - `primary_column`: `"geometry"`
  - `columns.geometry.encoding`: `"WKB"`
  - `columns.geometry.geometry_types`: `["Point"]`
  - `columns.geometry.crs`: `OGC:CRS84` (WGS84 lon/lat)
  - `columns.geometry.covering.bbox`: `{ xmin: ["bbox", "xmin"], ymin: ["bbox", "ymin"], xmax: ["bbox", "xmax"], ymax: ["bbox", "ymax"] }`
- **Partitioning Convention (Public Archives):** Partitioned by coarse H3 Resolution 5 (`h3_r5={cell}/track_id={uuid}.parquet`) to prevent small-file explosion while storing `h3_index_r8` and `h3_index_r9` columns inside Parquet row groups for fast DuckDB/QGIS filtering.
- **Key Enumerations:**
  - `vertical_datum`: `UNREDUCED`, `MLLW`, `IGLD`, `LOCAL_LAKE`, `MANUAL_DATUM`
  - `calibration_tier`: `UNREDUCED`, `TIDE_REDUCED`, `SV_TIDE_CALIBRATED`, `MOTION_COMPENSATED`
  - `iho_s44_order`: `UNASSESSED_CSB` (default for all unverified crowd/recreational uploads), `SPECIAL_ORDER`, `ORDER_1A`, `ORDER_1B`, `ORDER_2`
  - `qartod_qc_flag`: `1` (Good), `2` (Not Evaluated), `3` (Suspect), `4` (Bad)
  - `provenance`: `user-track`, `noaa-dcdb`, `usgs-3dep`, `gebco`

### 6.1 Public Privacy Redaction Guarantees
Before any sounding enters public GeoParquet archives, OGC collections, or public `.pmtiles`:
1. Any sounding inside an owner's `redaction_zones` polygon or an active `protected_site_takedowns` polygon is excluded from public output.
2. Any sounding within the contributor's chosen `trim_start_m` cumulative distance from track start or `trim_end_m` cumulative distance from track end (default `0 m`, user-adjustable via slide bars up to `5,000 m`) is completely erased from public output, preserving short shore dives when set to `0` while making beach-entry trimming effortless.
3. If the contributor optionally opts into `fuzz_endpoints = true` (default `false`), soundings within their configured `fuzz_radius_m` (default `150.0 m`, configurable `25.0 m` to `500.0 m` plus optional per-track jitter) of track start, track end, or registered shoreline launch ramps have their `lon`, `lat`, and `geometry` replaced by the centroid of their enclosing **H3 Resolution 8** cell (`~460 m` edge length) with `is_fuzzed = true`. Whenever trimming or endpoint fuzzing is active, public timestamps are quantized to 15-minute buckets to prevent launch reconstruction, and public track geometries (`MultiLineStringZ`) are trimmed of hidden endpoints and split across redacted gaps.
4. If the contributor enables optional regional grid binning (`H3 Resolution 9` or `H3 Resolution 10`), soundings are aggregated into H3 cell summaries (`min_depth_m`, `mean_depth_m`, `median_depth_m`, `max_depth_m`, `sounding_count`, `vertical_datum`).

---

## 7. Open Export Formats & Vector Tile Schema

### 7.1 IHO B-12 Crowd-Sourced Bathymetry (CSB)
Defined in [iho-b12-csb-v1.json](../schemas/iho-b12-csb-v1.json). Exports both **IHO B-12 GeoJSON** (metadata header with `platform`, `trustedNode`, `crs`, `verticalDatum`, `license: "CC0-1.0"`, and `features` array with `depth`, `time`, `uncertainty`; single-track `scope=TRACK` outputs one GeoJSON file, while multi-track `scope=BBOX|RELEASE` outputs a `.zip` bundle containing one valid B-12 GeoJSON per `(track_id, verticalDatum)`) and flat **IHO B-12 CSV** (`LON,LAT,DEPTH,TIME,PLATFORM_NAME,UNCERTAINTY,VERTICAL_DATUM,QARTOD_FLAG`, with CSV formula-injection escaping on string columns). Mid-water `DIVE_PROFILE` tracks are excluded from IHO B-12 CSB exports.

### 7.2 CF-1.10 NetCDF & Cloud-Optimized GeoTIFF (COG)
Gridded bathymetry exports leave unmapped cells beyond `max_search_radius_m` strictly as `NODATA` (`NaN`) rather than interpolating across sparse gaps, enforce a maximum export size of `25,000,000` grid pixels (and `10,000 km^2` for offline packs), and include standardized metadata:
- **CF-1.10 NetCDF:** `Conventions = "CF-1.10"`, coordinate variables `lon` (`degrees_east`), `lat` (`degrees_north`), data variables `depth` (`standard_name = "sea_floor_depth_below_sea_surface"` or datum-specific `long_name`, `units = "m"`, `positive = "down"`), `depth_uncertainty` (`tvu_m`), `sounding_count`, plus global provenance attributes (`vertical_datum`, `gridding_method`, `grid_resolution_m`, `algorithm_version`, `doi`, `license`, `disclaimer`).
- **COG:** Single-band or multi-band float32 GeoTIFF (`Band 1: depth_m`, `Band 2: tvu_m`, `Band 3: sounding_count`), `EPSG:4326` or `EPSG:3857`, internal tiling `512x512`, `DEFLATE` compression, internal overviews, and GDAL metadata tags matching CF-NetCDF provenance.

### 7.3 Public & Offline `.pmtiles` Vector Tile Schema
Single-file `.pmtiles` archives served over HTTP byte-range requests contain up to four bathymetric layers (kept separate from self-hosted OSM land/shoreline `.pmtiles` to honor ODbL 1.0 boundaries):

| Layer Name | Geometry | Zoom Range | Standardized Feature Attributes |
| :--- | :--- | :--- | :--- |
| `depth_soundings` | `Point` | `z10` to `z16` | `track_id`, `depth_m`, `raw_depth_m`, `vertical_datum`, `calibration_tier`, `iho_s44_order`, `qartod_qc_flag`, `tvu_m`, `thu_m`, `is_fuzzed`, `provenance` |
| `depth_contours` | `LineString` | `z6` to `z16` | `depth_m`, `vertical_datum`, `contour_interval_m`, `provenance` |
| `depth_h3_bins` | `Polygon` | `z4` to `z12` | `h3_index`, `h3_res`, `mean_depth_m`, `min_depth_m`, `max_depth_m`, `sounding_count`, `vertical_datum`, `provenance` |
| `waypoints` | `Point` | `z8` to `z16` | `waypoint_id`, `title`, `feature_category`, `substrate_type`, `cmecs_substrate_code`, `cmecs_biotic_code`, `darwin_core_taxon`, `recorded_depth_m` |

Offline pack manifests (`manifest.json` bundled alongside `.pmtiles` metadata) include `pack_id`, `bbox`, `generated_at_utc`, `included_layers`, `include_noaa_baseline`, `osm_attribution`, and the mandatory non-navigational disclaimer.

---

## 8. Sanitized Synthetic Sample Fixtures

Public synthetic fixtures (zero real user coordinates or private launch locations) live in `docs/samples/` for third-party developers, GIS engineers, and CI validation:
- [synthetic-coastal-track.gpx](samples/synthetic-coastal-track.gpx): Synthetic coastal survey GPX track with depth/temperature extensions, realistic spike outlier, and shoreline entry segment.
- [synthetic-lake-track.geojson](samples/synthetic-lake-track.geojson): Synthetic inland freshwater lake track (`LOCAL_LAKE`, `0.0 PSU` salinity) with dive waypoint metadata.
- [synthetic-nmea-0183.nmea](samples/synthetic-nmea-0183.nmea): Valid checksummed NMEA 0183 stream (`$GPGGA`, `$GPRMC`, `$SDDBT`, `$SDDPT`, `$YXMTW`) for parser interoperability testing.
- [synthetic-anglers-feet.csv](samples/synthetic-anglers-feet.csv): Synthetic freshwater CSV track in feet (`depth_unit=FEET`) verifying unit conversion to positive-downward meters.
- [synthetic-dive-profile.ssrf](samples/synthetic-dive-profile.ssrf): Synthetic Subsurface scuba dive log verifying `track_modality=DIVE_PROFILE` classification and exclusion from seafloor bathymetry.
