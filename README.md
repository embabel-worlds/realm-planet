# realm-planet

The planet around your places: **live air quality + UV** (Open-Meteo) and
**recent earthquakes** (USGS), both keyed on coordinates.

Track a place anywhere in the world by name. A keyless geocoder resolves it
once and the coordinates are stored on a `PlanetPlace`, which then grows:

- `-[:HAS_AIR]->(:AirQuality)` — PM2.5, PM10, ozone, NO₂, US/EU AQI, UV index
- `-[:HAS_QUAKE]->(:Earthquake)` — M2.5+ within 400 km, last 30 days

so one line of Cypher answers the cross-cutting questions:

```cypher
MATCH (p:PlanetPlace)-[:HAS_AIR]->(a:AirQuality)
RETURN p.name, toFloat(a.usAqi) AS aqi ORDER BY aqi DESC LIMIT 1
```

The realm owns its places, so it works **installed on its own** — nothing else
to install first. Installed beside
[realm-uk-streets](https://github.com/embabel-worlds/realm-uk-streets), every
`UkPlace` you watch there grows the same `HAS_AIR` and `HAS_QUAKE` readings,
with nothing to re-enter.

`AirQuality` and `Earthquake` are **virtual** — fetched per query, cached 30
minutes, gone at rollback. To join them onto another coordinate-bearing type,
copy a `virtualJoins` entry in `types/planet.yml` and change `anchorLabel`.

## Adding a place

In the **Planet Pulse** app, type a place name and press *Track*. From chat or
code:

```javascript
const r = (await gateway.openMeteoGeo.searchPlaces({ name: 'Lisbon', count: '5' })).results[0]
await gateway.repository.createEntry({ type: 'PlanetPlace', data: {
  name: 'Lisbon', latitude: r.latitude, longitude: r.longitude,
  country: r.country, countryCode: r.country_code, region: r.admin1, timezone: r.timezone,
}})
```

## What ships

| Piece | What it does |
|---|---|
| `apis/` | Vendored OpenAPI for Open-Meteo air quality, Open-Meteo geocoding and the USGS catalog — keyless, no accounts |
| `producers/planet.yml` | `airNow` and `quakesNear`, composite-key producers (`latitude`+`longitude`) |
| `types/planet.yml` | The `PlanetPlace` anchor; `AirQuality` and `Earthquake` with `virtualJoins` onto `PlanetPlace` and `UkPlace` |
| `views/planet.yml` | `AirAtMyPlaces`, `HighestUv`, `QuakesNearMyPlaces`, `ShakiestPlace` |
| `skills/planet/` | The chat skill: tracking a place, bands, gotchas, never-invent-coordinates |
| `apps/planet-pulse.html` | **Planet Pulse** — track places by name; AQI gauges and an animated seismic radar per place, with a global significant-quakes ticker |

No API keys. Works the moment it is installed.

## License

Apache-2.0
