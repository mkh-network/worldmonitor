# Phase 4: Global Map & Positive Events - Research

**Researched:** 2026-02-23
**Domain:** Deck.gl map layers, geolocation of positive news, kindness data pipeline
**Confidence:** HIGH

## Summary

Phase 4 adds two new Deck.gl map layers to the happy variant: a positive-events layer (MAP-01, MAP-02) showing geocoded news stories with warm-colored markers, and a kindness layer (KIND-01, KIND-02) showing animated green pulses representing acts of goodwill worldwide. Both layers follow the established Deck.gl `ScatterplotLayer` pattern already used extensively in `DeckGLMap.ts`.

The codebase already has all the foundational infrastructure: the happy variant config (`src/config/variants/happy.ts`), the positive news pipeline with curated feeds + GDELT supplementary + sentiment gating (Phase 3), the `inferGeoHubsFromTitle()` geocoding system in `src/services/geo-hub-index.ts`, and the pulsing animation pattern used for hotspots and news locations. The GDELT GEO 2.0 API (already used for protests at `https://api.gdeltproject.org/api/v2/geo/geo`) can be repurposed with positive-themed queries to return GeoJSON with coordinates for positive events.

The kindness layer's hybrid data approach (KIND-02) combines estimated baseline pulses weighted by population density with real events. The `CountryPopulation` type with `densityPerKm2` is already available via the WorldPop/displacement service. GoFundMe and Change.org do not offer reliable public APIs for trending/victories data -- scraping or RSS alternatives are needed.

**Primary recommendation:** Add two new boolean keys (`positiveEvents`, `kindness`) to the `MapLayers` interface, create two new Deck.gl `ScatterplotLayer` methods in `DeckGLMap.ts` following the exact hotspot pulse pattern, and build a server-side RPC that queries GDELT GEO API with positive-themed queries to return geocoded positive events.

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| MAP-01 | Interactive map displaying geocoded positive events with warm-colored markers (green/gold pulses instead of red threat markers) | Use existing `ScatterplotLayer` pulse pattern from hotspots/news layers; green/gold color palette from happy theme; new `positiveEvents` key in `MapLayers` |
| MAP-02 | Positive event layer showing news stories geolocated on the map | GDELT GEO 2.0 API with positive queries returns GeoJSON with coordinates; existing `inferGeoHubsFromTitle()` geocodes RSS items; both feed into a unified positive-events data store |
| KIND-01 | Animated map layer showing acts of kindness, donations, volunteer events geolocated worldwide with green pulses | Separate `kindness` layer key; reuse pulse animation interval pattern; ScatterplotLayer with green RGB values and sine-wave radius oscillation |
| KIND-02 | Hybrid data approach: estimated baseline pulses weighted by population density, overlaid with real events from GoFundMe trending, Change.org wins, etc. | `CountryPopulation.densityPerKm2` available from WorldPop service; generate synthetic baseline points from major city coordinates; GoFundMe/Change.org lack public APIs -- use RSS feeds, web scraping via Apify, or curated kindness feed sources instead |
</phase_requirements>

## Standard Stack

### Core (Already Installed)
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| `@deck.gl/layers` | ^9.2.6 | `ScatterplotLayer` for both positive-events and kindness dots | Already used for all map markers in `DeckGLMap.ts` |
| `@deck.gl/core` | ^9.2.6 | Layer base types, `PickingInfo` | Already imported and used |
| `@deck.gl/mapbox` | ^9.2.6 | `MapboxOverlay` integration with MapLibre | Already the rendering pipeline |
| `maplibre-gl` | ^5.16.0 | Base map rendering | Already the map engine |
| `supercluster` | (installed) | Clustering for dense marker areas | Already used for protests, tech HQs, datacenters |

