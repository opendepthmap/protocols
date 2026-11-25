# OpenDepthMap Public Intent & Problem Statement

- **Version:** `1.0.0` (Phase 1 Core System Baseline)
- **License:** CC0-1.0 Universal (Public Domain Dedication)
- **Scope:** Public mission, user personas, human-factors safety principles, and Phase 1 open contract scope for the OpenDepthMap bathymetric mapping ecosystem.

> **Non-Navigational Notice:** Supplemental bathymetric reference only. Not certified for primary marine navigation; always consult official NOAA/IHO nautical charts.

---

## 1. Why OpenDepthMap Exists

OpenDepthMap is an open-source, vendor-neutral bathymetric mapping ecosystem and subsea GIS data standard organized around three core pillars that address six persistent gaps in underwater mapping:

### Pillar 1: Nearshore White-Ribbon Bathymetry
1. **Closing the Nearshore "White Ribbon" Survey Gap:** Official hydrographic survey ships draw too much draft to safely survey shallow rocky coastlines, intertidal reefs, and kelp beds (under 10 to 15 meters depth), while satellite-derived bathymetry (SDB) fails in turbid, dark, or optically complex water. Surface micro-buoys and diver-towed sensors naturally fill this hydrographic gap.
2. **Temporal Change Detection After Storms and Dredging:** Mobile sandbars, river mouths, and harbor approaches shift rapidly after storms and dredging operations. Open, repeatable survey logs with consistent datum reduction enable localized bathymetric differencing over time.
3. **Uncharted Inland Lakes, Quarries, and Reservoirs:** Millions of small inland water bodies lack official bathymetric charts. Community mapping with explicit freshwater sound-velocity and datum handling unlocks usable depth references for local divers, limnologists, and watershed managers.

### Pillar 2: Vendor-Neutral Sonar Ingestion
4. **Rescuing Trapped Sonar Data from Proprietary Silos:** Recreational boaters and anglers generate vast quantities of single-beam sonar logs locked inside closed vendor formats. OpenDepthMap launches with core open, vendor-neutral formats (`GPX`, `GEOJSON`, `CSV`, `NMEA_0183`), explicit vertical datum labeling, and privacy-by-default controls, exporting cleanly into open GIS standards (`GeoParquet 1.1`, `OGC API - Features`, `IHO B-12 CSB`, `PMTiles`), with extended/proprietary parsers and gridded raster exports documented in [phase-1.5-roadmap.md](phase-1.5-roadmap.md).

### Pillar 3: Low-Cost Subsea Situational Awareness
5. **Financial and Navigational Barriers:** Commercial hydrographic survey rigs and Ultra-Short Baseline (USBL) positioning systems cost tens of thousands of dollars, placing precise bathymetric logging out of reach for individual divers, citizen scientists, and local search-and-recovery groups. Analog magnetic compasses are also disrupted by cross-currents, wrecks, reinforced concrete, steel cylinders, and low visibility; OpenDepthMap brings surface-tethered positioning and bathymetric logging within standard DIY maker budgets.
6. **Subsea Ecological Ground-Truthing and Hazard Mapping:** Raw depth soundings gain lasting scientific and practical value when paired with diver-verified waypoints, substrate classifications (sand, silt, bedrock, boulder), and hazard markers (expanding to CMECS/Darwin Core taxonomy and EXIF-stripped geo-referenced media in Phase 1.5).

---

## 2. Core User Personas & Workflow Outcomes

### Persona 1: The Recreational Scuba Diver & Forager
- **Needs:** High-contrast dark-water bathymetric map visuals, trip and route planning with leg distances and magnetic headings, planned-vs-actual track drift review, and strict privacy protection for sensitive reefs, ledges, and shoreline entry ramps.
- **Phase 1 Outcome:** Zero-login dark-water map exploration, private-by-default vault, user-controlled entry/exit trim sliders, interactive polygon redaction before public sharing, universal water route & survey transect planning, and downloadable offline `.pmtiles` regional chart packs.

### Persona 2: The Coastal Boater, Angler & Small-Craft Operator
- **Needs:** Explicit vertical datum labeling (never conflating unreduced transducer depth with tide-reduced MLLW), shoal and channel depth clarity, and frictionless upload of chartplotter sonar logs (`GPX`, `GEOJSON`, `CSV`, `NMEA_0183`).
- **Phase 1 Outcome:** Direct upload of open sonar logs paired with one-time vessel/transducer offset registration, automated NOAA CO-OPS MLLW tide reduction for US coastal waters, and real-time comparison between raw crowd soundings and tide-reduced MLLW depths.

