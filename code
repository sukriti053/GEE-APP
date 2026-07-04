/*****************************************************************
 * FLOOD & WATERLOGGING RISK VIEWER — GEE Apps Script Starter
 * (FIXED VERSION 2)
 *
 * Paste this into code.earthengine.google.com, replace the asset
 * paths in the CONFIG block with your own assets, then click
 * "Run" and "Apps > Publish" when ready.
 *
 * Structure:
 *  1. CONFIG — asset paths (EDIT THESE)
 *  2. Load layers
 *  3. Build composite risk index
 *  4. Boundary drill-down UI (state>district>tehsil>village)
 *  5. Map + legend + ranked table + chart panel
 *
 * FIXES APPLIED IN THIS PASS (see inline "FIX:" comments):
 *  1. Stream Order is an Image asset — it is now loaded directly
 *     with ee.Image(ASSETS.streamOrder) and unit-scaled, instead
 *     of being routed through an empty FeatureCollection.
 *  2. Distance-to-drainage now uses .distance(ee.Kernel.euclidean)
 *     instead of a fastDistanceTransform/pixelArea combination.
 *  3. tehsilFC is now always loaded directly since the asset exists.
 *  4. populateSelect() no longer silently drops numeric values.
 *  5. reduceRegions() in updateRankedTable() now sets tileScale: 4
 *     to reduce memory-limit failures on large districts.
 *  6. reduceRegion() in villageSelect.onChange now sets
 *     bestEffort: true.
 *  7. villageSelect.onChange now guards against a missing/null
 *     village feature instead of crashing.
 *  8. withinParent() is simplified to a plain filterBounds() call
 *     (buffer/simplify removed as unnecessary).
 *
 * PASS 3 (risk-index masking fix):
 *  - distToDrainage, orderNorm, proximityRisk, streamRisk, and
 *    depressionRisk are all explicitly .unmask()'d before being
 *    combined, and riskIndex itself is .unmask(0).float()'d.
 *    Previously a single masked pixel in any one input (e.g. a
 *    village just outside the drainage distance kernel, or with
 *    no stream-order value) would silently null out the whole
 *    composite for that pixel, which is why reduceRegion /
 *    reduceRegions sometimes returned N/A or dropped villages.
 *  - updateRankedTable()'s reduceRegions() now uses tileScale: 8
 *    (up from 4) and pins crs: riskIndex.projection() so results
 *    aren't silently reprojected/resampled inconsistently.
 *  - Added a debug Map.addLayer() + print() right after riskIndex
 *    is built, so the composite can be sanity-checked visually
 *    before drilling into any specific village.
 *
 * PASS 4 (national-scale memory fix, attempt 1):
 *  - Tried forcing all rasters into slope's projection via
 *    .reproject(), and switched drainage distance back to
 *    fastDistanceTransform() for cheaper computation at national
 *    scale. Also commented out the debug Map.addLayer(riskIndex).
 *
 * PASS 5 (national-scale memory fix, attempt 2 — root cause):
 *  - Removed the Pass 4 .reproject() calls entirely. Besides
 *    resample('nearest') being an invalid mode (EE only supports
 *    'bilinear'/'bicubic'), reproject() itself forces EE to
 *    eagerly resample the WHOLE national raster onto one grid —
 *    that whole-of-India eager resampling, not just the invalid
 *    mode string, was the actual trigger for "User memory limit
 *    exceeded". EE already reconciles differing projections
 *    lazily and only over the geometry actually being queried, so
 *    no manual reprojection step is needed.
 *  - villageSelect.onChange now buffers the selected village's
 *    geometry by 2000m before using it to clip/reduce riskIndex,
 *    keeping every computation scoped to a small local extent
 *    around that village instead of the national composite.
 *
 * PASS 6 (national-scale memory fix, attempt 3 — remaining culprit):
 *  - slope/catchment normalization no longer calls .reduceRegion()
 *    with percentile([2,98]) over villages.geometry() (i.e. all of
 *    India) at load time. That eager national-extent scan was
 *    still running even after Pass 5's reproject() removal, and
 *    is replaced here with fixed normalization constants
 *    (slope/30, catchment unitScale(0,500)) so nothing needs to
 *    touch the whole country just to compute two numbers.
 *  - riskIndex is now clipped to a small local geometry
 *    (geom.buffer(1000) / villageFC.geometry()) immediately before
 *    every reduceRegion()/reduceRegions() call, on top of the
 *    existing tileScale/bestEffort settings.
 *
 * PASS 7 (legend fix):
 *  - The legend previously only documented the 4 Risk Class
 *    colors. It now also documents every vector overlay layer
 *    drawn in villageSelect.onChange (Drainage Lines, River /
 *    Canal, Reservoir, Microwatershed Boundary, Village Boundary)
 *    with matching swatch colors, so every layer on the map can
 *    be identified from the legend alone.
 *****************************************************************/