### Supporting (Already Available)
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| WorldPop service | existing | Population density data for baseline kindness pulses | `fetchCountryPopulations()` returns `CountryPopulation[]` with `densityPerKm2` |
| `geo-hub-index.ts` | existing | Geocode news titles to lat/lon via keyword matching | Already assigns lat/lon to RSS items via `inferGeoHubsFromTitle()` |
| GDELT GEO 2.0 API | existing | GeoJSON with coordinates for keyword queries | Already used for protests; reuse with positive queries |
| `positive-classifier.ts` | existing | Categorize positive events by type | Already classifies into 6 categories with color mappings |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| GDELT GEO API for positive event geocoding | GDELT DOC API + separate geocoding | DOC API does NOT return coordinates; GEO API returns GeoJSON natively -- use GEO API |
| Supercluster for kindness clustering | Manual grid-based bucketing | Supercluster is already in the codebase and proven; use it if kindness points get dense |
| GoFundMe API for real kindness events | RSS scraping / Apify scraper / curated feeds | GoFundMe Pro API requires partner access; Change.org API is deprecated; use curated RSS or scraper |

**Installation:**
```bash
# No new dependencies needed -- all libraries already installed
```

## Architecture Patterns

### Recommended Project Structure
```
src/
├── config/
│   ├── variants/happy.ts          # Add positiveEvents + kindness layer keys
│   └── panels.ts                  # Add layer keys to MapLayers type + HAPPY config
├── types/
│   └── index.ts                   # Add positiveEvents + kindness to MapLayers interface
├── services/
│   ├── positive-events-geo.ts     # NEW: fetch geocoded positive events from GDELT GEO API
│   └── kindness-data.ts           # NEW: generate baseline + fetch real kindness events
├── components/
│   └── DeckGLMap.ts               # Add createPositiveEventsLayer() + createKindnessLayer()
server/
└── worldmonitor/
    └── positive-events/v1/        # NEW: server-side RPC for GDELT GEO positive queries
        ├── handler.ts
        └── list-positive-geo-events.ts
proto/
└── worldmonitor/
    └── positive-events/v1/
        └── service.proto          # NEW: proto for geocoded positive events
```

### Pattern 1: ScatterplotLayer with Pulse Animation
**What:** Two-layer combo: solid fill layer + stroked ring layer with oscillating `radiusScale`
**When to use:** All animated marker layers (hotspots, news pulses, positive events, kindness)
**Example:**
```typescript
// Source: src/components/DeckGLMap.ts line 2107-2167 (existing hotspot pulse pattern)
private createPositiveEventsLayers(): Layer[] {
  const layers: Layer[] = [];

  // Solid fill layer -- green/gold based on category
  layers.push(new ScatterplotLayer({
    id: 'positive-events-layer',
    data: this.positiveEvents,
    getPosition: (d) => [d.lon, d.lat],
    getRadius: 15000,
    getFillColor: (d) => {
      // Green for nature/kindness, gold for science/innovation
      return d.category === 'nature-wildlife' || d.category === 'humanity-kindness'
        ? [34, 197, 94, 180]   // green
        : [234, 179, 8, 180];  // gold
    },
    radiusMinPixels: 4,
    radiusMaxPixels: 14,
    pickable: true,
  }));

  // Pulse ring layer -- animated expanding ring
  const pulse = 1.0 + 0.6 * (0.5 + 0.5 * Math.sin((this.pulseTime || Date.now()) / 500));
  layers.push(new ScatterplotLayer({
    id: 'positive-events-pulse',
    data: this.positiveEvents.filter(d => this.isRecent(d)),
    getPosition: (d) => [d.lon, d.lat],
    getRadius: 15000,
    radiusScale: pulse,
    radiusMinPixels: 6,
    radiusMaxPixels: 24,
    stroked: true,
    filled: false,
    getLineColor: [34, 197, 94, 100],
    lineWidthMinPixels: 1.5,
    pickable: false,
    updateTriggers: { radiusScale: this.pulseTime },
  }));

  return layers;
}
```

