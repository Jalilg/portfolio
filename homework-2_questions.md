# Homework 2 -- Analytics with PostGIS

This homework will rely strongly on PostgreSQL and PostGIS documentation. Also refer to recommended readings, past lectures and labs.

**NOTE:** I take logic for solving problems into account when grading. When in doubt, write your thinking for solving the problem even if you aren't able to get a full response.

**NOTE:** If you run out of space in your carto account, you can delete tables you no longer use, or email me (andyepenn@design.upenn.edu) to get more storage space.

## Homework Problems

1. Which bus stops have the largest/smallest population within 800 meters? As a rough estimation, consider any block group that intersects the buffer as being part of the 800 meter buffer. Use 0.008 degrees as an approximation for 800 meters to avoid using geography which will slow the query and potentially cause DB timeouts. See problem 10 if you want to use `geography` without timeouts.

```SQL
SELECT s.the_geom, s.the_geom_webmercator, s.cartodb_id, coalesce(sum(cbgs.total_pop_2010), 0) as population
FROM gault34.septa_bus_stops as s
LEFT JOIN gault34.philadelphia_cbgs_w_population as cbgs
ON ST_DWithin(s.the_geom, cbgs.the_geom, 0.008)
GROUP BY 1, 2, 3
ORDER BY population DESC

--Follow-up Question: are you asking a list with the top ten largest and the top ten smallest? or just one list, sorted by with the smallest or largest? 

```

2. Using an 800m buffer around each bus stop, intersect with census block groups to find the ratio of the area of spatial intersection of the buffer with the original area of the block group polygon. Multiply this ratio by the total population of the census block groups. Finally, sum up the result to get an estimated population within the radius. _If you get query timeouts, choose about a dozen bus stops in and around Penn's campus. Once you're successful on the query that's limited, write the full query below._

Include the `stop_name`, estimated population, and point geometry. Order by estimated population (largest on top).

