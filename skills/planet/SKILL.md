---
name: planet
description: Air quality, UV and earthquakes for any place in the world, and around the places the user tracks. Activate for "air quality in X", "is the air bad", "UV index", "any earthquakes near X / near my places", "is it safe to run outside", "track X" / "keep an eye on X". Every source is keyless — never tell the user this needs an API key, and never send them to install another realm first.
---

# Planet

Three keyless APIs and one anchor the realm owns, `PlanetPlace`. All calls go
through `gateway.<ns>.<method>(args)` from inside `code_mode` — never as
top-level tools.

| Surface | Shape |
|---|---|
| `gateway.openMeteoGeo.searchPlaces({ name, count, language })` | Place name → candidates with coordinates. `results` is ABSENT when nothing matches. |
| `gateway.openMeteoAir.airQuality({ latitude, longitude, current, timezone })` | Current air quality and UV. Coordinates are **strings**. |
| `gateway.usgs.quakesNear({ format: 'geojson', latitude, longitude, maxradiuskm, minmagnitude })` | Quakes within a radius; omit `starttime` for the default last-30-days window. |
| `gateway.usgs.significantQuakes({})` | The month's significant quakes, worldwide. |
| `gateway.view.run({ name: 'AirAtMyPlaces' })` | Air and UV at every tracked PlanetPlace, worst first. Also: `HighestUv`, `QuakesNearMyPlaces`, `ShakiestPlace`. |

## Tracking a place

Resolve the name ONCE, store the coordinates on the place — every join keys
on them:

```javascript
const hits = (await gateway.openMeteoGeo.searchPlaces({ name: 'Lisbon', count: '5', language: 'en' })).results || []
// 'Lisbon' is also a village in Ohio: show country + admin1 and let the user
// pick, or take the most populous candidate when they clearly mean the city.
const r = hits.sort((a, b) => (b.population || 0) - (a.population || 0))[0]
await gateway.repository.createEntry({ type: 'PlanetPlace', data: {
  name: 'Lisbon', latitude: r.latitude, longitude: r.longitude,
  country: r.country, countryCode: r.country_code, region: r.admin1, timezone: r.timezone,
}})
```

If `results` is missing, say the place was not found — **never invent
coordinates from memory**. A wrong lat/lon returns a real, confident-looking
reading for the wrong place, and nothing about the answer looks wrong.

For a one-off question ("air quality in Delhi?") geocode and call
`airQuality` directly; only store a PlanetPlace when the user wants to keep
watching it.

## UK places from realm-uk-streets

When realm-uk-streets is installed, its `UkPlace` anchors carry the same
`HAS_AIR` and `HAS_QUAKE` joins — do not duplicate them as PlanetPlaces. The
saved views read PlanetPlace only (a query naming `UkPlace` is refused on a
world without that realm), so for UK places MATCH the label directly:

```cypher
MATCH (u:UkPlace)-[:HAS_AIR]->(a:AirQuality)
RETURN u.name, toFloat(a.usAqi) AS aqi ORDER BY aqi DESC
```

To rank both kinds together, UNION inside a subquery — each branch must name
its OWN variables, or the engine refuses the query:

```cypher
CALL {
  MATCH (p:PlanetPlace)-[:HAS_AIR]->(a:AirQuality)
  RETURN p.name AS place, toFloat(a.usAqi) AS aqi
  UNION ALL
  MATCH (u:UkPlace)-[:HAS_AIR]->(ua:AirQuality)
  RETURN u.name AS place, toFloat(ua.usAqi) AS aqi
}
RETURN place, aqi ORDER BY aqi DESC
```

## Reading the numbers

- `us_aqi`: 0–50 good, 51–100 moderate, 101–150 unhealthy for sensitive
  groups, 151+ unhealthy. Use these words, not the bare number.
- `uv_index`: 3+ protection advised, 8+ extreme.
- Quakes: `properties.time` is epoch **milliseconds**; `features` may be
  empty — quiet ground is a normal, good answer, not a failure.

## Or just ask the graph

The joins mean one Cypher line answers the cross-cutting questions:

```cypher
MATCH (p:PlanetPlace)-[:HAS_AIR]->(a:AirQuality)
RETURN p.name, toFloat(a.usAqi) AS aqi ORDER BY aqi DESC LIMIT 1
```

Projected values arrive as strings — always `toFloat()` before ordering.