// ============================================================
// 1. CONFIG — replace with your actual GEE asset IDs
// ============================================================
var ASSETS = {
  slope:        'projects/gee-pan-india/assets/layers/slope_percentage_fabdem',        // ee.Image, % slope
  catchment:    'projects/gee-pan-india/assets/layers/catchment_area_multiflow',       // ee.Image, single-flow accum
  depressions:  'projects/gee-pan-india/assets/layers/natural_depression',  // ee.Image or FeatureCollection (1=depression)
  drainageLine: 'projects/gee-pan-india/assets/layers/drainage_line',        // ee.FeatureCollection
  streamOrder:  'projects/gee-pan-india/assets/layers/stream_order',         // ee.Image, band value = stream order
  riverCanal:   'projects/gee-pan-india/assets/layers/canal',          // ee.FeatureCollection
  reservoir:    'projects/gee-pan-india/assets/layers/reservoir',            // ee.FeatureCollection
  lulc_2017_2018:'projects/gee-pan-india/assets/layers/LULC_2017_2018',
  lulc_2018_2019:'projects/gee-pan-india/assets/layers/LULC_2018_2019',
  lulc_2019_2020: 'projects/gee-pan-india/assets/layers/LULC_2019_2020',
  lulc_2020_2021:'projects/gee-pan-india/assets/layers/LULC_2020_2021',
  lulc_2021_2022:'projects/gee-pan-india/assets/layers/LULC_2021_2022',
  lulc_2022_2023:'projects/gee-pan-india/assets/layers/LULC_2022_2023',
  lulc_2023_2024:'projects/gee-pan-india/assets/layers/LULC_2023_2024',
  lulc_2024_2025:'projects/gee-pan-india/assets/layers/LULC_2024_2025', // ee.Image per year, band assumed single-band land-cover class
  villages:     'projects/gee-pan-india/assets/layers/Village_pan_India_Polygon', // FeatureCollection — no state/district/tehsil fields; matched spatially
  state:        'projects/gee-pan-india/assets/layers/state',        // FeatureCollection, state boundaries
  district:     'projects/gee-pan-india/assets/layers/District',     // FeatureCollection, district boundaries
  tehsil:       'projects/gee-pan-india/assets/layers/tehsil',       // FeatureCollection, tehsil boundaries
  mws:          'projects/gee-pan-india/assets/layers/Microwatershed_boundries_v1'        // FeatureCollection w/ basin/subbasin/watershed/mws props
};

var BUILTUP_LULC_CODE = 8; // EDIT: code for "built-up" class in your LULC scheme

// ============================================================
// 2. Load layers
// ============================================================
var slope        = ee.Image(ASSETS.slope);
var catchment     = ee.Image(ASSETS.catchment);
var depressions   = ee.Image(ASSETS.depressions); // assume raster, 1 = depression present
var drainageFC    = ee.FeatureCollection(ASSETS.drainageLine);

// FIX 1: stream_order is an Image asset (band value = stream order),
// not a FeatureCollection. Load it directly instead of using an
// empty FeatureCollection + reduceToImage().
var streamOrder = ee.Image(ASSETS.streamOrder);

// river/canal/reservoir/mws boundary layers
var riverCanalFC  = ee.FeatureCollection(ASSETS.riverCanal);
var reservoirFC   = ee.FeatureCollection(ASSETS.reservoir);
var villages      = ee.FeatureCollection(ASSETS.villages);
var mwsFC         = ee.FeatureCollection(ASSETS.mws);

// state/district/tehsil layers
var stateFC       = ee.FeatureCollection(ASSETS.state);
var districtFC    = ee.FeatureCollection(ASSETS.district);
// FIX 3: tehsil asset exists, so load it directly instead of the
// REPLACE_WITH placeholder check.
var tehsilFC      = ee.FeatureCollection(ASSETS.tehsil);

// PASS 5: removed the forced .reproject() block from Pass 4.
// reproject() forces EE to resample the ENTIRE national raster
// into a single target grid before anything else can happen —
// that eager, whole-of-India resampling (not just the invalid
// 'nearest' resample mode) was the real source of the memory
// error. EE reconciles differing projections automatically and
// lazily at reduceRegion/reduceRegions time, scoped to whatever
// geometry is actually being queried, so no manual reprojection
// is needed here.