### Persona 3: The DIY Hardware Maker & Embedded Tinkerer
- **Needs:** Vendor-neutral device registration, open upload contracts, and standardized metadata schemas for custom transducers, static draft offsets, and GNSS-to-transducer lever-arm geometry.
- **Phase 1 Outcome:** Write-only vessel/device API keys (`odm_dev_`), effective-dated sensor rig geometry revisions, and documented external upload contracts ready for Phase 2 survey buoys and Phase 3 diver wrist receivers.

### Persona 4: The GIS / Spatial Data Engineer & Marine Scientist
- **Needs:** Strict CRS/EPSG compliance (`OGC:CRS84` / `EPSG:4326`), transparent QARTOD QC flags, UNESCO sound-velocity calibration, 95% confidence Total Vertical/Horizontal Uncertainty (TVU/THU), and reproducible open-format exports.
- **Phase 1 Outcome:** Direct GeoParquet 1.1 archive access, OGC API - Features endpoints, and IHO B-12 CSB public domain datasets with honest calibration provenance.

---

## 3. Human-Factors, Trust & Safety Requirements

Every client, service, and dataset conforming to OpenDepthMap protocols MUST uphold these invariants:

1. **Supplemental Navigation Boundary:** OpenDepthMap outputs are supplemental bathymetric references only and are never certified for primary marine navigation or life-safety dive decompression planning. All map interfaces, OGC API collections, and exported manifests MUST carry the non-navigational disclaimer.
2. **Vertical Datum Truthfulness:** Unreduced raw transducer depths MUST NEVER be labeled or mixed silently with tide-reduced chart datums (`MLLW`, `IGLD`, `LOCAL_LAKE`, `MANUAL_DATUM`). Every sounding, vector tile feature, and export layer MUST carry an explicit `vertical_datum` attribute.
3. **Immutable Raw Provenance:** Raw uncalibrated soundings are preserved separately from derived calibrated/reduced products so scientific corrections (sound velocity, motion compensation, tide reduction) can be audited and reproduced without losing original measurements.
4. **Privacy-by-Default:**
   - All contributor uploads default to private (`is_public = false`).
   - Prior to public exposure, contributors control entry/exit privacy via interactive start/end trim sliders (erasing any chosen distance from the start or end of a track so short shore dives are never unintentionally destroyed) and custom spatial redaction polygons.
5. **Strict Credential Separation:** Personal User API Keys (`read:public`, `read:private_vault`, `write:tracks`, `write:waypoints`) are strictly separated from Device/Vessel API Keys (`write:tracks` only) so a lost or compromised field sensor cannot read private user vaults.
6. **Transparent Data Provenance & Attribution:** External public baselines (such as NOAA BlueTopo, NOAA DCDB / IHO CSB, GEBCO, USGS 3DEP, and public state inland lake bathymetry datasets like NH GRANIT / NHDES) are tagged distinctly from user-submitted tracks. Base map land/shoreline vectors derived from OpenStreetMap strictly comply with ODbL 1.0 attribution (*"Map data © OpenStreetMap contributors"*) and remain in clean layers separate from CC0-1.0 bathymetric soundings.

---

## 4. Phased Scope & Explicit Non-Goals

### In Scope for Phase 1 (Day-1 GA)
- Public external REST API ([openapi-v1.yaml](../schemas/openapi-v1.yaml)), Google + GitHub browser sessions, strictly separated Personal (`odm_usr_`) vs. write-only Device (`odm_dev_`) API keys, and OGC API - Features contracts.
- Two-step direct upload contract (`GPX`, `GEOJSON`, `CSV`, `NMEA_0183`) and asynchronous ingestion job lifecycle.
- GeoParquet 1.1 full-resolution sounding schema, QARTOD QC flags, user-controlled entry/exit trim sliders + polygon redaction, simple waypoints, universal route planner, IHO B-12 CSB (CSV/GeoJSON), and `.pmtiles` vector tile layer definitions.
- Sanitized synthetic sample datasets (`docs/samples/`) for external developer and GIS testing.

### Planned Post-Day-1 Extensions
- **Phase 1.5:** See [phase-1.5-roadmap.md](phase-1.5-roadmap.md) and [openapi-v1.5-draft.yaml](../schemas/openapi-v1.5-draft.yaml).
- **Phase 2 & Phase 3:** Autonomous surface survey buoy CAD/PCB/firmware, RS-485 tether wire frame specs, and diver hand/wrist receiver hardware + `opendepthmap.local` captive portal.

### Explicit Non-Goals
- Replacing certified official NOAA, UKHO, or IHO Electronic Navigational Charts (ENCs) for commercial or primary marine navigation.
- Serving as a primary dive decompression computer or life-support telemetry monitor.
- Redistributing proprietary vendor SDKs or exposing closed-source cloud infrastructure inside public protocol repositories.
- Requiring paid commercial APIs or closed licenses to read, validate, or consume OpenDepthMap open datasets.
