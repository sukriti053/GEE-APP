# GEE-APP
Flood &amp; Waterlogging Risk Viewer --This tool estimates flood and waterlogging risk for villages across India by combining terrain, drainage, and land-cover data into a single 0–1 risk score, then lets you drill down from state to district to tehsil to village to inspect that score and the underlying map layers for any specific location.
Flood & Waterlogging Risk Viewer
Google Earth Engine Apps Script — Technical & User Documentation
1. What This App Does
This is an interactive Google Earth Engine (GEE) App for estimating flood and waterlogging risk for villages across India. It combines terrain, hydrology, and land-cover datasets into a single composite risk score for every 30m pixel nationwide, then lets a user drill down through administrative boundaries — state, district, tehsil, and finally village — to inspect that score and the surrounding map layers for one specific location at a time.
The app is designed around a core constraint of Earth Engine: computing or rendering the full national-scale risk raster at once reliably triggers "User memory limit exceeded" errors. Every part of the design — from how normalization is done to how the risk layer is clipped before any reduceRegion() call — exists to keep every computation scoped to a small local area (the selected village plus a buffer) rather than the whole country.
2. Core Workflow
•	The user selects a State from a dropdown, populated from the state boundary layer.
•	Selecting a state loads and filters the District layer down to districts inside that state.
•	Selecting a district loads Tehsils within that district (or villages directly, if no tehsil layer is available).
•	Selecting a tehsil loads Villages within that tehsil.
•	Selecting a village centers the map on that village, draws the risk layer and supporting map layers clipped to a 2 km buffer around it, and reports the village's mean risk index.
•	At the district/tehsil level, a ranked table also lists the ten highest-risk villages in the current selection, sorted by mean risk index.
3. Data Inputs and How Each Layer Is Used
The app loads a fixed set of GEE image and feature-collection assets (configured at the top of the script in the ASSETS object). The table below explains what each one is and the specific role it plays in the app.
Layer	Type	Role in the app
Slope	Raster (Image)	Normalized 0–1 (slope ÷ 30, clamped) and weighted 25% of the risk index. Flatter terrain drains poorly and scores higher.
Catchment area	Raster (Image)	Normalized 0–1 via a fixed unit scale (0–500) and weighted 20%. Larger upstream contributing area means more runoff can converge on a pixel.
Stream order	Raster (Image)	Normalized 0–1 (order 1–7) and used to weight how much a nearby stream contributes to risk — larger, higher-order streams contribute more.
Drainage line	Vector (FeatureCollection)	Rasterized to a 0/1 mask, then converted to a distance-to-drainage layer. Proximity to a drainage line increases risk, combined with stream order (30% total weight).
Natural depressions	Raster (Image)	Any pixel flagged as a depression (value > 0) contributes a flat risk boost, weighted 25% — low-lying spots that visibly pool water.
River / Canal	Vector (FeatureCollection)	Displayed as a map overlay for context around the selected village; not directly part of the numeric risk formula.
Reservoir	Vector (FeatureCollection)	Displayed as a map overlay for context; shows nearby water storage that may affect local flood behavior.
Microwatershed boundaries (MWS)	Vector (FeatureCollection)	Displayed as a map overlay showing the local catchment/watershed unit the selected village sits within.
LULC (2018–2025, one image per year)	Raster (Image, per year)	Used to identify built-up pixels (a configurable class code) that overlap High/Very High risk zones — the basis for exposure analysis (see exposureForYear() in the code), which flags built-up area exposed to flood risk in a given year.
Villages	Vector (FeatureCollection)	The finest administrative unit in the drill-down; also the unit the ranked risk table and the per-village mean risk score are computed over.
State / District / Tehsil	Vector (FeatureCollections)	Administrative boundaries used purely for the drill-down UI — each level's dropdown is populated by spatially filtering the child layer against the selected parent's geometry.
4. The Composite Risk Index — Detailed Calculation
The final risk_index is a weighted sum of four normalized 0–1 sub-scores. Each sub-score is computed from a raw input layer, normalized onto a 0–1 scale using a fixed formula (so no national-scale statistics need to be computed), then clamped to stay within [0, 1]. The exact formulas used in the script are shown below.
4.1 Slope Risk — 25% weight
slopeRisk = clamp( slope ÷ 30 , 0, 1 )
The raw slope layer (in % slope) is divided by a fixed reference value of 30. A pixel at 0% slope scores 0 (no slope-driven risk); a pixel at 30% slope or steeper scores the maximum of 1. Flatter terrain drains poorly, so lower slope actually corresponds to higher waterlogging risk in the overall index once combined below — this sub-score on its own simply measures how flat the land is, normalized 0–1.
4.2 Catchment Risk — 20% weight
catchRisk = clamp( unitScale(catchment, 0, 500) , 0, 1 )
The raw catchment/flow-accumulation layer is rescaled so that 0 maps to 0 and 500 (a fixed reference value) maps to 1, then clamped. A larger contributing upstream area means more runoff can converge on that pixel during a storm, so a larger catchment value produces a higher risk score.
4.3 Stream Risk — 30% weight
This sub-score combines two parts: how close a pixel is to a mapped drainage line, and how large that stream is (its stream order).
distance = fastDistanceTransform(drainageRaster) × 30  (meters)
proximityRisk = clamp( 1 − unitScale(distance, 0, 2000) , 0, 1 )
orderNorm = clamp( unitScale(streamOrder, 1, 7) , 0, 1 )
streamRisk = clamp( proximityRisk × (0.5 + 0.5 × orderNorm) , 0, 1 )
Distance to the nearest drainage line is computed in meters, then rescaled so 0m maps to maximum proximity risk (1) and 2000m or farther maps to 0. That proximity score is then scaled up or down depending on the stream's order (1 = smallest headwater stream, 7 = largest river): a pixel near a low-order stream keeps only 50–75% of its proximity risk, while a pixel near a high-order (larger) stream keeps up to 100%.
4.4 Depression Risk — 25% weight
depressionRisk = ( depressions > 0 ) ? 1 : 0
This is a binary flag rather than a continuous scale: any pixel identified in the natural depressions layer (value greater than 0) is treated as a low-lying spot where water is expected to pool, and receives the full weight of this component.
4.5 Combining the Four Components
risk_index = clamp( 0.25×slopeRisk + 0.20×catchRisk + 0.30×streamRisk + 0.25×depressionRisk , 0, 1 )
Every one of the four terms above is explicitly .unmask()'d (missing/no-data pixels are treated as 0) before this sum is computed, and the result itself is unmasked and clamped again. This matters because a single missing value in any one input — for example a pixel just outside the reach of the drainage distance kernel, or with no recorded stream order — would otherwise silently null out the entire composite score for that pixel, which is what previously caused some villages to report a mean risk index of N/A.
Weight summary:
•	Slope — 25%
•	Catchment area — 20%
•	Stream proximity × stream order — 30%
•	Natural depressions — 25%
(Total: 100%, so risk_index is always a weighted average bounded between 0 and 1.)
4.6 Risk Classification
The continuous 0–1 risk_index is grouped into four bands for display and for the ranked village table:
•	Low — risk_index < 0.25
•	Medium — 0.25 ≤ risk_index < 0.5
•	High — 0.5 ≤ risk_index < 0.75
•	Very High — risk_index ≥ 0.75
5. Map Layers and Legend (per selected village)
When a village is selected, the following layers are drawn on the map, all clipped to a 2 km buffer around the village:
Map layer	Style	What it shows
Risk Class	Colored fill: green / yellow / orange / red	The classified 0–1 risk score — Low, Medium, High, Very High.
Drainage Lines	Blue line, 3px	Mapped drainage channels used in the stream-proximity part of the risk score.
River / Canal	Teal/cyan line, 3px	Rivers and canals near the village, for context.
Reservoir	Purple line, 3px	Nearby water storage bodies, for context.
Microwatershed Boundary	Bright yellow line, 4px	Outline of the local catchment unit the village belongs to.
Village Boundary	Black line, 3px	The exact outline of the selected village.
Colors were deliberately spread across distinct hues (rather than several shades of the same blue, as in an earlier version) so every layer is distinguishable at a glance, and line widths were increased so no layer is lost against the satellite basemap.
6. Left Control Panel
•	Title and short description of the tool.
•	Four cascading dropdowns: State → District → Tehsil → Village.
•	A results label showing the selected village's name and mean risk index.
•	A ranked list of the ten highest-risk villages in the current district/tehsil selection.
6.1 Right-Side Documentation Panel
A dedicated documentation panel sits on the right side of the screen, separate from the left control panel, so the left stays focused purely on the state/district/tehsil/village selectors.
This panel is collapsed by default. A small, always-visible toggle button — labeled "How App is Used »" — sits between the map and the documentation panel. Clicking it expands the panel and relabels the button "« Hide"; clicking again collapses it. Because the button lives in its own small panel rather than inside the documentation panel itself, it stays reachable at all times regardless of whether the documentation is currently shown.
The documentation panel explains, in plain language:
•	What the tool does and the overall drill-down workflow.
•	How the risk score is built, including the weight of each component.
•	What each map layer shown for a selected village represents.
Each explanatory point is rendered as its own individually-wrapped label (rather than one block of text with manual line breaks), so long lines wrap correctly inside the panel instead of being cut off — an issue in an earlier version where text using a non-wrapping style caused some words to be clipped from view.
7. Known Performance Considerations
Because Earth Engine computes everything lazily and on-demand, several design choices exist purely to prevent "User memory limit exceeded" errors at national scale:
•	Normalization of slope and catchment area uses fixed constants rather than data-driven percentiles computed over the whole country.
•	No manual .reproject() calls — Earth Engine reconciles differing projections automatically, and only over the geometry actually being queried.
•	Every reduceRegion() / reduceRegions() call is scoped to a small buffered geometry around the current selection, uses bestEffort: true, and a raised tileScale (4–8) to trade compute time for lower memory use.
•	The debug visualization of the full national risk_index layer is kept commented out in the source, since even rendering it nationally is enough to trigger a memory error.
8. Configuration Notes for Future Edits
•	All asset paths live in the ASSETS object at the top of the script — update these if any dataset moves or is re-exported.
•	BUILTUP_LULC_CODE controls which LULC class is treated as "built-up" for exposure analysis — confirm this matches your specific LULC classification scheme.
•	LAYER_COLORS and LAYER_WIDTHS are the single source of truth for every map layer's color and line thickness — both the map rendering code and the legend read from these two objects, so changing a color or width only requires editing one place.
•	FIELD_NAMES maps each administrative level to the actual property name in its FeatureCollection (e.g. district name field, tehsil name field) — update if your boundary assets use different attribute names.