```SQL
-------------------------------------------------------------------- #this was my solution before I nested it
SELECT s.the_geom,
 s.the_geom_webmercator, 
 s.cartodb_id, 
 coalesce(sum(cbgs.total_pop_2010), 0) as cbgs_population, 
 coalesce(ST_Area(ST_Intersection(ST_Buffer(s.the_geom, 0.008), cbgs.the_geom))/ST_Area(cbgs.the_geom),0) as intersection_ratio,
 coalesce(coalesce(sum(cbgs.total_pop_2010), 0) * ST_Area(ST_Intersection(ST_Buffer(s.the_geom, 0.008), cbgs.the_geom))/ST_Area(cbgs.the_geom), 0) as assoc_population
FROM gault34.septa_bus_stops as s
LEFT JOIN gault34.philadelphia_cbgs_w_population as cbgs
ON ST_DWithin(s.the_geom, cbgs.the_geom, 0.008)
GROUP BY s.the_geom,
 s.the_geom_webmercator, 
 s.cartodb_id, cbgs.the_geom
ORDER BY cartodb_id DESC
---------------------------------------------------------------------- #after nesting, I am getting an error that I am issing a FROM caluse
SELECT s.the_geom,
 s.the_geom_webmercator, 
 s.cartodb_id, s.stop_id, 
 SUM(pop_by_stop.assoc_population) as total_pop
FROM (
SELECT s.the_geom,
 s.the_geom_webmercator, 
 s.cartodb_id, 
 coalesce(sum(cbgs.total_pop_2010), 0) as cbgs_population,  
 coalesce(ST_Area(ST_Intersection(ST_Buffer(s.the_geom, 0.008), cbgs.the_geom))/ST_Area(cbgs.the_geom),0) as intersection_ratio,
 coalesce(coalesce(sum(cbgs.total_pop_2010), 0) * ST_Area(ST_Intersection(ST_Buffer(s.the_geom, 0.008), cbgs.the_geom))/ST_Area(cbgs.the_geom), 0) as assoc_population
FROM gault34.septa_bus_stops as s
LEFT JOIN gault34.philadelphia_cbgs_w_population as cbgs
ON ST_DWithin(s.the_geom, cbgs.the_geom, 0.008)
GROUP BY s.the_geom,
 s.the_geom_webmercator, 
 s.cartodb_id, cbgs.the_geom
ORDER BY cartodb_id DESC
) as pop_by_stop
GROUP BY s.the_geom,
 s.the_geom_webmercator, 
 s.cartodb_id, 
 s.stop_id 
 ORDER BY s.the_geom,
 s.the_geom_webmercator, 
 cbgs.the_geom, s.cartodb_id, s.stop_id DESC
----------------------------------------------------------------------- #here's the same code, but in a help-friendly way
 SELECT s.the_geom,
 s.the_geom_webmercator, 
 s.cartodb_id, s.stop_id
 cartodb_id, SUM(pop_by_stop.assoc_population) as total_pop
FROM (
SELECT s.the_geom,
 s.the_geom_webmercator, 
 s.cartodb_id, 
 calculation1 as cbgs_population, 
 calculation2 as intersection_ratio,
 calculation3 as assoc_population
FROM gault34.septa_bus_stops as s
LEFT JOIN gault34.philadelphia_cbgs_w_population as cbgs
ON within_800m_function
GROUP BY s.the_geom,
 s.the_geom_webmercator, 
 s.cartodb_id, cbgs.the_geom
ORDER BY cartodb_id DESC
) as pop_by_stop
GROUP BY s.the_geom,
 s.the_geom_webmercator, 
 s.cartodb_id, s.stop_id
------------------------------------------------------------------------- #this is another function where I calculate the 
SELECT s.the_geom,
 s.the_geom_webmercator,
 s.cartodb_id, s.stop_id,
 sum(cbgs.total_pop_2010) * ST_Area(ST_Intersection(ST_Buffer(s.the_geom, 0.008), cbgs.the_geom))/ST_Area(cbgs.the_geom) as total_assoc_population
FROM gault34.septa_bus_stops as s
LEFT JOIN gault34.philadelphia_cbgs_w_population as cbgs
ON ST_DWithin(s.the_geom, cbgs.the_geom, 0.008)
GROUP BY s.the_geom,
 s.the_geom_webmercator, 
 cbgs.the_geom,
 s.cartodb_id,
 s.stop_id
ORDER BY s.the_geom,
 s.the_geom_webmercator, 
 cbgs.the_geom, s.cartodb_id, s.stop_id DESC
-------------------------------------------------------------------------

SELECT s.the_geom,
 s.the_geom_webmercator,
 s.cartodb_id, s.stop_id,
 sum(cbgs.total_pop_2010) * calculation as total_assoc_population
FROM gault34.septa_bus_stops as s
LEFT JOIN gault34.philadelphia_cbgs_w_population as cbgs
ON within_800m_function
GROUP BY s.the_geom,
 s.the_geom_webmercator, 
 cbgs.the_geom,
 s.cartodb_id,
 s.stop_id
ORDER BY s.the_geom,
 s.the_geom_webmercator, 
 cbgs.the_geom, s.cartodb_id, s.stop_id DESC

------------------------------------------------------------------------- Why does the cobe below run endlesly? Also on Piazza
SELECT pop_by_stop.the_geom,
 pop_by_stop.stop_id, 
 SUM(pop_by_stop.assoc_population) as total_pop
FROM (
SELECT s.the_geom,
 s.the_geom_webmercator, 
 s.cartodb_id,
 s.stop_id,
 coalesce(sum(cbgs.total_pop_2010), 0) as cbgs_population,
 coalesce(ST_Area(ST_Intersection(ST_Buffer(s.the_geom, 0.008), cbgs.the_geom))/ST_Area(cbgs.the_geom),0) as intersection_ratio,
 coalesce(coalesce(sum(cbgs.total_pop_2010), 0) * ST_Area(ST_Intersection(ST_Buffer(s.the_geom, 0.008), cbgs.the_geom))/ST_Area(cbgs.the_geom), 0) as assoc_population
FROM gault34.septa_bus_stops as s
LEFT JOIN gault34.philadelphia_cbgs_w_population as cbgs
ON ST_DWithin(s.the_geom, cbgs.the_geom, 0.008)
GROUP BY s.the_geom,
 s.the_geom_webmercator, 
 s.cartodb_id, cbgs.the_geom
ORDER BY cartodb_id DESC
) as pop_by_stop
GROUP BY pop_by_stop.the_geom,
 pop_by_stop.the_geom_webmercator, 
 pop_by_stop.cartodb_id, 
 pop_by_stop.stop_id 
 ORDER BY pop_by_stop.the_geom,
 pop_by_stop.the_geom_webmercator, 
 pop_by_stop.cartodb_id, 
 pop_by_stop.stop_id DESC

--------------------------------------------------------------------------  I can't get the geom to show in the right place

SELECT sum(interpolated_pop) as pop, stop_name
FROM (
	SELECT
    blocks.total_pop_2010 * ST_Area(ST_Intersection(blocks.the_geom, stops.the_geom)::geography) / ST_Area(blocks.the_geom::geography) AS interpolated_pop,
    stop_name
  FROM gault34.philadelphia_cbgs_w_population AS blocks
  JOIN  (
	SELECT stop_name, ST_Buffer(the_geom,0.008) as the_geom 
	FROM gault34.septa_bus_stops
    LIMIT 12 --what does this "limit" mean? How do I know which 12 buffers are being created?
   ) as stops
  ON ST_Intersects(blocks.the_geom, stops.the_geom)
ORDER BY 2 desc
) as solution
GROUP BY stop_name
LIMIT 5


```

