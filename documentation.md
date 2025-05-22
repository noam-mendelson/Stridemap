# Web Application Project Documentation

## 1. StrideMap Overview

### Motivation  
**What problem does this web application solve?**  
StrideMap addresses *the difficulty of planning outdoor running or walking routes that meet specific distance and elevation goals*. Existing tools (e.g., Strava, Google Maps) either rely on manually drawn paths or provide limited suggestions without real-time feedback on terrain. For structured training—hill intervals, distance progression, taper runs—this lack of control leads to inefficient, frustrating planning.

**Why is this application useful in a real-world scenario?**  
While preparing for a 25 km hilly trail run, I discovered how hard it was to generate outdoor routes matching both elevation and distance requirements. StrideMap fills this gap by supporting *custom, terrain-aware, goal-driven route planning* that lets users specify:

- Start / end locations  
- Target distance  
- Desired elevation gain  
- Terrain preference (via walkable-path filtering)

By integrating spatial (lat/long), topographic (elevation), and network-connectivity data, StrideMap performs *real-time, multi-criteria* route generation that adapts to the user’s goals.

---

### Web Application Functions  

- **Custom Route Generation**  
  Loop or point-to-point routes that match user-defined distance and elevation constraints, selected via a multi-dimensional scoring system.

- **Interactive Mapping**  
  Leaflet map with the route polylined in an elevation-gradient colour scheme; distinct start/end markers.

- **User Input Controls**  
  Sidebar form for start/end locations, distance target, and elevation priority (distance-first, elevation-first, or balanced).

- **Real-Time Statistics**  
  Displays total distance, elevation gain, and estimated duration as soon as a route is generated.

- **Geocoding Support**  
  Place names are geocoded via `planet_osm_point` and snapped to the nearest walkable node.

---

### High-Dimensional Query Usage

StrideMap combines **geospatial**, **graph**, and **elevation** dimensions in every route request:

| Dimension             | Operation                                                                  |
|-----------------------|-----------------------------------------------------------------------------|
| Spatial Filtering     | Exclude non-pedestrian segments using `highway` tags.                       |
| Network Traversal     | Use pgRouting Dijkstra / custom heuristic search to explore the graph.      |
| Elevation Matching    | Join each node to the DEM raster (`ST_DWithin` + `ST_Value`).               |
| Multi-Criteria Scoring| Weight distance and elevation deviations with dynamic tolerances.           |

These integrated queries power *interactive, user-driven* route planning while demonstrating spatial-database efficacy for high-dimensional data problems.

---

## 5. Queries Implemented

### Query 1 – Nearest Node Snap  
**Task:** Snap user-entered coordinates to the nearest walkable network node. This ensures that routing starts and ends at valid, connected locations within the main graph component. This is a core function of any route generation request.
**Real-World Use:** When a user selects a location or enters a place name, it must be aligned with a node in the graph for routing algorithms to work correctly.

```sql
SELECT id
FROM walkable_ways_vertices_pgr
ORDER BY the_geom <-> ST_SetSRID(ST_MakePoint(:lng, :lat), 4326)
LIMIT 1;

```

**Explanation of Variables:**
- `:lng`, `:lat`: User input longitude and latitude values.
- `ST_SetSRID(...)`: Ensures geometry is interpreted using WGS84 (SRID 4326).
- `<->`: KNN operator used with GiST index to efficiently find the nearest node.

**Unexpected Value Handling:**
- If no node is found within a reasonable bounding box, the system returns a frontend error prompting the user to adjust location.
- Coordinates outside the map bounds are caught by input validation.

### Query 2 – Shortest Point-to-Point Route (Dijkstra)  
**Task:** Return the shortest walkable path between two snapped nodes using pgRouting.  
**Real-World Use:** Used for generating direct routes between different start and end points.

```sql
SELECT * FROM pgr_dijkstra(
  'SELECT id, source, target, cost, reverse_cost FROM walkable_ways_main',
  :start_node, :end_node, false
);
```

**Explanation of Variables:**
- `:start_node`, `:end_node`: Integer IDs of nodes returned from the snapping query.
- `cost`, `reverse_cost`: Precomputed traversal cost (e.g. distance in metres).

**Unexpected Value Handling:**
- If no path is found (e.g. due to disconnected components), the backend falls back to an alternative strategy or returns a user-facing warning.

---

### Query 3 – Elevation Join (DEM Raster)  
**Task:** Assign elevation values to route points using a spatial join with the raster DEM layer.  
**Real-World Use:** Enables elevation-aware route scoring, elevation profile construction, and elevation gain calculation.

```sql
SELECT rp.seq, ST_X(rp.geom) AS lon, ST_Y(rp.geom) AS lat, ner.elevation
FROM (
  SELECT (ST_DumpPoints(ST_LineMerge(ST_Union(way)))).geom AS geom, row_number() OVER () AS seq
  FROM walkable_ways_main
  WHERE id = ANY(:route_edge_ids)
) AS rp
LEFT JOIN node_elevation_raster ner
  ON ST_DWithin(rp.geom, ner.geom, 0.0001);
```

**Explanation of Variables:**
- `:route_edge_ids`: List of edge IDs representing a candidate path.
- `ST_DWithin(...)`: Ensures elevation values are joined only when the node lies within 10 m of the DEM cell.

**Unexpected Value Handling:**
- If no elevation is found for a point, a fallback elevation of 0 or null is assigned. These are filtered or smoothed in post-processing to prevent elevation "spikes."

---

### Query 4 – Loop Route Generation (Multi-Dimensional Optimisation)  
**Task:** Generate a loop route that starts and ends at the same point, with a target distance and optional elevation constraints.  
**Real-World Use:** Used when users want to run a loop route from a single location, common for training runs.

```sql
WITH RECURSIVE loop_candidates AS (
  SELECT 
    path,
    ST_Length(ST_Transform(ST_Collect(way), 3857)) as length_m,
    SUM(CASE 
      WHEN e2.elevation > e1.elevation 
      THEN e2.elevation - e1.elevation 
      ELSE 0 
    END) as elev_gain
  FROM pgr_ksp(
    'SELECT id, source, target, cost, reverse_cost FROM walkable_ways_main',
    :start_node, :start_node, 3, false
  ) r
  JOIN walkable_ways_main w ON r.edge = w.id
  JOIN walkable_ways_vertices_pgr e1 ON w.source = e1.id
  JOIN walkable_ways_vertices_pgr e2 ON w.target = e2.id
  GROUP BY path
)
SELECT *
FROM loop_candidates
WHERE 
  length_m BETWEEN :min_dist AND :max_dist
  AND elev_gain BETWEEN :min_elev AND :max_elev
ORDER BY 
  ABS(length_m - :target_dist) + 
  ABS(elev_gain - :target_elev)
LIMIT 1;
```

**Explanation of Variables:**
- `:start_node`: Node ID to start/end the loop.
- `:min_dist`, `:max_dist`: Acceptable distance range.
- `:min_elev`, `:max_elev`: Acceptable elevation gain range.
- `:target_dist`, `:target_elev`: Ideal distance and elevation for scoring.

**Unexpected Value Handling:**
- If no suitable loop is found, the backend falls back to out-and-back route generation.
- Handles disconnected components by checking main graph membership.
- Implements dynamic tolerance based on route length.