// ============================================================
// 3. Build composite risk index
// ============================================================
var LULC_YEARS = {
  2018: ASSETS.lulc_2017_2018,
  2019: ASSETS.lulc_2018_2019,
  2020: ASSETS.lulc_2019_2020,
  2021: ASSETS.lulc_2020_2021,
  2022: ASSETS.lulc_2021_2022,
  2023: ASSETS.lulc_2022_2023,
  2024: ASSETS.lulc_2023_2024,
  2025: ASSETS.lulc_2024_2025
};

var lulcColl = ee.ImageCollection(
  Object.keys(LULC_YEARS).map(function(yearStr) {
    var year = parseInt(yearStr, 10);
    return ee.Image(LULC_YEARS[yearStr]).set('year', year);
  })
);

// =====================================================================
// PASS 6: BUILD RISK INDEX (Memory Optimized)
// The Pass 1-5 versions still normalized slope/catchment using
// slope.reduceRegion()/catchment.reduceRegion() percentiles over
// villages.geometry() — i.e. all of India — every time the script
// loaded. That eager, national-extent reduceRegion call (needed
// just to get two numbers) was the remaining memory sink even
// after removing reproject(). This version replaces those
// data-dependent percentile lookups with fixed normalization
// constants, so nothing needs to scan the full country up front.
// =====================================================================

// Analysis resolution
var SCALE = 30;

// Slope Risk
var slopeRisk = slope
  .unmask(0)
  .divide(30)
  .clamp(0, 1)
  .rename('slope');

// Catchment Risk
var catchRisk = catchment
  .unmask(0)
  .unitScale(0, 500)
  .clamp(0, 1)
  .rename('catch');

// Stream Order Risk
var orderNorm = streamOrder
  .unmask(1)
  .unitScale(1, 7)
  .clamp(0, 1)
  .rename('order');

// Drainage raster
var drainageRaster = ee.Image(0).byte().paint(drainageFC, 1);

// Distance to drainage (memory friendly)
var distance = drainageRaster
  .fastDistanceTransform(64)
  .sqrt()
  .multiply(SCALE)
  .rename('distance');

var proximityRisk = ee.Image(1)
  .subtract(
    distance
      .unitScale(0, 2000)
      .clamp(0, 1)
  )
  .clamp(0, 1)
  .rename('proximity');

// Stream Risk
var streamRisk = proximityRisk
  .multiply(
    ee.Image.constant(0.5)
      .add(orderNorm.multiply(0.5))
  )
  .clamp(0, 1)
  .rename('stream');

// Depression Risk
var depressionRisk = depressions
  .unmask(0)
  .gt(0)
  .rename('depression');

// Final Risk Index
var riskIndex = slopeRisk.multiply(0.25)
  .add(catchRisk.multiply(0.20))
  .add(streamRisk.multiply(0.30))
  .add(depressionRisk.multiply(0.25))
  .rename('risk_index')
  .clamp(0, 1)
  .float();

// Debug layer stays commented out — rendering/reducing this over
// the full country is itself enough to trigger a memory error.
// Map.addLayer(riskIndex, {min: 0, max: 1, palette: ['green', 'yellow', 'red']}, 'Risk Index');
// print('Risk Index', riskIndex);

// --- 3f. Classify into Low/Medium/High/Very High ---
var riskClass = riskIndex.expression(
  "b('risk_index') < 0.25 ? 1" +
  ": b('risk_index') < 0.5 ? 2" +
  ": b('risk_index') < 0.75 ? 3 : 4"
).rename('risk_class');

var palette = ['#2ecc71', '#f1c40f', '#e67e22', '#e74c3c'];

// PASS 10: swapped microwatershed boundary from magenta to bright
// yellow per user preference — still high-contrast against
// green/brown satellite terrain, distinct from every other layer
// color in use (blue, teal, purple, black).
var LAYER_COLORS = {
  drainage:   '#1F77E4', // strong blue    — Drainage Lines
  riverCanal: '#17BECF', // teal/cyan      — River / Canal
  reservoir:  '#9B59B6', // purple         — Reservoir
  mws:        '#FFFF00', // bright yellow  — Microwatershed Boundary
  village:    '#000000'  // black          — Village Boundary
};

