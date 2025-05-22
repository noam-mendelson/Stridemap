# Stridemap
StrideMap is a web application that allows users to generate, analyse, and discover running routes in Brisbane based on distance and elevation preferences. Users can specify start and end locations, desired distance, elevation gain, and route type (loop or point-to-point). The app leverages OpenStreetMap and DEM from OpenTopography data, PostGIS, and pgRouting for route generation and analysis.

<p align="center">
  <!-- Replace with an actual screenshot or GIF committed under docs/ -->
  <img src="https://github.com/user-attachments/assets/b46184c8-d5cb-4370-a48e-d02055ec70af" width="700" alt="StrideMap route example">
</p>

# Features
* Custom Route Generation: Generate running routes based on user-specified distance and elevation gain.
* Loop and Point-to-Point Routes: Supports both looped and point-to-point routes.
* Elevation Analysis (optional): colour-coded elevation changes displayed on the map for analysis.
* Interactive Map: View generated routes on an interactive map with detailed statistics above.
* Location Autocomplete: Search and select start/end locations with autocomplete.
* Modern, responsive UI design

# Tech Stack

| Layer      | Technologies & Libraries                                                          |
|------------|-----------------------------------------------------------------------------------|
| Frontend   | React 18, Leaflet.js (react-leaflet), Axios                                       |
| Backend¹   | Node.js (Express), pg (PostgreSQL driver), dotenv, RESTful API                |
| Database   | PostgreSQL 13+, PostGIS, pgRouting, OpenStreetMap data, OpenTopography DEM        |


## Prerequisites

```bash
# Ubuntu / Debian
sudo apt update
sudo apt install postgresql-13 postgresql-13-postgis-3 postgresql-13-pgrouting \
                 osm2pgsql gdal-bin build-essential nodejs npm
# macOS (Homebrew)
brew install postgresql@13 postgis pgrouting osm2pgsql gdal node
brew services start postgresql@13
```

Verify versions:

```bash
node -v
psql --version
psql -d postgres -c "SELECT postgis_version(), pgr_version();"
```

---
    
## Installation and Setup

### 0. Unpack the Project
Unzip the project folder and open a terminal in the project root directory.

## 1. Database Setup
#### Install PostgreSQL and PostGIS
```bash
# check course resources for installation instructions
```

### Create Database and enable extensions
Create a PostgreSQL database (e.g., stridemap) and enable PostGIS
```sql
-- Create the database
CREATE DATABASE stridemap;

-- Connect to the database
\c stridemap

-- Enable PostGIS and pgRouting
CREATE EXTENSION postgis;
CREATE EXTENSION pgrouting;
```
---

## 2. Import OpenStreetMap (OSM) Data

**a. Download OSM Data**

* Download a `.osm.pbf` file for your region (e.g., Brisbane) from https://download.geofabrik.de/australia-oceania/australia.html 

**b. Download DEM Data**

