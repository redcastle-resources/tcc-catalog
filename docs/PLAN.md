# Plan: Publish NLCD Tree Canopy Cover (TCC) v2025-6 to Source Cooperative with Portolan

## Context
This repo holds docs and scripts for publishing USFS NLCD Tree Canopy Cover to [Source Cooperative](https://source.coop) as a Portolan catalog. The workflow follows the skills in [portolan-sdi/portolan-skills](https://github.com/portolan-sdi/portolan-skills) (cloned locally into `portolan-skills/`, gitignored): `portolan-bootstrap`, `sourcecoop`, `portolan-thumbnails`, `catalog-grader`, `reading-portolan`, `register-catalog`.

## Decisions so far
| Item | Value |
|---|---|
| Source service | https://imagery.geoplatform.gov/iipp/rest/services/Vegetation/USFS_EDW_NLCD_TCC_CONUS/ImageServer |
| Scope | CONUS NLCD only (41 annual rasters, 1985-2025) |
| Destination | Source Coop org `redcastle-resources`, repo slug `tcc-v2025-6` (catalog title "TCC v2025-6") |
| Remote | `s3://redcastle-resources/tcc-v2025-6/` via `https://data.source.coop` |
| License | CC0-1.0 (user-chosen; https://creativecommons.org/publicdomain/zero/1.0/). The service itself states no license, so record this as a maintainer decision, not an attested claim |
| Maintainer | Lila Leatherman, lst.leatherman@gmail.com |
| Logo | Red Castle logo goes on the Source Coop org page only, not in the catalog |
| Catalog style | Bucket-based, via `portolan-bootstrap` (not git-backed) |
| Acquisition | **OPEN** - user is investigating data access options (see below) |

## Findings from the ImageServer (2026-11-04)
- Mosaic dataset, 1 band, `U8`, 30 m, native service SR EPSG:3857 (WKID 102100), CONUS extent.
- Catalog items named `nlcd_tcc_conus_wgs84_v2025_6_<YYYY>0101_<YYYY>1231`, `beginyear`/`endyear` fields, 1985 onward (query the `query` endpoint for the full list).
- Pixel values: 0-100 percent canopy; 254 = non-processing area; 255 = background. (Service description.)
- Description says NLCD data are masked for non-treed features (water, non-tree crops) and smoothed to reduce interannual noise.
- Download and methods docs are linked from the service description: https://data.fs.usda.gov/geodata/rastergateway/treecanopycover/
- Not cloud-native files: the service renders images on request (`maxImageWidth` 100000, `maxDownloadImageCount` 20), so pulling 41 full-resolution CONUS years through `exportImage` is impractical and risks resampling artifacts.
- Discrepancy to resolve: part of the service's item description still says "v2023-5 ... 1985 through 2023" while the main description says v2025-6 / 1985-2025. Confirm against the Raster Gateway before writing any version claim.
- `portolan extract` supports ArcGIS feature services, WFS and Carto, not ImageServer, so acquisition needs a scripted download.
- Local tools not yet installed: `uv`, `portolan`, `source-coop`, `rashid`, `gdalinfo` (Node 24 and git are present).

## Open question: data access
Options being investigated:
1. USFS Raster Gateway downloads (recommended by the service's own description) - verify file URLs, formats, sizes, CRS, and use statement.
2. ImageServer `exportImage` tiling - slow, limited, possible resampling.
3. Other mirrors (e.g. MRLC / other hosted copies) - to be checked.

Whichever is chosen, the mirror-path probe (below) decides metadata-only vs full mirror.

## Steps (mapped to skill checkpoints)
1. **Setup tools.** Install `uv`, then `portolan-cli`, `source-coop` (0.3.0+), `rashid`, GDAL; install the Portolan Claude Code plugin. Verify with `--version`. Thumbnails need Node 24.12+ (present). On Windows run the push from Git Bash or WSL (POSIX `eval` credential export).
2. **Research checkpoint.** Per `portolan-bootstrap`: read portal pages, data dictionary, methods docs, producer/program, licensing terms. Tag every claim attested / cited / derived. Producer = USDA Forest Service; host = Red Castle Resources (a mirror, so `via` link, `source_url`, `updated` are required). Log undecodable values in `known_issues`.
3. **Mirror-path probe.** Run `portolan-skills/skills/portolan-bootstrap/scripts/probe-upstream.sh` on representative assets (Range, CORS, size); `gdalinfo /vsicurl/...` for tiling, overviews, `LAYOUT=COG`. Present size/time/cost before downloading.
4. **Layout.** One collection `tcc-conus-nlcd` (name TBD) with one item directory per year (`<year>/<year>.json` + `<year>.tif`) and `items.parquet` via `--stac-geoparquet`. Keep native CRS in the COGs.
5. **Acquire + convert.** Scripted download into a gitignored staging dir; convert to COG if not already; `portolan scan --suggest-collections`, `portolan check --fix --dry-run`, then `--fix`. Helper scripts live in `scripts/` and are logged in the collection `AGENTS.md`.
6. **Init + metadata.** `portolan init --title "TCC v2025-6" --auto --license CC0-1.0`; `portolan metadata init`; fill `contact`, `license`, `providers`, `source_url`, `attribution`, `processing_notes`, `known_issues`; `portolan add . --stac-geoparquet`; `portolan readme`.
7. **Docs.** Replace all `AGENTS.md` stubs with real content: CRS, value meanings (0-100, 254, 255), recipes that were actually run (GDAL/rasterio/DuckDB). Markdown-link every URL; human-facing links use `source.coop`.
8. **Styles (checkpoint).** Raster canopy ramp; check what the browser can render and note legend limits; get approval before generating anything.
9. **Thumbnails + QC.** `portolan-thumbnails`; open the catalog in the Portolan browser; verify bbox, default style, links.
10. **Validate.** `portolan metadata validate`, `portolan check`, `catalog-grader` on the local catalog.
11. **Auth + dry-run push (checkpoint).** `source-coop login`; `.env` (gitignored) with `PORTOLAN_REMOTE=s3://redcastle-resources/tcc-v2025-6/` and `PORTOLAN_S3_ENDPOINT=https://data.source.coop`; export creds with `eval "$(source-coop creds --format env)"`; `portolan push --dry-run`; get explicit approval.
12. **Push + verify.** `portolan push --verbose`; `rashid check --live --live-base-url https://data.source.coop/redcastle-resources/tcc-v2025-6/ .`; open `https://browser.portolan-sdi.org/#/external/data.source.coop/redcastle-resources/tcc-v2025-6/catalog.json`; re-run `catalog-grader` on the published URL; run documented queries via `reading-portolan`.
13. **Register + maintain.** Optional `register-catalog`. Add a refresh runbook (re-acquire, `portolan add . --force`, `portolan readme`, push). Upload the logo to the org page manually.

## Risks
- Credentials expire; re-run the `eval` line on 403. Do not set `PORTOLAN_PROFILE`.
- Multi-scene rasters without item directories are a known defect; lay out items before `add`.
- CONUS 30 m x 41 years is large; estimate before downloading and confirm storage/transfer.
- License, title, and contact are always user-confirmed; do not state unverified claims.

## Verification
- `portolan check` and `rashid check --live` clean; `catalog-grader` grades reported at the publish checkpoint.
- Catalog loads in the Portolan browser with visible data and thumbnails.
- Every query in `AGENTS.md` runs against the published URLs.
- `aws s3 ls s3://redcastle-resources/tcc-v2025-6/ --profile source-coop --recursive` matches the local file list.