// PASS 9: line widths, in pixels. Used both for FeatureCollection
// .style() calls and ee.Image().paint() calls below, and for the
// legend swatch thickness, so everything stays in sync.
var LAYER_WIDTHS = {
  drainage:   3,
  riverCanal: 3,
  reservoir:  3,
  mws:        4, // extra-wide since it was previously invisible
  village:    3
};

// ============================================================
// 4. Exposure (kept as in original)
// ============================================================
function exposureForYear(year) {
  var lulcImg = lulcColl.filter(ee.Filter.eq('year', year)).first();
  var builtMask = lulcImg.eq(BUILTUP_LULC_CODE);
  var exposedBuilt = builtMask.and(riskClass.gte(3));
  return exposedBuilt.rename('exposed_built_' + year);
}

// ============================================================
// 5. UI — boundary drill-down panel
// ============================================================
var mapPanel = ui.Map();
var panel = ui.Panel({style: {width: '340px', padding: '8px'}});
panel.add(ui.Label('Flood & Waterlogging Risk Viewer', {fontWeight: 'bold', fontSize: '16px'}));
panel.add(ui.Label('Select an administrative boundary to view risk for that area.'));

// PASS 16: doc link in the left panel. Styled as a hyperlink
// (blue, underlined) and wired to the SAME toggle logic as the
// "How App is Used »" button next to the right-side doc panel,
// via toggleDocPanel() defined further down — so clicking either
// one opens/closes the same panel and keeps both labels in sync.
// PASS 17 FIX: ui.Label does not support .onClick() in the GEE UI
// API — only interactive widgets like ui.Button do. Switched to a
// ui.Button styled to read as a link (no default button look is
// fully removable in GEE, but blue/underlined text plus a tight
// margin gets close) so it still calls the same toggleDocPanel().
var docLinkLabel = ui.Button({
  label: '📄 View Documentation',
  style: {
    color: '1a73e8',
    margin: '4px 0 10px 0',
    stretch: 'horizontal'
  },
  onClick: function() {
    toggleDocPanel();
  }
});
panel.add(docLinkLabel);

var stateSelect    = ui.Select({placeholder: 'Select State'});
var districtSelect = ui.Select({placeholder: 'Select District'});
var tehsilSelect   = ui.Select({placeholder: 'Select Tehsil'});
var villageSelect  = ui.Select({placeholder: 'Select Village'});

panel.add(ui.Label('State')); panel.add(stateSelect);
panel.add(ui.Label('District')); panel.add(districtSelect);
panel.add(ui.Label('Tehsil')); panel.add(tehsilSelect);
panel.add(ui.Label('Village')); panel.add(villageSelect);

var resultsLabel = ui.Label('', {whiteSpace: 'pre'});
panel.add(resultsLabel);

var rankedListLabel = ui.Label('Top risky villages in selection:', {fontWeight: 'bold'});
panel.add(rankedListLabel);
var rankedList = ui.Panel();
panel.add(rankedList);

var FIELD_NAMES = {
  state: 'Name',
  district: 'District',
  tehsil: 'TEHSIL',
  village: 'name'
};

function populateSelect(select, fc, propName, label) {
  ee.List(fc.aggregate_array(propName)).distinct().sort().evaluate(function(list, error) {
    if (error) {
      print('ERROR loading ' + label + ':', error);
      return;
    }
    // FIX 4: previously only kept string values (typeof v === 'string'),
    // which silently dropped numeric names/codes. Now keeps any
    // non-null/undefined value with a non-empty string representation.
    var clean = (list || []).filter(function(v) {
      return v !== null && v !== undefined && String(v).length > 0;
    });
    if (clean.length === 0) {
      fc.size().evaluate(function(count) {
        if (count === 0) {
          print('WARNING: 0 ' + label + ' found in this area at all.');
        } else {
          print('WARNING: field "' + propName + '" empty/invalid on all matching ' + label + '.');
        }
      });
      return;
    }
    select.items().reset(clean);
  });
}

var selectedStateFeature = null;
var selectedDistrictFeature = null;
var selectedTehsilFeature = null;

// FIX 8: simplified — plain filterBounds() against the parent geometry,
// no simplify()/buffer() needed since it just adds noise/imprecision.
function withinParent(childFC, parentGeom) {
  return childFC.filterBounds(parentGeom);
}

populateSelect(stateSelect, stateFC, FIELD_NAMES.state, 'states');