### Pattern 2: GDELT GEO API Query for Positive Events
**What:** Server-side fetch of GDELT GEO 2.0 API with positive-themed queries, returning GeoJSON features with coordinates
**When to use:** Getting geocoded positive event data without separate geocoding step
**Example:**
```typescript
// Source: server/worldmonitor/unrest/v1/list-unrest-events.ts line 108-177 (existing pattern)
const GDELT_GEO_URL = 'https://api.gdeltproject.org/api/v2/geo/geo';

async function fetchPositiveGeoEvents(query: string): Promise<PositiveGeoEvent[]> {
  const params = new URLSearchParams({
    query,
    format: 'geojson',
    maxrecords: '100',
    timespan: '24h',
  });

  const response = await fetch(`${GDELT_GEO_URL}?${params}`, {
    headers: { Accept: 'application/json' },
    signal: AbortSignal.timeout(10000),
  });

  const data = await response.json();
  return (data?.features || [])
    .filter(f => f.geometry?.coordinates?.length >= 2)
    .map(f => ({
      lat: f.geometry.coordinates[1],
      lon: f.geometry.coordinates[0],
      name: f.properties?.name || '',
      count: f.properties?.count || 1,
    }));
}
```

### Pattern 3: Population-Weighted Baseline Kindness Points
**What:** Generate synthetic kindness markers at major cities weighted by population density
**When to use:** KIND-02 baseline layer that provides visual activity even without real event data
**Example:**
```typescript
// Generate baseline kindness pulses from major city coordinates
// Weighted by population density -- more pulses in dense areas
import { MAJOR_CITIES } from './kindness-cities';  // curated list with lat/lon/pop

function generateBaselineKindness(): KindnessPoint[] {
  const now = Date.now();
  return MAJOR_CITIES.map(city => ({
    lat: city.lat + (Math.random() - 0.5) * 0.5,  // jitter within city
    lon: city.lon + (Math.random() - 0.5) * 0.5,
    intensity: Math.min(1, city.population / 10_000_000),
    type: 'baseline' as const,
    timestamp: now - Math.random() * 3600_000,  // random time within last hour
  }));
}
```

### Pattern 4: MapLayers Extension
**What:** Adding new boolean keys to the `MapLayers` interface and wiring them through the variant config
**When to use:** Every new map layer addition
**Example:**
```typescript
// Source: src/types/index.ts line 486-526 (existing pattern)
// 1. Add to MapLayers interface
export interface MapLayers {
  // ... existing keys ...
  // Happy variant layers
  positiveEvents: boolean;
  kindness: boolean;
}

// 2. Add to happy variant config (src/config/panels.ts + src/config/variants/happy.ts)
const HAPPY_MAP_LAYERS: MapLayers = {
  // ... all existing false keys ...
  natural: true,
  positiveEvents: true,   // NEW
  kindness: true,          // NEW
};

// 3. Add to layer toggles in DeckGLMap.ts createLayerToggles()
// 4. Add to legend in DeckGLMap.ts createLegend()
// 5. Add to buildLayers() conditional
```

### Anti-Patterns to Avoid
- **Creating custom WebGL shaders for pulse animation:** The codebase uses `radiusScale` + `Math.sin()` + interval timer -- do NOT write custom shaders. The existing pattern works and is proven.
- **Fetching GDELT GEO API client-side:** All external API calls go through server-side RPC handlers. The client calls the sebuf service which proxies to GDELT.
- **Adding positive/kindness layers to non-happy variants:** These layers are ONLY for `SITE_VARIANT === 'happy'`. Guard all additions with variant checks.
- **Over-fetching GDELT data:** Use circuit breakers and caching (5-minute TTL) as done for all other GDELT calls. Rate limit compliance is critical.
- **Blocking UI on GDELT GEO fetch:** Data should load asynchronously after initial render, not block the happy pipeline.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Geocoding news titles | Custom NLP entity extraction + geocoding API | `inferGeoHubsFromTitle()` from `geo-hub-index.ts` | Already 80+ strategic locations with keyword matching; proven in production |
| GeoJSON parsing from GDELT | Custom coordinate extraction | Standard GeoJSON `feature.geometry.coordinates` | GeoJSON is a standard; GDELT returns compliant GeoJSON |
| Point clustering at low zoom | Manual grid bucketing | `Supercluster` (already installed) | Handles zoom-dependent clustering with proven API |
| Pulse animation timing | `requestAnimationFrame` loop | `setInterval` + `radiusScale` pattern | Existing pattern uses 500ms interval with `sin()` oscillation; performant and proven |
| Population density data | Manual country density calculations | `fetchCountryPopulations()` from WorldPop service | Returns `CountryPopulation[]` with `densityPerKm2` |

