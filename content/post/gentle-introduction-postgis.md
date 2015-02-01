+++
Categories = ["Geospatial Analysis"]
Description = "A Gentle Introduction To Geospatial Analysis"
Tags = ["PostGIS", "PostgreSQL"]
date = "2015-02-01T10:39:49+01:00"
title = "A Gentle Introduction To Geospatial Analysis"

+++

This is meant as a gentle introduction into geospatial analysis.  
We're going to use OpenStreetMap datasets and work on PostgreSQL with the PostGIS extension.


## Requirements

For our geospatial analysis we need a few packages for importing, querying and exporting the datasets.

* PostgreSQL
* PostGIS
* osm2pgsql
* ogr2ogr

Install the requirements from your package manager.
For reference: my system is Debian GNU/Linux 8.0 (jessie):

```bash
aptitude install postgresql-9.4 postgis osm2pgsql gdal-bin
```


## Preparing The Database

As user postgres (i.e. sudo su postgres) create a new user playground, a new table playground and activate the PostGIS extension:

```bash
createuser playground
createdb --owner playground gis
psql --dbname=gis --command 'create extension postgis;'
```


## Importing OpenStreetMap Datasets

Geofabrik hosts OpenStreetMap dumps at http://download.geofabrik.de  
I highly recommend starting with a small dataset first. We're using sweden-latest.osm.pbf here.

Import the dataset into Postgres. This creates relations and fills the database appropriately for efficient querying:

```bash
osm2pgsql --create --database gis --latlong --slim --cache 4096 --username playground --password --number-processes 4 --multi-geometry --disable-parallel-indexing sweden-latest.osm.pbf
```

See the osm2pgsql manpage; you may want to change a few parameters e.g. the cache size.  

Hint: this is I/O bound from what I can tell (check iotop, htop), so your cpus won't help much after a few minutes.
In case you don't have a lot of memory you may want to temporarily lower the swappiness before importing:

```bash
sudo sysctl vm.swappiness=10
```

Warning: importing with the latlong switch means we're using the Spacial Reference System Id 4326 (WGS 84, EPSG:4326), therefore the units are in degree.
We have to cast the geometry type to the geography type if we want to work with meters. Just keep this in mind.

See http://wiki.openstreetmap.org/wiki/Osm2pgsql


## Exporting Query Results As GeoJSON

ogr2ogr is able to issue queries against PostgreSQL, generating GeoJSON's FeatureCollections and inserting your gemometry with description on the fly. See the example exports below for how this can be done. Other formats are available, too.

If you don't want to use ogr2ogr you have to assemble GeoJSON from primitives such as points and lines yourself.
You're then able to use the exported GeoJSON files as data source e.g. for Mapbox.

See http://www.gdal.org/ogr2ogr.html

In the following I'm using Mapbox for visualization (as long as my data limit is not reached).


## Resources For Working On The Database

Here are two good starting points for working on the database.

* Database schema: http://wiki.openstreetmap.org/wiki/Osm2pgsql/schema
* Map Features: http://wiki.openstreetmap.org/wiki/Map_Features

Scan the Map Features quickly for what is available.
In the following we're doing a few queries against those features.


## Queries Against Features And Polygons

Connect to the database:

```bash
psql --dbname=gis --username=playground
```

Let's see what kind of relations and columns are available:

```bash
\d
\d planet_osm_point
```

Helpful PostGIS functions:

* ST\_AsGeoJSON on the geometry returns geojson
* ST\_{XMin,YMin} on the geometry returns latlong
* ST\_DWithin, ST\_Distance and other for filtering and measurements

See http://postgis.net/docs/manual-2.1/reference.html

Hints for doing queries:

* Limit your query while exploring the dataset, but don't forget to unset the limit for exporting.
* The column's type is text in almost all cases, check for number '^\[0-9\]+$' and cast; in order by clauses, too
* use CTRL-A/CTRL+E to jump to the left/right in psql, CTRL+L to clear screen

Now let's do a few queries!


### All Airports

Let's search for all airports in the dataset.

* Scroll through Map Features, find aerialways: http://wiki.openstreetmap.org/wiki/Map_Features#Aerialway
* Aeroway tag: http://wiki.openstreetmap.org/wiki/Key:aeroway
* Aerodrome value: http://wiki.openstreetmap.org/wiki/Tag:aeroway%3Daerodrome

