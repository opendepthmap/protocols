# OpenDepthMap Public Intent & Problem Statement

- **Version:** `1.0.0` (Phase 1 Core System Baseline)
- **License:** CC0-1.0 Universal (Public Domain Dedication)
- **Scope:** Public mission, user personas, human-factors safety principles, and Phase 1 open contract scope for the OpenDepthMap bathymetric mapping ecosystem.

> **Non-Navigational Notice:** Supplemental bathymetric reference only. Not certified for primary marine navigation; always consult official NOAA/IHO nautical charts.

---

## 1. Why OpenDepthMap Exists

OpenDepthMap is an open-source, vendor-neutral bathymetric mapping ecosystem and subsea GIS data standard designed to solve six persistent gaps in underwater mapping:

1. **Financial and Navigational Barriers:** Commercial hydrographic survey rigs and Ultra-Short Baseline (USBL) positioning systems cost tens of thousands of dollars, placing precise bathymetric logging out of reach for individual divers, citizen scientists, and local search-and-recovery groups.
2. **Closing the Nearshore "White Ribbon" Survey Gap:** Official hydrographic survey ships draw too much draft to safely survey shallow rocky coastlines, intertidal reefs, and kelp beds (under 10 to 15 meters depth), while satellite-derived bathymetry (SDB) fails in turbid, dark, or optically complex water.
3. **Temporal Change Detection After Storms and Dredging:** Mobile sandbars, river mouths, and harbor approaches shift rapidly after storms and dredging operations. Open, repeatable survey logs with consistent datum reduction enable localized bathymetric differencing over time.
4. **Rescuing Trapped Sonar Data from Proprietary Silos:** Recreational boaters and anglers generate vast quantities of single-beam sonar logs locked inside closed vendor formats (Lowrance `.sl2`/`.sl3`, Garmin `.rsd`, Humminbird `.son`). OpenDepthMap defines an open vault and standardized open GIS export pipeline (GeoParquet, OGC API - Features, IHO B-12 CSB, CF-NetCDF, COG, PMTiles).
5. **Uncharted Inland Lakes, Quarries, and Reservoirs:** Millions of small inland water bodies lack official bathymetric charts. Community mapping with explicit freshwater sound-velocity and datum handling unlocks usable depth references for local divers, limnologists, and watershed managers.
6. **Subsea Ecological Ground-Truthing and Hazard Mapping:** Raw depth soundings gain lasting scientific and practical value when paired with diver-verified substrate classifications, ecological tags (CMECS and Darwin Core), and geo-referenced media.

---

## 2. Core User Personas & Workflow Outcomes

### Persona 1: The Recreational Scuba Diver & Forager
- **Needs:** High-contrast dark-water bathymetric map visuals, pre-dive route planning with leg distances and magnetic headings, planned-vs-actual track drift review, and strict privacy protection for sensitive reefs, ledges, and shoreline entry ramps.
- **Phase 1 Outcome:** Zero-login dark-water map exploration, private-by-default dive vault, interactive polygon redaction, automatic 150.0 m shoreline entry-endpoint fuzzing before public sharing, pre-dive route drawing, and downloadable offline `.pmtiles` regional chart packs.

### Persona 2: The Coastal Boater, Angler & Small-Craft Operator
- **Needs:** Explicit vertical datum labeling (never conflating unreduced transducer depth with tide-reduced MLLW), shoal and channel depth clarity, and frictionless upload of existing chartplotter sonar logs (`.sl2`, `.sl3`, `.rsd`, `.son`, GPX, NMEA).
- **Phase 1 Outcome:** Direct upload of multi-vendor sonar logs paired with one-time vessel/transducer offset registration, automated NOAA CO-OPS MLLW tide reduction for US coastal waters, and real-time comparison between raw crowd soundings and tide-reduced MLLW depths.

### Persona 3: The DIY Hardware Maker & Embedded Tinkerer
- **Needs:** Vendor-neutral device registration, open upload and OAuth2 Device Authorization APIs, and standardized metadata schemas for custom transducers, static draft offsets, and GNSS-to-transducer lever-arm geometry.
- **Phase 1 Outcome:** Headless device pairing via OAuth2 Device Flow (RFC 8628), write-only vessel/device API keys, and documented external upload/telemetry contracts ready for future Phase 2 survey buoys and Phase 3 diver wrist receivers.