**Key insight:** Every visualization primitive needed for this phase already exists in `DeckGLMap.ts`. The work is composing existing patterns with new data, not inventing new rendering techniques.

## Common Pitfalls

### Pitfall 1: MapLayers Type Exhaustiveness
**What goes wrong:** Adding new keys to `MapLayers` interface without updating ALL variant configs causes TypeScript errors
**Why it happens:** `MapLayers` is used in `panels.ts` for FULL, TECH, FINANCE, and HAPPY variants, plus mobile variants
**How to avoid:** When adding `positiveEvents` and `kindness` keys, add them (set to `false`) in ALL 8 variant layer configs (4 desktop + 4 mobile), plus the `LAYER_TO_SOURCE` map if applicable
**Warning signs:** TypeScript build errors about missing properties in object literals

### Pitfall 2: GDELT GEO API Rate Limits
**What goes wrong:** Too many concurrent queries to GDELT GEO API cause 429 errors or temporary blocks
**Why it happens:** Phase queries for 6+ positive topic categories in parallel
**How to avoid:** Sequential fetching with 500ms delays between queries, or batch into a single compound query using OR operators. Use circuit breaker pattern already in `gdelt-intel.ts`.
**Warning signs:** Empty results after initial success, 429 HTTP status codes

### Pitfall 3: Too Many Kindness Baseline Points
**What goes wrong:** Generating hundreds of synthetic baseline points causes frame drops in `buildLayers()`
**Why it happens:** One point per city times multiple random jitters creates excessive ScatterplotLayer data
**How to avoid:** Cap baseline points at 50-100 globally. Use major cities only (population > 2M). Regenerate on a 5-minute timer, not every render cycle.
**Warning signs:** `buildLayers took >16ms` console warnings, choppy map interaction

### Pitfall 4: Pulse Animation Memory Leak
**What goes wrong:** setInterval for pulse animation continues running when map tab is inactive
**Why it happens:** No cleanup when map panel is hidden or render is paused
**How to avoid:** Follow existing `syncPulseAnimation()` pattern: check `renderPaused` flag, call `stopPulseAnimation()` on pause, restart on resume. Tie to existing `newsPulseIntervalId` mechanism or create parallel `kindnessPulseIntervalId`.
**Warning signs:** Growing memory, CPU usage when tab is backgrounded

### Pitfall 5: Missing Layer Toggle for Happy Variant
**What goes wrong:** New layers don't appear in the layer toggle panel
**Why it happens:** `createLayerToggles()` has no `SITE_VARIANT === 'happy'` branch -- it falls through to the full variant
**How to avoid:** Add a new happy variant branch in `createLayerToggles()` with `positiveEvents` and `kindness` toggles (plus `natural` which is already enabled)
**Warning signs:** Layers render but users can't toggle them off

### Pitfall 6: GoFundMe/Change.org API Unavailability
**What goes wrong:** Building a pipeline that depends on APIs that don't exist publicly
**Why it happens:** GoFundMe Pro API requires partner access; Change.org API is deprecated/unreliable
**How to avoid:** Design kindness data pipeline with graceful degradation. Baseline (synthetic) always works. Real events from curated RSS feeds (Good News Network, Positive News, Reasons to be Cheerful) are the reliable source. GoFundMe/Change.org are aspirational -- use scraper-based approach (Apify) or skip until APIs become available.
**Warning signs:** Empty real-events data, broken promises on API access

## Code Examples

Verified patterns from the existing codebase:

### Adding a New Map Layer Key (Complete Flow)
```typescript
// Step 1: src/types/index.ts -- add to MapLayers interface
export interface MapLayers {
  // ... existing ...
  positiveEvents: boolean;
  kindness: boolean;
}

// Step 2: src/config/panels.ts -- add to ALL variant configs (set false for non-happy)
const FULL_MAP_LAYERS: MapLayers = {
  // ... existing ...
  positiveEvents: false,
  kindness: false,
};
// Repeat for FULL_MOBILE, TECH, TECH_MOBILE, FINANCE, FINANCE_MOBILE

const HAPPY_MAP_LAYERS: MapLayers = {
  // ... existing (all false except natural: true) ...
  positiveEvents: true,   // ENABLED for happy
  kindness: true,          // ENABLED for happy
};
// Repeat for HAPPY_MOBILE

// Step 3: src/config/variants/happy.ts -- add to both DEFAULT_MAP_LAYERS and MOBILE
// (mirrors panels.ts happy config)
```

### Data Setter Pattern (Following Existing Convention)
```typescript
// Source: DeckGLMap.ts lines 3342-3362 (existing pattern for setClimateAnomalies, setNewsLocations)
public setPositiveEvents(events: PositiveGeoEvent[]): void {
  this.positiveEvents = events;
  this.render();
}

public setKindnessData(points: KindnessPoint[]): void {
  this.kindnessPoints = points;
  this.render();
}
```

### Tooltip Pattern (Following Existing Convention)
```typescript
// Source: DeckGLMap.ts line 2329 (existing getTooltip switch)
case 'positive-events-layer':
  return {
    html: `<div class="map-tooltip">
      <strong>${text(obj.name)}</strong>
      <br>${text(obj.category)} &bull; ${obj.count} reports
    </div>`,
  };

case 'kindness-layer':
  return {
    html: `<div class="map-tooltip">
      <strong>${text(obj.city || obj.name)}</strong>
      <br>Acts of kindness nearby
    </div>`,
  };
```

### Legend Items for Happy Variant
```typescript
// Source: DeckGLMap.ts line 3034 (existing legend pattern)
const legendItems = SITE_VARIANT === 'happy'
  ? [
      { shape: shapes.circle('rgb(34, 197, 94)'), label: 'Positive Event' },
      { shape: shapes.circle('rgb(234, 179, 8)'), label: 'Breakthrough' },
      { shape: shapes.circle('rgb(74, 222, 128)'), label: 'Act of Kindness' },
      { shape: shapes.circle('rgb(255, 100, 50)'), label: 'Natural Event' },
    ]
  : SITE_VARIANT === 'tech'
  // ... existing
```

