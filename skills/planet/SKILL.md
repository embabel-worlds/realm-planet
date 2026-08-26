---
name: planet
description: Air quality, UV and earthquakes for any coordinates, and around the places the user tracks. Activate for "air quality in X", "is the air bad", "UV index", "any earthquakes near X / near my places", "is it safe to run outside". Every source is keyless — never tell the user this needs an API key.
---

# Planet

Two keyless APIs over the user's tracked places. All calls go through
`gateway.<ns>.<method>(args)` from inside `code_mode` — never as top-level tools.

| Surface | Shape |
|---|---|
| `gateway.openMeteoAir.airQuality({ latitude, longitude, current, timezone })` | Current air quality and UV. Coordinates are **strings**. |
| `gateway.usgs.quakesNear({ format: 'geojson', latitude, longitude, maxradiuskm, minmagnitude })` | Quakes within a radius; omit `starttime` for the default last-30-days window. |
| `gateway.usgs.significantQuakes({})` | The month's significant quakes, worldwide. |
| `gateway.view.run({ name: 'AirAtMyPlaces' })` | Air and UV at every tracked place, worst first. Also: `HighestUv`, `QuakesNearMyPlaces`, `ShakiestPlace`. |

## Coordinates come from the graph or a geocoder — never from memory

Both APIs take coordinates, not place names. For the user's own places, read
them (they are stored resolved):

```javascript
const places = (await gateway.kg.query({
  cypher: "MATCH (l:WeatherLocation) RETURN l.name AS name, l.latitude AS lat, l.longitude AS lon",
})).rows
const air = await gateway.openMeteoAir.airQuality({
  latitude: String(places[0].lat), longitude: String(places[0].lon),
  current: 'pm2_5,us_aqi,european_aqi,uv_index', timezone: 'auto',
})
```

For a place the user only NAMED, geocode first if a geocoding surface is
available in this world; if none is, say you cannot resolve the place —
**never invent coordinates from memory**. A wrong lat/lon returns a real,
confident-looking reading for the wrong place, and nothing about the answer
looks wrong.

## Reading the numbers

- `us_aqi`: 0–50 good, 51–100 moderate, 101–150 unhealthy for sensitive
  groups, 151+ unhealthy. Use these words, not the bare number.
- `uv_index`: 3+ protection advised, 8+ extreme.
- Quakes: `properties.time` is epoch **milliseconds**; `features` may be
  empty — quiet ground is a normal, good answer, not a failure.

## Or just ask the graph

The joins mean one Cypher line answers the cross-cutting questions:

```cypher
MATCH (l:WeatherLocation)-[:HAS_AIR]->(a:AirQuality)
RETURN l.name, toFloat(a.usAqi) AS aqi ORDER BY aqi DESC LIMIT 1
```

Projected values arrive as strings — always `toFloat()` before ordering.