**Answer:** Create in the table below with the first five rows of the results, even if just a sample of bus stops.

| stop_name | estimated_pop_800m | geom |
|-----------|--------------------|------|
| a | 10 | Point(...) |


3. Using the OSM Buildings dataset, pair each building with its closest bus stop. The final result should give the building name, bus stop name, and distance apart in meters. Order by distance (largest on top).

```SQL
SELECT stops.stop_name as stop_name,
 buildings.osm_id,
ST_Distance(buildings.the_geom, stops.the_geom) as distance_apart
FROM gault34.philadelphia_osm_buildings as buildings
CROSS JOIN LATERAL
	(SELECT stop_name,
		the_geom
		FROM gault34.septa_bus_stops
		ORDER BY buildings.the_geom <-> the_geom
		LIMIT 1)
	AS stops
    ORDER BY distance_apart DESC
    LIMIT 5

------------- Why didnt this work? I just recieved an error that column buildings.the_geom_webmarcator does not exist, and I'm trying to project to the correct units. In my answer, I just ended up multiplying by 10,000

SELECT stops.stop_name as stop_name,
 buildings.osm_id,
 ST_Transform(the_geom, 3857) as the_geom_webmarcator,
ST_Distance(buildings.the_geom_webmarcator, stops.the_geom_webmarcator) as distance_apart
FROM gault34.philadelphia_osm_buildings as buildings
CROSS JOIN LATERAL
	(SELECT stop_name,
		ST_Transform(the_geom, 3857) as the_geom_webmarcator
		FROM gault34.septa_bus_stops
		ORDER BY buildings.the_geom_webmarcator <-> the_geom_webmercator
		LIMIT 1)
	AS stops
    ORDER BY distance_apart DESC
    LIMIT 5
```

**Answer:** Fill in the table below for the five buildings furthest from a bus stop.

| stop_name | osm_id | distance_apart |
|-----------|--------|----------------|
| Oak Hill Apartments Loop | 767734667 | 0.01679977936581504 |
| Oak Hill Apartments Loop | 767734665 | 0.016140143234183203 |
| Enterprise Av & Fort Mifflin Rd | 690041863 | 0.014722507496355001 |
| Enterprise Av & Fort Mifflin Rd | 489426102 | 0.014525954999586421 |
| Enterprise Av & Fort Mifflin Rd | 690041864 | 0.014514947559328646 |