### Persona 4: The GIS / Spatial Data Engineer & Marine Scientist
- **Needs:** Strict CRS/EPSG compliance (`OGC:CRS84` / `EPSG:4326`), transparent QARTOD QC flags, UNESCO sound-velocity calibration, 95% confidence Total Vertical/Horizontal Uncertainty (TVU/THU) mapped to IHO S-44 Survey Orders, CMECS/Darwin Core ecological metadata, and reproducible open-format exports.
- **Phase 1 Outcome:** Direct GeoParquet 1.1 archive access, OGC API - Features endpoints, IHO B-12 CSB public domain datasets, CF-1.10 NetCDF and Cloud-Optimized GeoTIFF (COG) gridded exports, and DOI-ready dataset releases with documented gridding provenance.

---

## 3. Human-Factors, Trust & Safety Requirements

Every client, service, and dataset conforming to OpenDepthMap protocols MUST uphold these invariants:

1. **Supplemental Navigation Boundary:** OpenDepthMap outputs are supplemental bathymetric references only and are never certified for primary marine navigation or life-safety dive decompression planning. All map interfaces, OGC API collections, and exported manifests MUST carry the non-navigational disclaimer.
2. **Vertical Datum Truthfulness:** Unreduced raw transducer depths MUST NEVER be labeled or mixed silently with tide-reduced chart datums (`MLLW`, `IGLD`, `LOCAL_LAKE`, `MANUAL_DATUM`). Every sounding, vector tile feature, and export layer MUST carry an explicit `vertical_datum` attribute.
3. **Immutable Raw Provenance:** Raw uncalibrated soundings are preserved separately from derived calibrated/reduced products so scientific corrections (sound velocity, motion compensation, tide reduction) can be audited and reproduced without losing original measurements.
4. **Privacy-by-Default & Cultural Site Protection:**
   - All contributor uploads default to private (`is_public = false`).
   - Prior to public exposure, any sounding within 150.0 m geodesic distance of a track start coordinate, track end coordinate, or registered shoreline launch ramp is stripped of exact coordinates and snapped to its enclosing H3 Resolution 8 cell centroid (`is_fuzzed = true`).
   - User-defined spatial redaction polygons suppress interior soundings from public exports and tiles.
   - Verified marine archaeologists, tribal nations, and site stewards can register protected site takedown polygons that suppress public exposure while preserving private contributor records.
5. **Strict Credential Separation:** Personal User API Keys (`read:public`, `read:private_vault`, `write:tracks`, `write:waypoints`) are strictly separated from Device/Vessel API Keys (`write:tracks` only) so a lost or compromised field sensor cannot read private user vaults.
6. **Transparent Data Provenance & Attribution:** External public baselines (such as NOAA DCDB / IHO CSB, GEBCO, USGS 3DEP) are tagged distinctly from user-submitted tracks. Base map land/shoreline vectors derived from OpenStreetMap strictly comply with ODbL 1.0 attribution (*"Map data © OpenStreetMap contributors"*) and remain in clean layers separate from CC0-1.0 bathymetric soundings.

---

## 4. Phased Scope & Explicit Non-Goals

### In Scope for Phase 1 (Current Specification)
- Public external REST API, OAuth2/OIDC & Device Flow, Personal vs. Device API Key semantics, and OGC API - Features contracts.
- Two-step direct upload contract and asynchronous ingestion job lifecycle.
- GeoParquet 1.1 full-resolution sounding schema, QARTOD QC flags, IHO S-44 survey orders, and CMECS/Darwin Core waypoint metadata schema.
- Open export contracts: IHO B-12 CSB (CSV/GeoJSON), CF-NetCDF, Cloud-Optimized GeoTIFF (COG), and `.pmtiles` vector tile layer definitions.
- Sanitized synthetic sample datasets (`docs/samples/`) for external developer and GIS testing.

### Deferred to Phase 2 & Phase 3
- Autonomous surface survey buoy CAD/PCB/firmware and RS-485 tether wire frame specifications.
- Diver hand/wrist receiver hardware, Bar30/BNO085 sensor fusion firmware, and `opendepthmap.local` onboard captive portal endpoints.

### Explicit Non-Goals
- Replacing certified official NOAA, UKHO, or IHO Electronic Navigational Charts (ENCs) for commercial or primary marine navigation.
- Serving as a primary dive decompression computer or life-support telemetry monitor.
- Redistributing proprietary vendor SDKs or exposing closed-source cloud infrastructure inside public protocol repositories.
- Requiring paid commercial APIs or closed licenses to read, validate, or consume OpenDepthMap open datasets.
