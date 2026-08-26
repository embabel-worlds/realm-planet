# realm-planet

The planet around your places: **live air quality + UV** (Open-Meteo) and
**recent earthquakes** (USGS), both keyed on coordinates — so this realm joins
onto anything carrying a `latitude` and `longitude`.

Installed beside [realm-weather](https://github.com/embabel-worlds/realm-weather),
every tracked `WeatherLocation` grows:

- `-[:HAS_AIR]->(:AirQuality)` — PM2.5, PM10, ozone, NO₂, US/EU AQI, UV index
- `-[:HAS_QUAKE]->(:Earthquake)` — M2.5+ within 400 km, last 30 days

so one line of Cypher answers the cross-cutting questions:

```cypher
MATCH (l:WeatherLocation)-[:HAS_AIR]->(a:AirQuality)
RETURN l.name, toFloat(a.usAqi) AS aqi ORDER BY aqi DESC LIMIT 1
```

Both types are **virtual** — fetched per query, cached 30 minutes, gone at
rollback. The realm persists nothing of its own: it is a pure enrichment layer.
To join air and quakes onto a different coordinate-bearing type, copy a
`virtualJoins` entry in `types/planet.yml` and change `anchorLabel`.

## What ships

| Piece | What it does |
|---|---|
| `apis/` | Vendored OpenAPI for Open-Meteo air quality and the USGS catalog — keyless, no accounts |
| `producers/planet.yml` | `airNow` and `quakesNear`, composite-key producers (`latitude`+`longitude`) |
| `types/planet.yml` | `AirQuality` and `Earthquake` with `virtualJoins` onto `WeatherLocation` |
| `views/planet.yml` | `AirAtMyPlaces`, `HighestUv`, `QuakesNearMyPlaces`, `ShakiestPlace` |
| `skills/planet/` | The chat skill: bands, gotchas, never-invent-coordinates |
| `apps/planet-pulse.html` | **Planet Pulse** — AQI gauges and an animated seismic radar per place, with a global significant-quakes ticker |

No API keys. Works the moment it is installed.

## License

Apache-2.0
