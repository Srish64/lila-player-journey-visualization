# LILA BLACK — Player Journey Visualization Tool
## Architecture Document

---

### Tech Stack & Why

**Frontend: Single-file HTML/CSS/JS (no framework)**

I chose a self-contained HTML file over React/Streamlit for one key reason: *zero deployment friction*. A Level Designer can open it directly in any browser — no server, no npm, no Python environment. Drop the file, double-click, done.

- **Canvas API** for rendering paths, markers, and heatmaps — far faster than SVG for 18K+ coordinate points
- **Vanilla JS** — no build step, no dependencies, fully auditable
- **Google Fonts CDN** — only external dependency (Share Tech Mono + Rajdhani for the tactical HUD aesthetic)

---

### Data Pipeline: Raw Parquet → What You See

```
player_data/*.nakama-0  (Apache Parquet, 1,243 files)
         │
         ▼
  Python parser (binary inspection)
  - Reads parquet files without pyarrow (unavailable)
  - Extracts schema: user_id, match_id, map_id, x, y, z, ts, event
  - Detects bots via numeric user_id prefix (e.g. "1440" vs UUID)
  - Extracts (x, z) coordinate pairs using Thrift field pattern: 0x18 0x04 {float32}
  - Converts world coords → minimap pixels using README formula:
      u = (x − origin_x) / scale
      v = (z − origin_z) / scale
      pixel_x = u × 512, pixel_y = (1−v) × 512
         │
         ▼
  matches_final.json (~1.1 MB)
  Structure: { match_id: { map_id, day, players: [{user_id, is_bot, events, coords}] } }
         │
         ▼
  HTML tool (self-contained, ~2.1 MB with embedded minimaps)
  - Minimap images resized 4320px → 512px (PIL) and base64-encoded inline
  - All 796 matches × 3 maps pre-indexed in JS memory
  - Canvas renders on filter change (no server round-trips)
```

**Why this approach?** The dataset is ~1,243 files totaling ~10 MB. Embedding everything in one HTML keeps the tool fully offline and shareable via a single link (e.g. GitHub Pages, Dropbox, any file host). A backend would add latency and infra complexity for no gain at this data scale.

---

### Key Decisions & Trade-offs

| Decision | Why | Trade-off |
|---|---|---|
| No pyarrow → custom binary parser | pyarrow unavailable in env; Thrift pattern `0x18 0x04 {float}` is reliable | Only extracts x/z/events; skips full row-level ts alignment |
| Embed all data in HTML | Zero-dependency deployment | 2.1 MB file; not suitable for >10K files |
| 512×512 canvas | Fast rendering, good clarity | Less sharp than native 4320px originals |
| Pre-compute pixel coords | Instant render on filter | Fixed to current coordinate formula |
| Up to 50 matches in "Select All" | Prevents canvas overload | Need to manually select more |
| Playback as % of coords (not time) | ts values encode match-elapsed-ms but not ordered consistently within a file | Not a true time-ordered replay |

---

### Coordinate Mapping (Key Nuance)

The `y` column in the data is **elevation**, not a map axis. For 2D minimap rendering:

```
Use x (horizontal) and z (depth) — NOT y
```

Map configs from README:
| Map | Scale | Origin X | Origin Z |
|---|---|---|---|
| AmbroseValley | 900 | −370 | −473 |
| GrandRift | 581 | −290 | −290 |
| Lockdown | 1000 | −500 | −500 |

---

### Bot Detection

Per README and filename convention:
- **Human**: `user_id` is a UUID (e.g. `f4e072fa-b7af-4761...`)
- **Bot**: `user_id` is numeric (e.g. `1440`, `382`)

Verified this against event types — bots generate `BotPosition`, `BotKill`, `BotKilled`; humans generate `Position`, `Kill`, `Killed`, `KilledByStorm`, `Loot`.

---

### What I'd Do Differently With More Time

1. **True timestamp playback** — Parse the `ts` column properly (int64 ms) to animate matches in real time. Currently playback is position-count-based.

2. **Backend + streaming** — For the full 5-day dataset with ~89K events, a lightweight FastAPI backend + DuckDB would enable SQL-level filtering and serve data on demand rather than loading everything upfront.

3. **More granular event placement** — Currently events (Kill, Loot, etc.) are shown at the player's *last* position. With proper row-level parsing, each event would appear at its exact world coordinate.

4. **Cluster analysis** — Identify hot-drop zones and storm-death clusters automatically using DBSCAN, surfaced as annotations on the map for level designers.

5. **Multi-match comparison** — Side-by-side view for A/B comparing two map versions or date ranges.

6. **Persistent URL state** — Encode selected map/date/matches in URL hash so designers can share specific views.

---

*Data: February 10–14, 2026 | Maps: AmbroseValley, GrandRift, Lockdown | Tool built with vanilla JS + Canvas API*
