# Working with Bureau of Transportation Statistics Dataset of Flights

### Structure pictured below:

![Flight Dataset DrawSQL](./images/flight_drawsql.png)

#### Querying the airport with the greatest average delay in arriving flights (minimum 100 ariving flights)

```sql
SELECT apts.description, AVG(flts.arrive_delay) AS avg_delay
FROM hw6.flights AS flts
LEFT JOIN hw6.airports AS apts 
  ON flts.origin_airport_id = apts.airport_id
GROUP BY apts.description
HAVING COUNT(*) > 100
ORDER BY avg_delay DESC LIMIT 1
;
```

#### Querying the 3 U.S. cities in descending order with the greatest number of inbound flights across all airports serving that city

```sql
SELECT cm.description AS airport, COUNT(cm.description) AS flight_count
FROM hw6.flights AS flts
LEFT JOIN hw6.city_markets as cm 
  ON cm.city_market_id = flts.dest_city_market_id
GROUP BY cm.description
ORDER BY COUNT(cm.description) DESC LIMIT 3
;
```

#### Querying the 5 airports that have the greatest number of outgoing flights per day on average (along with those averages)

```sql
-- This temp table is airports grouped by name, and then date meaning each entry in the table is groups of distinct aiport/date pairs and
-- a column that sums all the flights on that day for that distinct pair.
CREATE TEMP TABLE apfd AS (
  SELECT origin_airport_id, flight_date, COUNT(*) AS flights_perday
  FROM hw6.flights
  GROUP BY origin_airport_id, flight_date
  );

-- This query then joins the airport description (name) to the temp table apfd so we can group by airport (to which a bunch of 'flights
-- per day' values are related to, so we can take an average of those values which are grouped by airport to give us avg flights per
-- day of each airport.
SELECT apts.description, ROUND(AVG(apfd.flights_perday), 0) FROM apfd
LEFT JOIN hw6.airports as apts ON apts.airport_id = apfd.origin_airport_id
GROUP BY apts.description
ORDER BY ROUND(AVG(apfd.flights_perday), 0) DESC LIMIT 5
;
```

#### Querying the airports that represent the longest single flight distance for each airline (output is name of carrier, DEP and ARR airports flitering out duplicate round trips.)

```sql
-- This temp table is each carrier id paired with its longest flights distance
CREATE TEMP TABLE maxdist_flight AS(
  SELECT flts.carrier_id AS cid, MAX(flts.flight_distance) AS max_distance
  FROM hw6.flights as flts
  GROUP BY flts.carrier_id
  );

-- The DISTINCT ON query makes sure we only grab one observation from each grouping so we cannot have duplicates
SELECT DISTINCT ON(crs.name)crs.name AS carrier, mxd.max_distance, apts1.description AS origin_airport, apts2.description AS destination_airport
FROM maxdist_flight AS mxd
-- joining 'carriers' table to the mxd table so we are able to query the name of the carriers
LEFT JOIN hw6.carriers as crs 
  ON crs.carrier_id = mxd.cid
-- joining flights table back to temp table again so we can obtain origin & dest aiport ids which in a later join we can gdt the names for
-- these airports.
LEFT JOIN hw6.flights as flts 
  ON flts.carrier_id = mxd.cid AND flts.flight_distance = mxd.max_distance
-- joining airport name to mxd flight table so we can see airport name of the origin location
LEFT JOIN hw6.airports as apts1 
  ON apts1.airport_id = flts.origin_airport_id
-- joining airport name to mxd flight table so we can see airport name of the destination location
LEFT JOIN hw6.airports as apts2 
  ON apts2.airport_id = flts.dest_airport_id
ORDER BY crs.name
;
```

# Data Visualization with the flights dataset in Excel

### Simple bar chart showing the total number of flights departing each day of the week

#### Query to give me a CSV of counts of flights across each weekday:

```sql
COPY(
SELECT wd.day_name, COUNT(*) AS flight_count
FROM hw6.flights AS flts
LEFT JOIN hw6.weekdays AS wd 
  ON wd.code = flts.day_of_week
GROUP BY wd.code
)
TO '/Users/alexweirth/Documents/data_351/data/weekday_flights.csv'
WITH(FORMAT CSV, HEADER);
```

#### Visualization:
![weekday flight dist](./images/flight_weekdist.png)

