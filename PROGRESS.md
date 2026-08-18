# Progress

Where things currently stand. For the architecture, approach, and roadmap behind
these decisions, see `BUILD.md`.

## Module log

| Module | nbdev notebook | Exports | Status | Notes |
|---|---|---|---|---|
| core | `nbs/00_core.ipynb` | `UnprojectedCRSError`, `load_bathymetry` | Done | Raster loader enforces a projected CRS -- rejects both geographic *and* missing CRS. The missing-CRS case was a bug in the first draft (only geographic CRSs were rejected, so a CRS-less raster silently passed through); fixed, with a regression test. Nodata converted to NaN via `masked=True`. Tested against synthetic rasters (projected / geographic / no-CRS cases) plus a smoke test against real Bantry Bay tile data. |
| kernel / focal_mean | -- | -- | Next | Disc and annulus focal-mean kernels -- shared machinery TPI, BPI, and later LMI/CI all sit on top of. |
| standardize | -- | -- | Next | Global and local (windowed) z-score -- needed before any cross-method thresholding or voting; every terrain statistic (TPI/BPI in metres, Openness in degrees, LMI, CI, slope, curvature) is on a different scale. |
| tpi / bpi | -- | -- | Planned | `TPI = Z - focal_mean(Z, kernel)`; BPI is the same with an annulus kernel instead of a disc. First algorithm to take all the way through derive -> standardise -> threshold -> polygonize, to prove the pipeline shape before adding more. |
| openness | -- | -- | Planned | Yokoyama, Sirasawa & Pike (2002) 8-direction ray cast. |
| geomorphons / wood | -- | -- | Planned | Jasiewicz & Stepinski (2013) ternary-pattern archetypes; Wood (1996) curvature-based 6-class landform elements; Shape Index & Curvedness (Koenderink & van Doorn 1992) as the cross-cutting primitive underneath most of the Highs/Lows sub-type table. |
| lmi_ci | -- | -- | Planned | TPI+LMI (Highs) / TPI+CI (Lows) boundary-refinement hybrids. |
| shape / topographic / profile attributes | -- | -- | Planned | Per-polygon morphometrics -- see `~/geomorphology_algorithm_reference.md` §5-6 for formulas and the full per-term discriminator tables (all 40 Part 1 terms). |
| classify | -- | -- | Planned | Fuzzy/weight-of-evidence rule cascade assigning each polygon its Part 1 Morphology term, with confidence falling out of the same thresholds. |
| reconciliation (multi-method voting) | -- | -- | Parked | Agreement-based confidence across TPI/BPI/Openness/LMI-CI candidates. Deliberately deferred until the discrete detection functions above all exist -- see `BUILD.md` and `~/geomorphology_algorithm_reference.md` §2.5. |
| Part 2 engine | -- | -- | Not started | Setting/Process -> BGU -> BGU-T -> BGU-sT, building on the Part 1 output. |

## Data

Real 5m-resolution Bantry Bay bathymetry (EPSG:32629), tiled into 7 manageable,
non-empty pieces at `~/Documents/Data/BantryBay_5m_tiles/` (an 8th tile was 100%
nodata and was deleted). Used for smoke tests until Bass Strait data is available.

## Next up

`kernel`/`focal_mean` (disc and annulus) plus global/local standardisation --
everything else in the Map stage depends on these existing first.