### GDELT GEO Query for Positive Topics
```typescript
// Reuse queries from POSITIVE_GDELT_TOPICS in src/services/gdelt-intel.ts
// but send to GEO API instead of DOC API
const POSITIVE_GEO_QUERIES = [
  'breakthrough OR discovery OR "new treatment"',
  'renewable energy record OR "solar installation" OR "wind farm"',
  'species recovery OR "conservation success" OR "habitat restored"',
  'poverty decline OR "literacy rate" OR "peace agreement"',
  '"clean technology" OR "AI healthcare" OR "fusion energy"',
];
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Custom WebGL point rendering | Deck.gl ScatterplotLayer with `radiusScale` | Deck.gl 8.x+ (2022+) | No custom shaders needed; 100k+ points at 60fps |
| Manual geocoding via external API | Keyword-based location matching via geo-hub-index | Already in codebase | Zero API calls for geocoding RSS items |
| GDELT DOC API only | GDELT GEO 2.0 API for coordinates | Available since ~2020 | Returns GeoJSON with lat/lon natively |
| Separate animation RAF loop | Shared pulse interval with `radiusScale` update trigger | Current codebase pattern | Single timer drives all pulsing layers |

**Deprecated/outdated:**
- GDELT DOC API does NOT return coordinates -- do not try to extract geo from it. Use GEO API instead.
- GoFundMe public API -- deprecated/partner-only. Use scraping or curated feeds.
- Change.org REST API -- deprecated/unreliable. The Google Groups forum shows ongoing developer complaints.

## Open Questions

1. **Happy variant layer toggle UI**
   - What we know: `createLayerToggles()` has branches for tech, finance, and full -- no explicit happy branch. Happy falls through to full variant toggles.
   - What's unclear: Should happy have its own minimal toggle list (positiveEvents, kindness, natural) or should it show no toggles at all since there are only 2-3 layers?
   - Recommendation: Add a `SITE_VARIANT === 'happy'` branch with just `positiveEvents`, `kindness`, and `natural` toggles.

2. **Real kindness event data sources**
   - What we know: GoFundMe Pro API is partner-only. Change.org API is deprecated. No unified "world kindness events" API exists.
   - What's unclear: Whether scraping is reliable enough for production, and whether curated RSS feeds have enough kindness-specific content with geolocation.
   - Recommendation: Start with curated feeds (Good News Network, Reasons to be Cheerful, Positive.News) that already flow through Phase 3. Extract kindness-category items from the positive pipeline. Add GoFundMe scraping as a stretch goal only.

3. **GDELT GEO API positive query coverage**
   - What we know: The GEO API works for protests with `query: 'protest'`. Positive queries like "breakthrough OR discovery" should work similarly.
   - What's unclear: Whether positive-themed queries return enough geolocated results (the GEO API aggregates by location, so low-volume topics may return few points).
   - Recommendation: Test with real queries during implementation. Fall back to `inferGeoHubsFromTitle()` geocoding of DOC API results if GEO API coverage is thin.

4. **Kindness baseline regeneration frequency**
   - What we know: Baseline points should feel "alive" but not overwhelm the map.
   - What's unclear: Optimal refresh interval and number of baseline points.
   - Recommendation: Regenerate every 5 minutes with 50-80 points. Jitter positions slightly each cycle so the map feels dynamic. Use a deterministic seed per city so points don't jump randomly.

## Sources

### Primary (HIGH confidence)
- `src/components/DeckGLMap.ts` -- all layer creation patterns, pulse animation, tooltip, legend, layer toggles (2400+ lines, exhaustively reviewed)
- `src/types/index.ts` line 486-526 -- `MapLayers` interface definition
- `src/config/panels.ts` -- all variant layer configs (FULL, TECH, FINANCE, HAPPY, + mobile)
- `src/config/variants/happy.ts` -- happy variant panel and layer defaults
- `src/services/gdelt-intel.ts` -- GDELT DOC API client, positive topic queries, positive pipeline
- `src/services/positive-classifier.ts` -- positive content category classification
- `src/services/geo-hub-index.ts` -- keyword-based geocoding (80+ locations)
- `src/services/sentiment-gate.ts` -- ML sentiment filtering pipeline
- `src/services/population-exposure.ts` -- WorldPop population density data
- `server/worldmonitor/unrest/v1/list-unrest-events.ts` -- GDELT GEO API usage pattern (GeoJSON parsing)
- `src/App.ts` lines 3543-3634 -- happy variant multi-stage pipeline wiring

### Secondary (MEDIUM confidence)
- [GDELT GEO 2.0 API](https://blog.gdeltproject.org/gdelt-geo-2-0-api-debuts/) -- GeoJSON output with coordinates, "near" parameter, 3-month rolling window
- [GDELT DOC 2.0 API](https://blog.gdeltproject.org/gdelt-doc-2-0-api-debuts/) -- confirmed NO geographic coordinates in output
- [deck.gl ScatterplotLayer docs](https://deck.gl/docs/api-reference/layers/scatterplot-layer) -- `radiusScale`, `stroked`, `filled` properties for pulse effect

### Tertiary (LOW confidence)
- GoFundMe API -- partner-only access, no public trending endpoint confirmed. Needs validation.
- Change.org API -- Google Groups thread indicates ongoing deprecation. Needs validation.
- [GoFundMe Scraper (Apify)](https://apify.com/jupri/gofundme/api) -- third-party scraper exists but reliability unverified

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- all libraries already installed and used in the codebase
- Architecture: HIGH -- every pattern directly observed in existing DeckGLMap.ts code (2400+ lines reviewed)
- Pitfalls: HIGH -- identified from actual codebase patterns and known GDELT behavior
- Kindness data sources: MEDIUM -- GoFundMe/Change.org API availability is uncertain; baseline approach is solid
- GDELT GEO positive coverage: MEDIUM -- works for protests but positive queries untested

**Research date:** 2026-02-23
**Valid until:** 2026-03-23 (30 days -- stable domain, existing libraries)