* Visit [OpenTopography](https://opentopography.org/) and click on **Find Data**.
* Choose a dataset such as:
  - **SRTM 1 Arc-Second Global** (≈30m resolution) 
* Use the **bounding box** tool or manually input coordinates matching your OSM region. For Brisbane region, for example:
```
North −27.2 | South −27.6 | East 153.2 | West 152.9
```

* Select **GeoTIFF** as the output format and download the file.

**c. Import OSM and DEM Data into PostgreSQL**

* Use [osm2pgsql](https://osm2pgsql.org/) to import the data:

```bash
osm2pgsql -d stridemap -U youruser --create --slim -G --hstore -C 2000 --number-processes 4 -S /usr/share/osm2pgsql/default.style /path/to/brisbane.osm.pbf
```

* This will create tables like `planet_osm_point`, `planet_osm_line`, etc.

* To import DEM (elevation) data, use [`raster2pgsql`](https://postgis.net/docs/using_raster_dataman.html):

```bash
raster2pgsql -s 4326 -I -C /path/to/your_dem.tif -F public.dem_raster | psql -d stridemap -U youruser
```


* This will create the `dem_raster` table in your database, storing elevation data for the specified region as raster tiles.
---

## 3. Prepare Routing Tables

```sql
-- Example: Create a simplified walkable ways table
CREATE TABLE walkable_ways_main AS
SELECT
  osm_id,
  way,
  source,
  target,
  ST_Length(way::geography) AS length_m,
  cost, reverse_cost
FROM planet_osm_line
WHERE highway IN ('footway', 'path', 'residential', 'living_street', 'track', 'unclassified', 'tertiary', 'secondary', 'primary')
  AND (foot IS NULL OR foot NOT IN ('no', 'private'));

-- Generate routing topology -  topology (0.00001 deg ≈ 1 m at Brisbane’s latitude)

SELECT pgr_createTopology('walkable_ways_main', 0.00001, 'way', 'osm_id');

```


### 4. Join DEM to Routing Nodes

```sql
CREATE TABLE node_elevation_raster AS
SELECT
  id,
  ST_Centroid(the_geom) AS geom,
  (ST_Value(rast, ST_Centroid(the_geom))) AS elevation
FROM walkable_ways_vertices_pgr, dem_raster
WHERE ST_Intersects(the_geom, rast);
```

---

### 5. Configure Environment Variables

Create a `.env` file in your backend directory:
```env
PGUSER=youruser
PGPASSWORD=yourpassword
PGHOST=localhost
PGPORT=5432
PGDATABASE=stridemap
```

---

#### 6. Backend Setup
1. Open a terminal and navigate to the backend directory:
   ```bash
   cd backend
   ```
2. Install dependencies:
   ```bash
   npm install
   ```
3. Start the backend server:
   ```bash
   npm start
   ```
   The backend API will run at `http://localhost:5050` (or as configured).

#### 7 Frontend Setup
1. Open a new terminal window and navigate to the frontend directory:
   ```bash
   cd frontend
   ```
2. Install dependencies:
   ```bash
   npm install
   ```
3. Start the frontend development server:
   ```bash
   npm start
   ```
   The frontend will be available at `http://localhost:3000` by default.

---

Install and Run Backend & Frontend

#### Backend
```bash
cd backend
npm install
npm start
```

#### Frontend
```bash
cd ../frontend
npm install
npm start
```
- The frontend will be available at `http://localhost:3000`
- The backend API will run at `http://localhost:5050` (or as configured)

---

## Usage

1. Open the frontend in your browser (http://localhost:3000).
2. Enter a start location (and optionally an end location).
3. Specify your desired distance and elevation gain.
4. Select whether colour-coded polyline is wanted.
5. Click "Generate Route" to view the route and stats on the map.

---

## API Endpoints

### `POST /api/generate-route`
Generates a running route based on user preferences.

**Parameters (JSON body):**
- `start` (string): Start location name
- `end` (string, optional): End location name (leave blank for loop route)
- `target_distance` (number, optional): Desired distance in km
- `elevation_target` (number, optional): Desired elevation gain in meters
- `profile` (string, optional): Elevation profile category
- `priority` (string, optional): Route priority (distance, elevation, or equal)
- `colour-coded route` (optional): Display elevation on map

**Response Example:**
```js
{
  "geometry": [
    [153.02, -27.47],
    [153.03, -27.48],
    // more coordinates
  ],
  "summary": {
    "distance": 12.0,
    "elevation_gain": 200,
    "elevation_profile": [10, 12, 15 /* ... */]
  },
  "message": "Route generated successfully! Distance: 12.0 km, Elevation Gain: 200 m"
}
```
---
## Project Structure

```
├── backend/
│   ├── src/
│   │   ├── routes/
│   │   │   └── routes.js         # Main backend API routes
│   │   ├── db.js                 # Database connection
│   │   └── app.js                # Express app entry point
│   ├── package.json
│   └── .env                      # Backend environment variables
├── frontend/
│   ├── public/
│   ├── src/
│   │   ├── components/
│   │   │   ├── RouteGenerator.js # Main route generation UI
│   │   │   └── RouteMap.js       # Map display component
│   │   ├── index.js              # React entry point
│   │   └── index.css             # Global styles
│   └── package.json
└── README.md
```

---
## Troubleshooting

- **Database connection errors:** Check your `.env` file and ensure PostgreSQL is running.
- **No routes found:** Try different start/end locations or adjust your distance/elevation preferences.
- **Missing map tiles:** Ensure you have internet access for Leaflet/OpenStreetMap tiles.
- **PostGIS and pgRouting** extensions must be enabled


| Symptom                                                              | Likely Cause                       | Fix / Command                                       |
|----------------------------------------------------------------------|------------------------------------|-----------------------------------------------------|
| `psql: FATAL: role "youruser" does not exist`                        | DB user not created                | `createuser -s youruser`                            |
| `ERROR:  function pgr_createtopology(...) does not exist`            | pgRouting extension not enabled    | `CREATE EXTENSION pgrouting;`                       |
| Blank map tiles                                                      | Offline or wrong tile URL          | Check Internet connection / Leaflet tile provider   |
| “No viable route found”                                              | Sparse network near start location | Move start pin, lower distance, or import larger OSM area |
| `ERROR:  could not load library "postgis-3"`                         | PostGIS missing or wrong version   | `CREATE EXTENSION postgis;` (or reinstall PostGIS)  |
| `ERROR:  relation "walkable_ways_main" does not exist`               | Routing tables not created         | Re-run the **Prepare Routing Tables** SQL step      |

> **Tip:** Always confirm extensions are loaded with  
> ```sql
> SELECT postgis_version(), pgr_version();
> ```

## Acknowledgements

* OpenStreetMap © contributors  
* OpenTopography (SRTM 1 Arc-Second DEM)  
* Leaflet, Tailwind CSS, pgRouting, Node.js community packages

---

## References

* **PostGIS documentation:** <https://postgis.net/docs/>  
* **pgRouting functions:** <https://docs.pgrouting.org/>  
* **osm2pgsql manual:** <https://osm2pgsql.org/doc/manual.html>  
* **GDAL raster import:** <https://gdal.org/>  





  
