# Stridemap
StrideMap is a web application that allows users to generate, analyse, and discover running routes in Brisbane based on distance and elevation preferences. Users can specify start and end locations, desired distance, elevation gain, and route type (loop or point-to-point). The app leverages OpenStreetMap and DEM from OpenTopography data, PostGIS, and pgRouting for route generation and analysis.

<img width="908" alt="Screenshot 2025-05-22 at 12 15 19 am" src="https://github.com/user-attachments/assets/b46184c8-d5cb-4370-a48e-d02055ec70af" />

# Features
* Custom Route Generation: Generate running routes based on user-specified distance and elevation gain.
* Loop and Point-to-Point Routes: Supports both looped and point-to-point routes.
* Elevation Analysis: coloru-coded elevation changes displayed on the map for analysis.
* Interactive Map: View generated routes on an interactive map with detailed statistics above.
* Location Autocomplete: Search and select start/end locations with autocomplete.
* Modern, responsive UI design

# Tech Stack
## Frontend: 
* React, Leaflet.js
## Backend: 
* Node.js, Express.js
## Database: 
* PostgreSQL, PostGIS, pgRouting
## Other: 
* OpenStreetMap data, OpenTopography DEM, Axios
