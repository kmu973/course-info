# Bike Share Station Density by Neighborhood

In the exercises this week, we're going to use PostGIS to surface some basic information that could be used when deciding which geographic areas to focus on for new bike share stations.

These exercises depend on:
* **Bikeshare station location data** (the _easiest_ way to get this is through the [station status GeoJSON download](http://www.rideindego.com/stations/json/), though I would normally advocate for using the GBFS resources)
* **Neighborhood polygon data** (there is no official source of neighborhood boundaries in Philadelphia, but Azavea, a local Philadelphia company that creates geospatial web-based applications and analyses, has created [the go-to source](https://github.com/azavea/geo-data/tree/master/Neighborhoods_Philadelphia))
* **Park polygon data** (I recommend using the [Philadelphia Parks and Recreation (PPR) Properties dataset](https://opendataphilly.org/dataset/ppr-properties))


```exercise
1. create new dataset in your postgres server 
2. create extension postgis in query;
3-1. create schema indego;
3-2. create schema phl;
3-3. create schema azavea;
4-1. 
ogr2ogr `
  -f "PostgreSQL" `
  -nln "indego.station_statuses" `
  -lco "OVERWRITE=yes" `
  -lco "GEOM_TYPE=geography" `
  -lco "GEOMETRY_NAME=the_geog" `
  PG:"host=localhost port=5432 dbname=week03 user=postgres password=K9737458k!" `
  "phl.json"

4-2. 
ogr2ogr `
  -f "PostgreSQL" `
  -nln "phl.parks" `
  -lco "OVERWRITE=yes" `
  -lco "GEOM_TYPE=geography" `
  -lco "GEOMETRY_NAME=the_geog" `
  PG:"host=localhost port=5432 dbname=week03 user=postgres password=K9737458k!" `
  "PPR_Properties.geojson"

4-3
ogr2ogr `
  -f "PostgreSQL" `
  -nln "azavea.neighborhoods" `
  -lco "OVERWRITE=yes" `
  -lco "GEOM_TYPE=geography" `
  -lco "GEOMETRY_NAME=the_geog" `
  PG:"host=localhost port=5432 dbname=week03 user=postgres password=K9737458k!" `
  "Neighborhoods_Philadelphia.geojson"

5.
select 
	nbd.name as name,
	nbd.the_geog as geog,
	count(*) as num_stations
from azavea.neighborhoods as nbd
join indego.station_statuses as stn
	on st_contains(nbd.the_geog::geometry, stn.the_geog::geometry)
group by nbd.name, nbd.the_geog
  
6.
select 
	nbd.name as name,
	nbd.the_geog as geog,
	count(stn.name) as num_stations
from azavea.neighborhoods as nbd
join indego.station_statuses as stn
	on st_contains(nbd.the_geog::geometry, stn.the_geog::geometry)
group by nbd.name, nbd.the_geog

7.
select 
	nbd.name as name,
	nbd.the_geog as geog,
	count(stn.name) / (st_area(nbd.the_geog) / 1000000) as density_sqkm
from azavea.neighborhoods as nbd
join indego.station_statuses as stn
	on st_contains(nbd.the_geog::geometry, stn.the_geog::geometry)
group by nbd.name, nbd.the_geog
order by density_sqkm desc

8.
with

neighborhood_densities as (
select 
	nbd.name as name,
	nbd.the_geog as geog,
	count(stn.name) / (st_area(nbd.the_geog) / 1000000) as density_sqkm
from azavea.neighborhoods as nbd
join indego.station_statuses as stn
	on st_contains(nbd.the_geog::geometry, stn.the_geog::geometry)
group by nbd.name, nbd.the_geog
order by density_sqkm desc 
),

avg_neighborhood_density as (
	select avg(density_sqkm) as avg_density_sqkm
	from neighborhood_densities
	where density_sqkm >0
	)
	
select
	nbd.name,
	nbd.geog,
	nbd.density_sqkm,
	case 
		when nbd.density_sqkm >= avgnbd.avg_density_sqkm
		then 'above'
		else 'below'
	end as rel_to_avg_density
from neighborhood_densities as nbd
cross join avg_neighborhood_density as avgnbd



```

Load those three datasets into a database.

1.  Write a query that lists which neighborhoods have the highest density of bikeshare stations. Let's say "density" means number of stations per square km.
    * Your query should return results containing:
      * The neighborhood name (in a column named `name`)
      * The neighborhood polygon as a `geography` (in a column named `geog`)
      * The number of bike share stations per square kilometer in the neighborhood (in a column named `density_sqkm`)
    * Your results should be ordered from most dense to least dense.
    * Be sure to include neighborhoods that have zero bike share stations in your results.
    * Note that the neighborhoods dataset has an area field; don't trust that field. Calculate the area using `ST_Area` yourself.

2.  Write a query to get the average bikeshare station density across _all the neighborhoods that have a non-zero bike share station density_.
    * The query should return a single record with a single field named `avg_density_sqkm`
    * Try using a common table expressions (CTE) to write this query.

3.  Write a query that lists which neighborhoods have a density above the average, and which have a density below average.
    * Your query should return results containing:
      * The neighborhood name (in a column named `name`)
      * The neighborhood polygon as a `geography` (in a column named `geog`)
      * The number of bike share stations per square kilometer in the neighborhood (in a column named `density_sqkm`)
      * The status relative to the average density (in a column named `rel_to_avg_density`). If the density is greater than or equal the average, the field should have a value of `'above'`. If the density is less than the average, the field should have a value of `'below'`.