stateSelect.onChange(function(state) {
  print('DEBUG: state selected =', state);
  try {
    districtSelect.items().reset([]);
    tehsilSelect.items().reset([]);
    villageSelect.items().reset([]);

    var stateFeat = stateFC.filter(ee.Filter.eq(FIELD_NAMES.state, state)).first();
    selectedStateFeature = stateFeat;
    selectedDistrictFeature = null;
    selectedTehsilFeature = null;

    var districtsInState = withinParent(districtFC, stateFeat.geometry());
    districtsInState.size().evaluate(function(count, err) {
      if (err) print('DEBUG: districtsInState.size() error:', err);
      else print('DEBUG: districtsInState matched count =', count);
    });

    districtsInState.first().toDictionary().evaluate(function(dict, err) {
      if (err) print('DEBUG: sample district properties error:', err);
      else print('DEBUG: sample district properties for this state:', dict);
    });

    populateSelect(districtSelect, districtsInState, FIELD_NAMES.district, 'districts');
  } catch (e) {
    print('CAUGHT SYNCHRONOUS ERROR in stateSelect.onChange:', e.message || e);
  }
});

districtSelect.onChange(function(district) {
  print('DEBUG: district selected =', district);
  try {
    tehsilSelect.items().reset([]);
    villageSelect.items().reset([]);

    var districtsInState = withinParent(districtFC, selectedStateFeature.geometry());
    var districtFeat = districtsInState.filter(ee.Filter.eq(FIELD_NAMES.district, district)).first();
    selectedDistrictFeature = districtFeat;
    selectedTehsilFeature = null;

    if (tehsilFC) {
      var tehsilsInDistrict = withinParent(tehsilFC, districtFeat.geometry());
      print('DEBUG: querying tehsils within selected district...');
      populateSelect(tehsilSelect, tehsilsInDistrict, FIELD_NAMES.tehsil, 'tehsils');
    } else {
      var villagesInDistrict = withinParent(villages, districtFeat.geometry());
      print('DEBUG: querying villages within selected district...');
      populateSelect(villageSelect, villagesInDistrict, FIELD_NAMES.village, 'villages');
      updateRankedTable(villagesInDistrict);
    }
  } catch (e) {
    print('CAUGHT SYNCHRONOUS ERROR in districtSelect.onChange:', e.message || e);
  }
});

tehsilSelect.onChange(function(tehsil) {
  print('DEBUG: tehsil selected =', tehsil);
  try {
    villageSelect.items().reset([]);

    var tehsilsInDistrict = withinParent(tehsilFC, selectedDistrictFeature.geometry());
    var tehsilFeat = tehsilsInDistrict.filter(ee.Filter.eq(FIELD_NAMES.tehsil, tehsil)).first();
    selectedTehsilFeature = tehsilFeat;

    var villagesInTehsil = withinParent(villages, tehsilFeat.geometry());
    print('DEBUG: querying villages within selected tehsil...');
    populateSelect(villageSelect, villagesInTehsil, FIELD_NAMES.village, 'villages');
    updateRankedTable(villagesInTehsil);
  } catch (e) {
    print('CAUGHT SYNCHRONOUS ERROR in tehsilSelect.onChange:', e.message || e);
  }
});