4. Using the `shapes.txt` file from GTFS bus feed, find the two routes with the longest trips (in meters). In the final query, give the `trip_headsign` that corresponds to the `shape_id` of this route and the length of the trip. You can use the [Carto batch SQL API tool](https://cartodb.github.io/customer_success/batch/) to do the geometry updates (`UPDATE table SET the_geom = ...`) to avoid DB timeouts.

```SQL
SELECT *, neighborhoods.listname
FROM gault34.neighborhoods_philadelphia as neighborhoods
LEFT JOIN gault34.septa_bus_stops 
ON ST_Contains(stops.the_geom, neighborhoods.the_geom) as contains_

I'm trying to create a table that shows true/false values for which stops are inside of a neighborhood
```

4. Rate neighborhoods by their bus stop accessibility for wheelchairs. Use the [neighborhood dataset from the open data portal](https://www.opendataphilly.org/dataset/philadelphia-neighborhoods) along with an appropriate dataset from the Septa GTFS bus feed. Use the [GTFS documentation](https://gtfs.org/reference/static/) for help. Use some creativity in the metric you devise in rating neighborhoods.

```SQL
-- write your query here
```

**Answer:** Fill in the table below for the **top** two neighborhoods.

| neighborhood_name | accessibility_metric | num_bus_stops_accessible | num_bus_stops_inaccessible |
|-------------------|----------------------|--------------------------|----------------------------|
| a | 87 | 37 | 4 |

Fill in the table below for the **bottom** two neighborhoods.

| neighborhood_name | accessibility_metric | num_bus_stops_accessible | num_bus_stops_inaccessible |
|-------------------|----------------------|--------------------------|----------------------------|
| a | 87 | 37 | 4 |


5. With a query, find out how many census block groups Penn's main campus fully contains. Discuss which dataset you chose for defining Penn's campus.

```SQL
-- write your query here
```

**Answer:**

6. With a query involving OSM buildings and census block groups, find the geoid of Meyerson Hall. ST_MakePoint() and functions like that are not allowed.

```SQL
-- write your query here
```

**Answer:**

7. Expand the `device_home_areas` stringified JSON entry of the Safegraph Neighborhood Patterns row corresponding to Meyerson Hall's geoid into a table using an appropriate PostgreSQL JSON function. Table aliases can alias column names (e.g., "SELECT * FROM (SELECT 1 as a, 2 as b) as t(c, d)" renames the columns a and b to c and d.) Order by origin geoid and the geoid of the home devices.

Hint: You can cast a well-formed json string into a json object using `::json` type casting.

```SQL
-- write your query here
```
Create a markdown table to store the first five entries of the query result.

**Answer:**

8. Using the result from \#7, write a query that gives the distance (in meters) from Meyerson Hall's block group to the census block groups of all the `device_home_areas`, the number of devices, and a linestring geometry connecting the centroids of each device home to Meyerson Hall's block group.

```sql
-- write your query here
```


9. You're tasked with giving more contextual information to rail stops to fill the `stop_desc` field in a GTFS feed. Using OSM building data, PostGIS functions (e.g., `ST_Distance`, `ST_Azimuth`), and postgres string functions, build a description (alias as `stop_desc`) paired with `stop_id`, `stop_name`, `stop_lon`, and `stop_lat`. Feel free to supplement with other datasets (must provide link to data used so it's reproducible), and other methods of describing the relationships. Postgres' `CASE` statements may be helpful for some operations.

**Tip when experimenting:** Use subqueries to limit your query to just a few rows to keep query times faster. Once your query is giving you answers you want, scale it up. E.g., instead of `FROM tablename`, use `FROM (SELECT * FROM tablename limit 10) as t`.

Final query results should be in this form (`stop_desc` is only an example, feel free to be creative, silly, descriptive, etc.):

```
stop_id | stop_name |           stop_desc          | stop_lon | stop_lat
1       | SQL Road  | 37 meters NE of ABC Building | 0        | 0
```

## Bonus Question

10. Create an index on the geography for the queries involved in questions 1 and 2. Time the queries before and after having the geography index using the tool we used in class (<https://cartodb.github.io/customer_success/batch/>). Write the expression you used to create the geography index, and provide screenshots highlighting the performance of the queries.

