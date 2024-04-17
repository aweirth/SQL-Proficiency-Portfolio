# POST GIS - Spatial Analysis

## Data Dictionary

### cities1000
The `cities1000.txt` file is tab delimited and has the following columns:

| Column Name       | Description                                                                                                                                                             |
|-------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| geonameid         | integer id of record in geonames database                                                                                                                               |
| name              | name of geographical point (utf8) varchar(200)                                                                                                                          |
| asciiname         | name of geographical point in plain ascii characters, varchar(200)                                                                                                      |
| alternatenames    | alternatenames, comma separated, ascii names automatically transliterated, convenience attribute from alternatename table, varchar(10000)                               |
| latitude          | latitude in decimal degrees (wgs84)                                                                                                                                     |
| longitude         | longitude in decimal degrees (wgs84)                                                                                                                                    |
| feature class     | see http://www.geonames.org/export/codes.html, char(1)                                                                                                                  |
| feature code      | see http://www.geonames.org/export/codes.html, varchar(10)                                                                                                              |
| country code      | ISO-3166 2-letter country code, 2 characters                                                                                                                            |
| cc2               | alternate country codes, comma separated, ISO-3166 2-letter country code, 200 characters                                                                                |
| admin1 code       | fipscode (subject to change to iso code), see exceptions below, see file admin1Codes.txt for display names of this code; varchar(20)                                    |
| admin2 code       | code for the second administrative division, a county in the US, see file admin2Codes.txt; varchar(80)                                                                  |
| admin3 code       | code for third level administrative division, varchar(20)                                                                                                               |
| admin4 code       | code for fourth level administrative division, varchar(20)                                                                                                              |
| population        | bigint (8 byte int)                                                                                                                                                     |
| elevation         | in meters, integer                                                                                                                                                      |
| dem               | digital elevation model, srtm3 or gtopo30, average elevation of 3''x3'' (ca 90mx90m) or 30''x30'' (ca 900mx900m) area in meters, integer. srtm processed by cgiar/ciat. |
| timezone          | the iana timezone id (see file timeZone.txt) varchar(40)                                                                                                                |
| modification date | date of last modification in yyyy-MM-dd format                                                                                                                          |


### country_info
The `country_info.txt` file is also tab delimited and has more detailed information about each country
| Column Name      | Description                                                       |
|------------------|-------------------------------------------------------------------|
| iso              | A country's two letter ISO-3166 code                              |
| iso3             | A country's three letter iso code                                 |
| iso_num          | A country's numeric iso code                                      |
| fips             | An alternative two letter identifying code                        |
| name             | The name of the country                                           |
| capital          | The capital of teh country                                        |
| area_sq_km       | The area of the country in square km                              |
| population       | The total population of the country                               |
| continent        | Two letter code indicating continent                              |
| tld              | Country domain name suffix                                        |
| curr_code        | Currency code of country                                          |
| curr_name        | Name of country currency                                          |
| phone            | Country phone prefix                                              |
| post_code_format | General format of postal codes in country                         |
| post_code_regex  | A regular expression showing the pattern of possible postal codes |
| languages        | ISO-639 language codes for spoken languages in the country        |
| neighbors        | Neighboring country iso codes                                     |
| eq_fips          | Equivalent fips code                                              |


# Analysis

Need city location in GIS format - adding geography POINT object in SRID 4326 using existing lat and long columns and GiST index on it.

```sql
-- SQL to create and populate new column
ALTER TABLE cities ADD COLUMN geog_loc geography(POINT, 4326);
UPDATE cities SET geog_loc = ST_SetSRID(ST_MakePoint(longitude,latitude)::geography,4326);

-- SQL to add index
CREATE INDEX geog_loc_index ON cities USING GIST (geog_loc);
```
## Assuming you can see 150km from the airplane, what 5 most populous cities will passengers be able to see from an airplane flying from Portland, OR to Paris?

### Query:
```sql
SELECT *
FROM cities
WHERE ST_DWithin(ST_SetSRID(ST_MakeLine(ST_MakePoint(-122.67621,45.52345), ST_MakePoint(2.3488, 48.85341)), 4326), geog_loc, 150000)
ORDER BY population DESC LIMIT 5
;
```

## Querying how many cities will you see from each country (country name and its city count, in descending order):

```sql
COPY(
  SELECT co.name AS country, COUNT(*) AS city_count
  FROM cities AS ci
  JOIN country_info AS co 
      ON co.iso = ci.country_code
  WHERE ST_DWithin(ST_SetSRID(ST_MakeLine(ST_MakePoint(-122.67621,45.52345), ST_MakePoint(2.3488, 48.85341)), 4326), geog_loc, 150000)
  GROUP BY country
  ORDER BY city_count DESC
)
TO '/Users/alexweirth/Documents/data_351/data/cities_per_country.csv'
WITH(FORMAT CSV, HEADER)
;
```

# Working with a Shapefile:

## After importing a shapefile for all U.S. county polygons, we wanted to find population density (people per square kilometer) of all Oregon Counties:

### Query:
```sql
-- modifying shapefile so it is only oregon counties
CREATE TABLE oregon_counties AS(
  SELECT *
  FROM counties
  WHERE statefp10 = '41'
);

-- query for populations by county and square km for each county into a temp table
CREATE TEMP TABLE county_pop_km2 AS(
  SELECT oregon_counties.namelsad10 AS county, 
      SUM(cities.population) AS population, 
      SUM(ST_Area(oregon_counties.geom::geography))/1000000 AS km2
  FROM oregon_counties
  JOIN cities ON ST_Intersects(cities.geog_loc, oregon_counties.geom)
  GROUP BY county
  ORDER BY population DESC
);

-- Calculate people/km2 and export the CSV
COPY(
  SELECT county, ROUND(population/km2::NUMERIC, 2) AS people_per_km2
  FROM county_pop_km2
  ORDER BY people_per_km2 DESC
)
TO '/Users/alexweirth/Documents/data_351/data/or_county_densities.csv'
WITH(FORMAT CSV, HEADER)
;
```