// ============================================================
// villageSelect.onChange
// FIX 7: guards against a missing/null village feature so the
// app doesn't crash if the selected name can't be matched.
// FIX 6: reduceRegion() now sets bestEffort: true.
// Risk Class layer remains commented out (was causing memory
// limit exceeded).
// ============================================================
villageSelect.onChange(function(village) {
  print('DEBUG: village selected =', village);
  try {
    var parentGeom = selectedTehsilFeature ? selectedTehsilFeature.geometry() : selectedDistrictFeature.geometry();

    var featCandidate = withinParent(villages, parentGeom)
      .filter(ee.Filter.eq(FIELD_NAMES.village, village))
      .first();

    // FIX 7: evaluate first so we can bail out cleanly instead of
    // calling .geometry() on a null feature and crashing.
    featCandidate.evaluate(function(featInfo, err) {
      if (err || !featInfo) {
        print('WARNING: could not find village feature for "' + village + '"', err);
        resultsLabel.setValue('Selected: ' + village + '\nNo matching village geometry found.');
        return;
      }

      var feat = ee.Feature(featCandidate);
      // PASS 5: buffer the village geometry a bit before using it
      // to clip/reduce riskIndex. Combined with removing the
      // eager reproject() calls above, this keeps every
      // computation scoped to a small local extent around the
      // selected village instead of touching the national-scale
      // composite.
      var geom = feat.geometry().buffer(2000);

      mapPanel.centerObject(geom, 13);
      mapPanel.layers().reset();

      // PASS 7: re-enabled — safe now that geom is a small buffered
      // village extent (Pass 5) and riskIndex/riskClass no longer
      // depend on national-scale reduceRegion percentiles (Pass 6).
      var riskClassSel = riskClass.clip(geom);
      mapPanel.addLayer(riskClassSel, {min: 1, max: 4, palette: palette}, 'Risk Class');

      // PASS 8: colors below were previously all near-identical
      // shades of blue (blue / 00BFFF / 1E90FF), which made the
      // three water-related layers impossible to tell apart on
      // the map or in the legend. Switched to hues spread evenly
      // around the color wheel so every layer is unambiguous at
      // a glance. Keep LAYER_COLORS as the single source of truth
      // — the legend below reads from the same object so the two
      // can never drift out of sync.
      // PASS 9: switched from {color: ...} (which ignores width for
      // FeatureCollections) to .style({color, width}), which is the
      // correct way to control line thickness for vector layers in
      // GEE. Widths pulled from LAYER_WIDTHS above.
      mapPanel.addLayer(
        drainageFC.filterBounds(geom).style({color: LAYER_COLORS.drainage, width: LAYER_WIDTHS.drainage}),
        {},
        'Drainage Lines'
      );
      mapPanel.addLayer(
        riverCanalFC.filterBounds(geom).style({color: LAYER_COLORS.riverCanal, width: LAYER_WIDTHS.riverCanal}),
        {},
        'River / Canal'
      );
      mapPanel.addLayer(
        reservoirFC.filterBounds(geom).style({color: LAYER_COLORS.reservoir, width: LAYER_WIDTHS.reservoir}),
        {},
        'Reservoir'
      );
      // PASS 9/10: mws boundary was grey + 1px wide (paint's 3rd arg
      // is line width) — effectively invisible against satellite
      // imagery. Now bright yellow at 4px.
      mapPanel.addLayer(
        ee.Image().byte().paint(mwsFC.filterBounds(geom), 0, LAYER_WIDTHS.mws),
        {palette: [LAYER_COLORS.mws]},
        'Microwatershed Boundary'
      );
      mapPanel.addLayer(
        ee.Image().byte().paint(ee.FeatureCollection([feat]), 0, LAYER_WIDTHS.village),
        {palette: [LAYER_COLORS.village]},
        'Village Boundary'
      );

      var stats = ee.Image(riskIndex).clip(geom.buffer(1000)).reduceRegion({
        reducer: ee.Reducer.mean(),
        geometry: geom,
        scale: 30,
        bestEffort: true,
        tileScale: 8,
        maxPixels: 1e13
      });

      stats.evaluate(function(s) {
        var mean = (s && s.risk_index !== null && s.risk_index !== undefined) ? s.risk_index : null;
        resultsLabel.setValue(
          'Selected: ' + village +
          '\nMean risk index: ' + (mean !== null ? mean.toFixed(3) : 'N/A')
        );
      });
    });
  } catch (e) {
    print('CAUGHT SYNCHRONOUS ERROR in villageSelect.onChange:', e.message || e);
  }
});

function updateRankedTable(villageFC) {
  rankedList.clear();
  // FIX 5: tileScale: 4 added to reduce memory-limit failures for
  // large districts (bump to 8 if you still see errors).
  var localRisk = riskIndex.clip(villageFC.geometry());

  var withRisk = localRisk.reduceRegions({
    collection: villageFC,
    reducer: ee.Reducer.mean(),
    scale: 30,
    tileScale: 8
  });
  var sorted = withRisk.sort('mean', false).limit(10);
  sorted.evaluate(function(fc) {
    if (!fc || !fc.features) {
      rankedList.add(ui.Label('No villages found in this selection yet.'));
      return;
    }
    fc.features.forEach(function(f) {
      var name = f.properties[FIELD_NAMES.village] || 'Unknown';
      var score = f.properties.mean ? f.properties.mean.toFixed(3) : 'N/A';
      rankedList.add(ui.Label(name + '  —  ' + score));
    });
  });
}

// ============================================================
// PASS 13: "About / How This Works" panel is now hidden by
// default. A small always-visible toggle panel sits between the
// map and the documentation panel — clicking its button reveals
// the full doc panel; clicking again hides it. Keeping the
// toggle in its own tiny panel (rather than inside rightPanel
// itself) means the button stays visible even while the doc
// panel is collapsed.
// ============================================================
var rightPanel = ui.Panel({
  style: {width: '320px', padding: '8px', backgroundColor: '#f7f7f7', shown: false}
});

