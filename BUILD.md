# Build

Architecture, scope, and the reasoning behind the decisions in this project --
`PROGRESS.md` tracks what's actually built; this is why it's built that way.

## What this is, and what it isn't

A self-implemented port of Dove et al.'s Part 1 Morphology glossary and Nanson et
al.'s Part 2 Geomorphology framework (the two-part MIM-GA seabed geomorphology
classification scheme), for a Bass Strait PhD project. Built from first principles
on `numpy`/`scipy`/`xarray`/`rioxarray`/`geopandas`/`shapely`/`rasterio`/
`networkx`/`scikit-image` (plus `hvplot`/`holoviews` for visualisation) --
deliberately **not** using WhiteboxTools, RichDEM, GRASS, or SAGA, since the point
is to understand and own every algorithm, not call into a black box that does the
terrain-analysis thinking for you.

`scikit-image` and `networkx` are treated as general-purpose array/graph
infrastructure -- the same tier as `scipy` -- not domain-specific geomorphometry
toolkits. Using `skimage.filters.threshold_multiotsu` is the same kind of move as
using `scipy.ndimage.label`. WhiteboxTools/GRASS/SAGA are excluded specifically
because they'd do the geomorphology *thinking*, not just provide array plumbing.

This is a **separate, fresh build** from `../geomorphology_tools` (a direct,
reverse-engineered port of Geoscience Australia's GA-SaMMT ArcGIS toolbox). The two
are not meant to merge or share code.

## Classification philosophy: fuzzy / weight-of-evidence, not hard thresholds

Both Part 1 feature-type assignment and Part 2 Setting/BGU classification use fuzzy
membership functions (sigmoid/triangular, centred on a threshold with a tunable
transition width) rather than crisp boolean rule cascades. This keeps the rules
fully interpretable and needs no training data (unlike a learned classifier), while
degrading gracefully near class boundaries instead of flipping discretely -- closer
to how a geomorphologist actually judges an ambiguous feature. Confidence scores
fall out of the same thresholds, not a separate rule-count.

This isn't just a preference -- it's what Dove et al. (2020) themselves argue for.
The Part 1 glossary deliberately avoids quantitative thresholds throughout, stating
that "approaches based on strict threshold rules often result in inaccurate
delineation of features... particularly for complex, aggregate, and superimposed
features (which are common)" (citing Sowers et al. 2020), while acknowledging the
trade-off: avoiding thresholds "retains an element of subjectivity, and that is a
limitation." Fuzzy membership is this project's answer to that stated trade-off --
keep the same descriptive, non-arbitrary spirit the scheme's authors intended, but
make the subjectivity a calibratable, auditable number (a membership degree and a
confidence score) instead of an unstated judgement call. See
`~/geomorphology_algorithm_reference.md` §6.6 for the full quote in context.

Reinforcement learning was considered and set aside for the core classification
step: it's a single-step "attributes -> label" problem, not a sequential decision
process, so RL would just be a less sample-efficient way of doing supervised
learning -- and the reward signal RL needs is the same missing ingredient
supervised learning needs (labelled ground truth, which doesn't exist yet for Bass
Strait). The one place an RL-adjacent idea might genuinely help: calibrating the
confidence threshold for auto-classify vs. flag-for-expert-review via a contextual
bandit, learning from expert corrections over time. Not implemented, just noted as
a plausible future direction.

## Reconciling multiple detection methods -- parked

TPI, BPI, Openness, and TPI+LMI/CI all target the same Highs/Lows but won't agree
exactly at boundaries. The planned approach: stack each (standardised) method's
output on a common grid, count per-cell agreement, threshold that count to define
the merged polygon boundary, and use the mean agreement across the polygon as a
**detection confidence** -- reported separately from the classification confidence
Part 2 produces. STAPLE (Warfield, Zou & Wells 2004) is the more rigorous
alternative (learns per-method reliability via Expectation-Maximization) but needs
calibration data that doesn't exist yet. Full write-up, worked example, and
per-term scope (where this does and doesn't apply -- it doesn't apply to
Plane/Slope/Escarpment or Centreline/Break-in-slope, for instance) in
`~/geomorphology_algorithm_reference.md` §2.5.

**Deliberately deferred** until the discrete per-method detection functions
(terrain derivatives, TPI/BPI, Openness, LMI/CI) all exist -- see `PROGRESS.md`.

## Build order

Bottom-up, following the dependency chain:

1. Raster I/O + basic scaffolding (`load_bathymetry`) -- done.
2. `kernel`/`focal_mean` (disc + annulus) + standardisation (global/local z-score)
   -- shared primitives everything else sits on.
3. TPI -> BPI -- cheapest real algorithm, taken all the way through
   derive -> standardise -> threshold -> polygonize once, to prove the pipeline
   shape before adding more algorithms on top.
4. Openness, then Geomorphons/Wood's classification -- structurally independent
   of TPI/BPI, slot in once the pipeline shape is proven.
5. LMI/CI hybrid boundary refinement -- depends on standardisation (2) and a
   working polygonize step (3).
6. Attributes, classification, lineament extraction -- depend on having final
   polygons in hand, so wait until 1-5 are done.
7. Part 2 (Setting/Process -> BGU -> BGU-T -> BGU-sT) -- builds on the full Part 1
   output.

## Algorithm reference

Formulas, citations, and per-term discriminators (all 40 Part 1 Morphology terms,
every pipeline stage, Part 2 attributes) live in
`~/geomorphology_algorithm_reference.md`, kept outside this repo since it's a
standalone research document, not project code. Read it before implementing a new
module -- it's organised by pipeline stage to match `PROGRESS.md`'s module log.

## Workflow

Each module: an nbdev notebook with `#| export` cells, inline `fastcore.test` unit
tests against synthetic data, plus a smoke test against real Bantry Bay data, a run
through `uv run nbdev-test`, a `PROGRESS.md` status update, and its own commit.