Warning: aerodrome is strictly speaking not only for airports. There are definitely a few missing, e.g. near Umeå, too.

Nodes with tags are stored in the planet\_osm\_point relation.

```sql
SELECT name,
       ST_AsGeoJSON(way) AS geojson
FROM planet_osm_point
WHERE aeroway='aerodrome'
ORDER BY name LIMIT 3;
```

```bash
        name        |                        geojson                         
--------------------+--------------------------------------------------------
 Ålleberg Airport   | {"type":"Point","coordinates":[13.6031192,58.1354088]}
 Älvsbyn Airport    | {"type":"Point","coordinates":[21.0611,65.6456985]}
 Anderstorp Airport | {"type":"Point","coordinates":[13.5993996,57.2641983]}
```

Nice! Now we either assemble a GeoJSON document manually from all the rows or we simply use ogr2ogr:

```bash
ogr2ogr -f "GeoJSON" airports.geojson -t_srs EPSG:4326 PG:"dbname='gis' user='playground' password='playground'" -s_srs EPSG:4326 -sql "select name, way from planet_osm_point where aeroway='aerodrome' order by name;"
```

ogr2ogr is quite clever and assembles GeoJSON from your geometries and description itself.

<iframe width='100%' height='500px' frameBorder='0' src='https://a.tiles.mapbox.com/v4/danieljh.l3jbdc17/attribution,zoompan,zoomwheel,geocoder,share.html?access_token=pk.eyJ1IjoiZGFuaWVsamgiLCJhIjoiTnNYb25JSSJ9.vYOnsuu1zeKcGW2nj0uJZw'></iframe>


### Largest Cities

Now what about the largest cities in the dataset.

* Places: http://wiki.openstreetmap.org/wiki/Places
* Place tag: http://wiki.openstreetmap.org/wiki/Key:place
* City: http://wiki.openstreetmap.org/wiki/Tag:place%3Dcity
* Population: http://wiki.openstreetmap.org/wiki/Key:population

Note: my dataset containes at least one entry where the population field has a value of ">50", i.e. not a number.

Keep this in mind in case you're doing sums, averages or casts in general.

```sql
SELECT sum(population::int) AS population
FROM planet_osm_point
WHERE population ~ '^[0-9]+$';
```

```bash
 population 
------------
  15339409
```

Hmmm... interesting. This should be around 9-10 million.

Now to the largest cities. The 'city' value for the place tag already excludes smaller towns and villages.

```sql
SELECT name,
       population::int,
       ST_AsGeoJSON(way) AS geojson
FROM planet_osm_point
WHERE place='city'
  AND population ~ '^[0-9]+$'
ORDER BY population::int DESC LIMIT 5;
```

```bash
   name    | population |                        geojson                         
-----------+------------+--------------------------------------------------------
 Stockholm |     829417 | {"type":"Point","coordinates":[18.0710935,59.3251172]}
 Göteborg  |     522259 | {"type":"Point","coordinates":[11.9670171,57.7072326]}
 Malmö     |     303873 | {"type":"Point","coordinates":[13.0001566,55.6052931]}
 Uppsala   |     128400 | {"type":"Point","coordinates":[17.64112,59.8594126]}
 Västerås  |     110877 | {"type":"Point","coordinates":[16.5463679,59.6110992]}
 ```

```bash
ogr2ogr -f "GeoJSON" cities.geojson -t_srs EPSG:4326 PG:"dbname='gis' user='playground' password='playground'" -s_srs EPSG:4326 -sql "select name, population::int), way from planet_osm_point where place='city' and population ~ '^[0-9]+\$' order by population::int desc;"
```

<iframe width='100%' height='500px' frameBorder='0' src='https://a.tiles.mapbox.com/v4/danieljh.l3jbk96g/attribution,zoompan,zoomwheel,geocoder,share.html?access_token=pk.eyJ1IjoiZGFuaWVsamgiLCJhIjoiTnNYb25JSSJ9.vYOnsuu1zeKcGW2nj0uJZw'></iframe>


### Municipalities

Now we want the city's boundaries e.g. the municipality as polygon.
The relation planet\_osm\_point only contains nodes with tags.
For the polygon data we have to query the relation planet\_osm\_polygon in addition.

To do this, we cross join the nodes with the polygons and filter the largest cities as above.
We also check that our city is inside the municipality polygon.