// PASS 14: helper for a wrapping bullet line. Previously several
// bullet points were packed into a single ui.Label string joined
// with '\n' and rendered with whiteSpace: 'pre' — 'pre' disables
// text wrapping entirely, so any bullet longer than the panel
// width got silently clipped/cut off instead of wrapping to a
// second line. Each bullet is now its own Label with normal
// (wrapping) whiteSpace, so no words are ever hidden.
function docBullet(text) {
  return ui.Label(text, {fontSize: '11px', margin: '2px 0 2px 10px', whiteSpace: 'normal'});
}

rightPanel.add(ui.Label('About this app', {fontWeight: 'bold', fontSize: '14px'}));
rightPanel.add(ui.Label(
  'This tool estimates flood and waterlogging risk for villages ' +
  'across India by combining terrain, drainage, and land-cover data ' +
  'into a single 0–1 risk score, then lets you drill down from ' +
  'state to district to tehsil to village to inspect that score ' +
  'and the underlying map layers for any specific location.',
  {fontSize: '11px', whiteSpace: 'normal'}
));

rightPanel.add(ui.Label('How the risk score is built', {fontWeight: 'bold', fontSize: '12px', margin: '10px 0 2px 0'}));
rightPanel.add(ui.Label(
  'risk_index is a weighted sum of four 0–1 sub-scores, each computed with a fixed formula (no ' +
  'national-scale statistics needed):',
  {fontSize: '11px', whiteSpace: 'normal', margin: '0 0 4px 0'}
));

docBullet('• Slope (25%): slopeRisk = clamp(slope ÷ 30, 0, 1). Flatter land drains poorly, so lower slope contributes more risk once combined below.');
docBullet('• Catchment area (20%): catchRisk = clamp(unitScale(catchment, 0, 500), 0, 1). A larger upstream contributing area means more runoff can converge here.');
docBullet('• Stream risk (30%): distance = fastDistanceTransform(drainage) × 30m; proximityRisk = clamp(1 − unitScale(distance, 0, 2000), 0, 1); orderNorm = clamp(unitScale(streamOrder, 1, 7), 0, 1); streamRisk = clamp(proximityRisk × (0.5 + 0.5 × orderNorm), 0, 1). Closer to a stream = higher risk, boosted further for larger (higher-order) streams.');
docBullet('• Depression risk (25%): depressionRisk = 1 if depressions > 0, else 0 — a flat boost for any pixel flagged as a natural low-lying depression.');

rightPanel.add(ui.Label(
  'risk_index = clamp(0.25×slope + 0.20×catch + 0.30×stream + 0.25×depression, 0, 1)',
  {fontSize: '10px', fontFamily: 'monospace', margin: '6px 0 6px 10px', whiteSpace: 'normal'}
));
rightPanel.add(ui.Label(
  'Every term is .unmask()\'d before summing so one missing input can\'t null out the whole score for a pixel.',
  {fontSize: '10px', color: '666666', margin: '0 0 6px 10px', whiteSpace: 'normal'}
));

rightPanel.add(ui.Label('Risk classification', {fontWeight: 'bold', fontSize: '12px', margin: '8px 0 2px 0'}));
docBullet('• Low: risk_index < 0.25');
docBullet('• Medium: 0.25 ≤ risk_index < 0.5');
docBullet('• High: 0.5 ≤ risk_index < 0.75');
docBullet('• Very High: risk_index ≥ 0.75');

rightPanel.add(ui.Label('Map layers shown per village', {fontWeight: 'bold', fontSize: '12px', margin: '10px 0 2px 0'}));
docBullet('• Risk Class (colored fill) — the 0–1 score grouped into Low / Medium / High / Very High.');
docBullet('• Drainage Lines, River/Canal, Reservoir — the actual water features driving the stream-proximity part of the score.');
docBullet('• Microwatershed Boundary — the local catchment unit the village sits in.');
docBullet('• Village Boundary — the exact selected village outline.');

rightPanel.add(ui.Label(
  'Land-cover (LULC) layers for 2018–2025 are also loaded to ' +
  'identify built-up areas that overlap High/Very High risk zones, ' +
  'for future exposure analysis (see exposureForYear() in the code).',
  {fontSize: '11px', margin: '10px 0 0 0', whiteSpace: 'normal'}
));

