# Web Application Project Documentation
## 1. Sridemap Overview
### Motivation
- What problem does this web application solve?
StrideMap was developed to solve a practical yet often overlooked problem: *the difficulty of planning outdoor running or walking routes that meet specific distance and elevation goals*. Existing tools like Strava and Google Maps either rely on manually drawn paths or provide limited route suggestions, often without real-time feedback on terrain or elevation. For users with structured training goals—such as hill intervals, distance progression, or taper runs—this lack of control can lead to inefficient and frustrating planning.


- Why is this application useful in a real-world scenario?
The motivation for this application was personal: while preparing for a 25 km hilly trail run, I found that even with access to varied terrain, it was surprisingly difficult to generate outdoor routes that matched my training program's elevation and distance requirements. This experience highlighted the need for a system that supports *custom, terrain-aware, goal-driven route planning*.

### Web Application Functions
StrideMap is a web-based application with the following key functionalities:

- **Custom Route Generation**  
  Users can request loop or point-to-point routes that match specific distance and elevation criteria. The backend generates multiple candidates and returns the best route based on a multi-dimensional scoring system.

- **Interactive Mapping**  
  A Leaflet map displays the generated route, with elevation visualised as a colour gradient along the path. Start and end markers are clearly highlighted.

- **User Input Controls**  
  A sidebar allows users to input start/end locations, set a target distance, and choose elevation preferences (e.g. prioritise distance, elevation, or balance both).

- **Real-Time Statistics**  
  Once a route is generated, the frontend displays its total distance, elevation gain, and estimated duration.

- **Geocoding Support**  
  Users can type in place names, which are geocoded using OSM’s `planet_osm_point` data and snapped to the nearest walkable node.

---

**Describe how high-dimensional queries are used in the application:**

StrideMap performs high-dimensional queries that combine *geospatial*, *graph*, and *elevation* dimensions. Specifically:
- **Spatial Filtering**: Paths are filtered to include only pedestrian-friendly segments using OSM tags.
- **Network Graph Traversal**: Routes are generated via pgRouting (for shortest paths) or custom heuristic searches (for loop and elevation-aware routing).
- **Elevation Matching**: Each node’s location is joined with DEM raster data to extract elevation via spatial joins.
- **Multi-Criteria Scoring**: Candidate routes are evaluated based on how closely they match the target distance and elevation, with dynamic tolerance adjustment depending on route length.

These dimensions are integrated in real-time to support *interactive, user-driven route planning*, demonstrating the power of spatial databases in high-dimensional querying scenarios.


## Queries Implemented

### Query 1: Nearest Node Snap  
**Task Description:**  
Snap user-entered coordinates to the nearest walkable network node. This ensures that routing starts and ends at valid, connected locations within the main graph component. This is a core function of any route generation request.

**Real-World Application:**  
When a user selects a location or enters a place name, it must be aligned with a node in the graph for routing algorithms to work correctly.

**SQL Query:**
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

### Query 2: Shortest Point-to-Point Route (Dijkstra)

**Task Description:**  
Return the shortest walkable path between two snapped nodes using pgRouting.

**Real-World Application:**  
Used for generating direct routes between different start and end points.

**SQL Query:**
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

### Query 3: Elevation Join (DEM Raster)

**Task Description:**  
Assign elevation values to route points using a spatial join with the raster DEM layer.

**Real-World Application:**  
Enables elevation-aware route scoring, elevation profile construction, and elevation gain calculation.

**SQL Query:**
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

### Query 4: Loop Route Generation

**Task Description:**  
Generate a loop route that starts and ends at the same point, with a target distance and optional elevation constraints.

**Real-World Application:**  
Used when users want to run a loop route from a single location, common for training runs.

**SQL Query:**
```sql
WITH RECURSIVE loop_candidates AS (
  SELECT 
    path,
    ST_Length(ST_Transform(ST_Collect(way), 3857)) as length,
    SUM(CASE 
      WHEN e2.elevation > e1.elevation 
      THEN e2.elevation - e1.elevation 
      ELSE 0 
    END) as elevation_gain
  FROM pgr_ksp(
    'SELECT id, source, target, cost, reverse_cost FROM walkable_ways_main',
    :start_node, :start_node, 3, false
  ) r
  JOIN walkable_ways_main w ON r.edge = w.id
  JOIN node_elevation_raster e1 ON w.source = e1.id
  JOIN node_elevation_raster e2 ON w.target = e2.id
  GROUP BY path
)
SELECT *
FROM loop_candidates
WHERE 
  length BETWEEN :min_distance AND :max_distance
  AND elevation_gain BETWEEN :min_elevation AND :max_elevation
ORDER BY 
  ABS(length - :target_distance) + 
  ABS(elevation_gain - :target_elevation)
LIMIT 1;
```

**Explanation of Variables:**
- `:start_node`: Node ID to start/end the loop
- `:min_distance`, `:max_distance`: Acceptable distance range
- `:min_elevation`, `:max_elevation`: Acceptable elevation gain range
- `:target_distance`, `:target_elevation`: Ideal distance and elevation

**Unexpected Value Handling:**
- If no suitable loop is found, falls back to out-and-back route generation
- Handles disconnected components by checking main graph membership
- Implements dynamic tolerance based on route length
