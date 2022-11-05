# SQL Language

### Make Tables

Create a table
```sql
CREATE TABLE weather (
	city varchar(80),
	temp_lo int,   -- low temperature
	temp_hi int,   -- high temperature
	prcp real,     -- precipitation 
	date date
);

CREATE TABLE cities (
	name  varchar(80),
	location  point
);
```

### Check Schema
Check all tables and detail of any specific tables as
```sql
orm=# \d
          List of relations
 Schema |  Name   | Type  |  Owner   
--------+---------+-------+----------
 public | weather | table | postgres
(1 row)

gorm=# 

gorm=# \d weather
                      Table "public.weather"
 Column  |         Type          | Collation | Nullable | Default 
---------+-----------------------+-----------+----------+---------
 city    | character varying(80) |           |          | 
 temp_lo | integer               |           |          | 
 temp_hi | integer               |           |          | 
 prcp    | real                  |           |          | 
 date    | date                  |           |          | 

```

### Populate Tables

You can list the columns in a different order if you wish or even omit some columns. Many developers consider explicitly listing the columns better style than relying on the order implicitly.

```sql
INSERT INTO weather VALUES ('India', 30, 40, 1.2, '2022-11-04'); -- column names in same order as in tables description
INSERT INTO weather (prcp, city, temp_lo, temp_hi, date) VALUES (1.4, 'Hyderabad', 25, 30, '2022-05-01');
INSERT INTO weather (city, temp_lo, temp_hi, date) VALUES ('Hyderabad', 25, 30, '2022-05-01');
SELECT * FROM weather;
SELECT * FROM weather ORDER BY city, temp_lo;
SELECT DISTINCT city FROM weather;

INSERT INTO cities VALUES ('India', '(-194.0, 53.0)');
```

### Joins

Join Queries

```sql
SELECT *
    FROM weather, cities
    WHERE city = name;
```
Select column names
```sql
SELECT weather.city, weather.temp_lo, weather.temp_hi,
       weather.prcp, weather.date, cities.location
    FROM weather JOIN cities ON weather.city = cities.name;

OR

SELECT *
    FROM weather, cities
    WHERE city = name;
```

Left Joins
```sql
SELECT *
    FROM weather LEFT OUTER JOIN cities ON weather.city = cities.name;
```
Self Join
- we wish to find all the weather records that are in the temperature range of other weather records
```sql
SELECT w1.city, w1.temp_lo AS low, w1.temp_hi AS high,
       w2.city, w2.temp_lo AS low, w2.temp_hi AS high
    FROM weather w1 JOIN weather w2
        ON w1.temp_lo < w2.temp_lo AND w1.temp_hi > w2.temp_hi;
```

The join is usually performed in a more efficient manner than actually comparing each possible pair of rows, but this is invisible to the user.

### Aggregate Functions - Where and Having

```sql
SELECT max(temp_lo) FROM weather;
gorm=# SELECT max(temp_lo) FROM weather;
 max 
-----
  30
(1 row)
```
we wanted to know what city (or cities) that reading occurred in
```sql
SELECT city FROM weather WHERE temp_lo = max(temp_lo);     WRONG
```
but this will not work since the aggregate max cannot be used in the `WHERE` clause. (This restriction exists because the `WHERE` clause determines which rows will be included in the aggregate calculation; so obviously it has to be evaluated before aggregate functions are computed.) 

Use a sub query instead
```sql
SELECT city FROM weather
    WHERE temp_lo = (SELECT max(temp_lo) FROM weather);
```
```sql
gorm=# SELECT city, max(temp_lo)
gorm-#     FROM weather
gorm-#     GROUP BY city;
   city    | max 
-----------+-----
 India     |  30
 Hyderabad |  25
(2 rows)
```

Select the cities where the max of temp_lo is less than 30 and count the number of same.
```sql
SELECT city, MAX(temp_lo), COUNT(*) FILTER (WHERE temp_lo < 30) FROM weather GROUP BY city HAVING MAX(temp_lo) < 30;
   city    | max | count 
-----------+-----+-------
 Hyderabad |  25 |     2
(1 row)
```

The fundamental difference between `WHERE` and `HAVING` is this: `WHERE` selects input rows before groups and aggregates are computed (thus, it controls which rows go into the aggregate computation), whereas HAVING selects group rows after groups and aggregates are computed. Thus, the `WHERE` clause must not contain aggregate functions; it makes no sense to try to use an aggregate to determine which rows will be inputs to the aggregates. On the other hand, the HAVING clause always contains aggregate functions. 

### Updates
```sql
UPDATE weather
    SET temp_hi = temp_hi - 2,  temp_lo = temp_lo - 2
    WHERE date > '1994-11-28';
UPDATE 4
```

### Deletions 
```sql
DELETE FROM weather WHERE city = 'Hyderabad';
DELETE 2
```

To delete all records
```sql
DELETE FROM tablename;
```