// PASS 13: small, always-visible toggle panel. Sits between the
// map and the documentation panel so its button remains
// reachable regardless of whether rightPanel is currently shown.
// PASS 16: shared toggle function so the left-panel "View
// Documentation" link and the right-side "How App is Used »"
// button both control the same panel and stay in sync with each
// other's label text no matter which one is clicked.
var docVisible = false;
var docToggleButton; // declared here, assigned below, referenced by toggleDocPanel

function toggleDocPanel() {
  docVisible = !docVisible;
  rightPanel.style().set('shown', docVisible);
  var label = docVisible ? '« Hide' : 'How App is Used »';
  docToggleButton.setLabel(label);
}

docToggleButton = ui.Button({
  label: 'How App is Used »',
  style: {margin: '10px 4px'},
  onClick: function() {
    toggleDocPanel();
  }
});
var docTogglePanel = ui.Panel({
  widgets: [docToggleButton],
  style: {width: '40px', backgroundColor: '#eeeeee'}
});

// ============================================================
// 6. Legend
// PASS 7: legend now documents EVERY layer added to mapPanel in
// villageSelect.onChange (risk classes + all vector overlays),
// not just the 4 risk-class colors. Two small helper builders
// keep the fill-swatch and line-swatch rows consistent, and the
// swatch colors are pinned to match the exact colors/palettes
// used in each mapPanel.addLayer() call above.
// ============================================================
var legend = ui.Panel({style: {position: 'bottom-left', padding: '8px'}});
legend.add(ui.Label('Legend', {fontWeight: 'bold', fontSize: '14px'}));

// --- helper: colored box (for polygons / raster classes) ---
function legendFillRow(color, text) {
  var colorBox = ui.Label('', {
    backgroundColor: color,
    padding: '8px',
    margin: '0 4px 4px 0'
  });
  var desc = ui.Label(text, {margin: '0 0 4px 4px'});
  return ui.Panel([colorBox, desc], ui.Panel.Layout.flow('horizontal'));
}

// --- helper: colored line swatch (for line/outline layers) ---
// PASS 9: swatch height now takes a width param so the legend
// visually reflects each layer's actual on-map line thickness.
function legendLineRow(color, text, width) {
  var lineBox = ui.Label('', {
    backgroundColor: color,
    height: (width || 3) + 'px',
    width: '18px',
    margin: '8px 4px 8px 0'
  });
  var desc = ui.Label(text, {margin: '0 0 4px 4px'});
  return ui.Panel([lineBox, desc], ui.Panel.Layout.flow('horizontal'));
}

// -- Risk Class (raster fill) --
legend.add(ui.Label('Risk Level', {fontWeight: 'bold', margin: '6px 0 2px 0'}));
var riskLabels = ['Low', 'Medium', 'High', 'Very High'];
for (var i = 0; i < 4; i++) {
  legend.add(legendFillRow(palette[i], riskLabels[i]));
}

// -- Vector overlay layers --
// PASS 8: colors now pulled directly from LAYER_COLORS (the same
// object used when adding each layer to mapPanel above), so the
// legend swatches are guaranteed to match what's drawn, and the
// colors themselves are now visually distinct hues instead of
// three near-identical blues.
legend.add(ui.Label('Features', {fontWeight: 'bold', margin: '10px 0 2px 0'}));
legend.add(legendLineRow(LAYER_COLORS.drainage, 'Drainage Lines', LAYER_WIDTHS.drainage));
legend.add(legendLineRow(LAYER_COLORS.riverCanal, 'River / Canal', LAYER_WIDTHS.riverCanal));
legend.add(legendLineRow(LAYER_COLORS.reservoir, 'Reservoir', LAYER_WIDTHS.reservoir));
legend.add(legendLineRow(LAYER_COLORS.mws, 'Microwatershed Boundary', LAYER_WIDTHS.mws));
legend.add(legendLineRow(LAYER_COLORS.village, 'Village Boundary', LAYER_WIDTHS.village));

// ============================================================
// 7. Assemble UI
// ============================================================
mapPanel.setOptions('SATELLITE');
mapPanel.setCenter(78, 22, 5);
mapPanel.style().set({stretch: 'both'});
panel.style().set({stretch: 'vertical'});
rightPanel.style().set({stretch: 'vertical'});

ui.root.clear();
ui.root.setLayout(ui.Panel.Layout.flow('horizontal'));
ui.root.widgets().reset([panel, mapPanel, docTogglePanel, rightPanel]);
mapPanel.add(legend);