### Further Visualization: how do canceled flights vary among airlines? Specifically, are there observations we could make that show certain airlines having differing common cancellation types?

### Goal: create a bar chart for airline cancelations colored by cancellation type

#### Query:
```sql
-- Creating table with counts of weather cancellations grouped by airline
CREATE TABLE weather_canc AS(
  SELECT crs.name AS carrier, COUNT(*) AS weather_canc
  FROM hw6.flights AS flts
  JOIN hw6.carriers AS crs 
    ON crs.carrier_id = flts.carrier_id
  JOIN hw6.cancellation_codes as cc 
    ON cc.code = flts.cancel_code
  WHERE cc.description = 'Weather'
  GROUP BY carrier
);

-- Creating table with counts of Carrier cancellations grouped by airline
CREATE TABLE carrier_canc AS(
  SELECT crs.name AS carrier, COUNT(*) AS carrier_canc
  FROM hw6.flights AS flts
  JOIN hw6.carriers AS crs 
    ON crs.carrier_id = flts.carrier_id
  JOIN hw6.cancellation_codes as cc 
    ON cc.code = flts.cancel_code
  WHERE cc.description = 'Carrier'
  GROUP BY carrier
);

-- Creating table with counts of National Air System cancellations grouped by airline
CREATE TABLE nas_canc AS(
  SELECT crs.name AS carrier, COUNT(*) AS nas_canc
  FROM hw6.flights AS flts
  JOIN hw6.carriers AS crs 
    ON crs.carrier_id = flts.carrier_id
  JOIN hw6.cancellation_codes as cc 
    ON cc.code = flts.cancel_code
  WHERE cc.description = 'National Air System'
  GROUP BY carrier
);

-- Creating joined table of all the counts together
CREATE TABLE canc_dist AS (
  SELECT wea.carrier, wea.weather_canc, car.carrier_canc, nas.nas_canc
  FROM weather_canc AS wea
  JOIN carrier_canc AS car 
    ON car.carrier = wea.carrier
  JOIN nas_canc AS nas 
    ON nas.carrier = wea.carrier
);

-- Altering joined table so I can also have the calculated percentages of each type of cancellation grouped by airline.
ALTER TABLE canc_dist ADD COLUMN total_canc INT;
UPDATE canc_dist SET total_canc = weather_canc + carrier_canc + nas_canc;

ALTER TABLE canc_dist ADD COLUMN perc_weather NUMERIC;
UPDATE canc_dist SET perc_weather = ROUND(weather_canc::NUMERIC / total_canc::NUMERIC, 2);

ALTER TABLE canc_dist ADD COLUMN perc_carrier NUMERIC;
UPDATE canc_dist SET perc_carrier = ROUND(carrier_canc::NUMERIC / total_canc::NUMERIC, 2);

ALTER TABLE canc_dist ADD COLUMN perc_nas NUMERIC;
UPDATE canc_dist SET perc_nas = ROUND(nas_canc::NUMERIC / total_canc::NUMERIC, 2);
```

#### Visulization 1: 

![cancelation vis 1](./images/2b_vis1.png)

This wasnt very helpful because it was hard to compare since some airlines in the dataset are much larger companies than the others and have way more flights so what I did was I used the percentage columns to create a 100% stacked barchart which allowed me to compare different airlines cancellation PERCENTAGES.

#### Visulization 2: 

![cancelation vis 2](./images/2b_vis2.png)

This was very helpful and what I could tell from this graph were a couple things:

1. Alaska airlines tends to have very few weather cancellations, most likely because they are based out of seattle where the weather can be rainy,
but for the most part not severe to the point that flights are canceled.

2. Jet Blue also had VERY few weather cancellations which was very weird since the dataset is during the month of january and Jet Blue is based in 
NY where the weather is usually pretty bad during that time of the year. So if you are from NY and are flying in Jan, maybe Jet Blue is least likely
to get canceled! Although some further research would be needed.

3. Some larger airlines have differing amounts of National Air System cancellations, Alaska, Southwest, Spirit, United all have NAS cancellations of around 1-3% while Delta, American, Jetblue all have much higher NAS cancelations anywhere from 15-44%. Not sure why this is but also. is another interesting question.

4. Alaska airlines had the second largest number of canceled flights in the data set, but it ranked FIRST in carrier cancellations by 30%! Not sure why this is but again is another avenue for further research.