```sql
SELECT polygon.name,
       polygon.way
FROM planet_osm_point AS point
CROSS JOIN planet_osm_polygon AS polygon
WHERE point.place='city'
  AND polygon.admin_level = '7'
  AND ST_Contains(polygon.way, point.way)
  AND point.population ~ '^[0-9]+$'
ORDER BY point.population::int DESC LIMIT 3;
```

For admin\_level (7 is municipality) see http://wiki.openstreetmap.org/wiki/Key:admin_level#admin_level

The output consists of polygons around the largest cities.

    ogr2ogr -f "GeoJSON" boundaries.geojson -t_srs EPSG:4326 PG:"dbname='gis' user='playground' password='playground'" -s_srs EPSG:4326 -sql "select polygon.name, polygon.way from planet_osm_point as point cross join planet_osm_polygon as polygon where point.place='city' and polygon.admin_level = '7' and ST_Contains(polygon.way, point.way) and point.population ~ '^[0-9]+\$' order by point.population::int desc;"

<iframe width='100%' height='500px' frameBorder='0' src='https://a.tiles.mapbox.com/v4/danieljh.l3jc1lpa/attribution,zoompan,zoomwheel,geocoder,share.html?access_token=pk.eyJ1IjoiZGFuaWVsamgiLCJhIjoiTnNYb25JSSJ9.vYOnsuu1zeKcGW2nj0uJZw'></iframe>


### Combining Airports With Largest Cities

Now we want all airports that are within a distance X (in kilometers, 50km in this example query) around large cities.

Warning: we imported with the latlong switch, which means SRID 4326.
Therefore the units are in degrees. We cast to the geography type for working with meters.

Let's cross join cities with airports and then filter by distance.

```sql
SELECT city.name AS city,
       airport.name AS airport_name,
       ST_AsGeoJSON(airport.way) AS airport,
       ST_Distance(city.way::geography, airport.way::geography) AS distance
FROM planet_osm_point AS city
CROSS JOIN planet_osm_point AS airport
WHERE city.place='city'
  AND city.population ~ '^[0-9]+$'
  AND airport.aeroway='aerodrome'
  AND ST_DWithin(city.way::geography, airport.way::geography, 50000)
ORDER BY distance LIMIT 3;
```

```bash
    city     |      airport_name      |                        airport                         |    distance    
-------------+------------------------+--------------------------------------------------------+----------------
 Linköping   | Linköping City Airport | {"type":"Point","coordinates":[15.658056,58.4080416]}  | 1970.028258466
 Halmstad    | Halmstad Flygplats     | {"type":"Point","coordinates":[12.8216423,56.6865017]} |   2601.7448687
 Norrköping  | Norrköping Flygplats   | {"type":"Point","coordinates":[16.2469701,58.5859972]} | 3338.144166465
```

```bash
ogr2ogr -f "GeoJSON" nearest.geojson -t_srs EPSG:4326 PG:"dbname='gis' user='playground' password='playground'" -s_srs EPSG:4326 -sql "select city.name as city, airport.name as airport_name, airport.way as airport, ST_Distance(city.way::geography, airport.way::geography) as distance from planet_osm_point as city cross join planet_osm_point as airport where city.place='city' and city.population ~ '^[0-9]+\$' and airport.aeroway='aerodrome' and ST_DWithin(city.way::geography, airport.way::geography, 50000) order by distance;"
```

<iframe width='100%' height='500px' frameBorder='0' src='https://a.tiles.mapbox.com/v4/danieljh.l3jc7m3p/attribution,zoompan,zoomwheel,geocoder,share.html?access_token=pk.eyJ1IjoiZGFuaWVsamgiLCJhIjoiTnNYb25JSSJ9.vYOnsuu1zeKcGW2nj0uJZw'></iframe>


### Combining GeoJSON

Combining GeoJSON is also not a problem.
Municipality polygons and airports around large cities for example.

<iframe width='100%' height='500px' frameBorder='0' src='https://a.tiles.mapbox.com/v4/danieljh.l3jcdofm/attribution,zoompan,zoomwheel,geocoder,share.html?access_token=pk.eyJ1IjoiZGFuaWVsamgiLCJhIjoiTnNYb25JSSJ9.vYOnsuu1zeKcGW2nj0uJZw'></iframe>


### And That's It

This should give you an initial feel for what is possible with geospatial analysis.
