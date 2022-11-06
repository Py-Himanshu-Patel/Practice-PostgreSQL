# Advance Features

### Views
```sql
CREATE VIEW myview AS
    SELECT name, temp_lo, temp_hi, prcp, date, location
        FROM weather, cities
        WHERE city = name;

SELECT * FROM myview;
```
```sql
gorm=# CREATE VIEW myview AS SELECT name, temp_lo, temp_hi, 
prcp, location from weather, cities WHERE city=name;
CREATE VIEW
gorm=# 
gorm=# SELECT * FROM myview;
 name  | temp_lo | temp_hi | prcp | location  
-------+---------+---------+------+-----------
 India |      28 |      38 |  1.2 | (-194,53)
 India |      28 |      38 |  1.2 | (-194,53)
(2 rows)
gorm=# 
```

### Foreign Keys

Before proceeding let's delete both the pre exisitng tables.
```sql
gorm=# DROP TABLE cities CASCADE;
NOTICE:  drop cascades to view myview
DROP TABLE
gorm=# drop table weather;
DROP TABLE
```

The new declaration of the tables would look like this:
```sql
CREATE TABLE cities (
        name     varchar(80) primary key,
        location point
);

CREATE TABLE weather (
        city      varchar(80) references cities(name),
        temp_lo   int,
        temp_hi   int,
        prcp      real,
        date      date
);
```

Now try inserting an invalid record:
```sql
INSERT INTO weather VALUES ('Berkeley', 45, 53, 0.0, '1994-11-28');
ERROR:  insert or update on table "weather" violates foreign key constraint "weather_city_fkey"
DETAIL:  Key (city)=(Berkeley) is not present in table "cities".
```

### Transactions - COMMIT, ROLLBACK

Transactions are a fundamental concept of all database systems. The essential point of a transaction is that it bundles multiple steps into a single, all-or-nothing operation. The intermediate states between the steps are not visible to other concurrent transactions, and if some failure occurs that prevents the transaction from completing, then none of the steps affect the database at all.

In PostgreSQL, a transaction is set up by surrounding the SQL commands of the transaction with `BEGIN` and `COMMIT` commands.

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100.00
    WHERE name = 'Alice';
-- etc etc
COMMIT;
```

```sql
gorm=# begin;
BEGIN
gorm=*# INSERT INTO cities VALUES ('Jaipur', '(1,1)');
INSERT 0 1
gorm=*# commit;
COMMIT
gorm=# SELECT * FROM cities;
   name    | location 
-----------+----------
 Hyderabad | (1,2)
 Jaipur    | (1,1)
(2 rows)
```
Instead of `COMMIT` we can `ABORT` and the transaction will be rolledback.

### Window Functions

Create a table with data inorder to proceed.
```sql
CREATE TABLE empsalary (
	depname varchar(50),
	empno int,
	salary int
);

INSERT INTO empsalary VALUES 
('develop', 11, 5200),
('develop', 7, 4200),
('develop', 9, 4500),
('develop', 8, 6000),
('develop', 10, 5200),
('personnel', 5, 3500),
('personnel', 2, 3900),
('sales', 3, 4800),
('sales', 1, 5000),
('sales', 4, 4800);
```

A window function performs a calculation across a set of table rows that are somehow related to the current row. This is comparable to the type of calculation that can be done with an aggregate function. However, window functions do not cause rows to become grouped into a single output row like non-window aggregate calls would.

Here is an example that shows how to compare each employee's salary with the average salary in his or her department:
```sql
SELECT depname, empno, salary, avg(salary) OVER (PARTITION BY depname) FROM empsalary;
  depname  | empno | salary |          avg
-----------+-------+--------+-----------------------
 develop   |    11 |   5200 | 5020.0000000000000000
 develop   |     7 |   4200 | 5020.0000000000000000
 develop   |     9 |   4500 | 5020.0000000000000000
 develop   |     8 |   6000 | 5020.0000000000000000
 develop   |    10 |   5200 | 5020.0000000000000000
 personnel |     5 |   3500 | 3700.0000000000000000
 personnel |     2 |   3900 | 3700.0000000000000000
 sales     |     3 |   4800 | 4866.6666666666666667
 sales     |     1 |   5000 | 4866.6666666666666667
 sales     |     4 |   4800 | 4866.6666666666666667
(10 rows)
```

```sql
SELECT depname, empno, salary,
       rank() OVER (PARTITION BY depname ORDER BY salary DESC)
FROM empsalary;
  depname  | empno | salary | rank
-----------+-------+--------+------
 develop   |     8 |   6000 |    1
 develop   |    10 |   5200 |    2
 develop   |    11 |   5200 |    2
 develop   |     9 |   4500 |    4
 develop   |     7 |   4200 |    5
 personnel |     2 |   3900 |    1
 personnel |     5 |   3500 |    2
 sales     |     1 |   5000 |    1
 sales     |     4 |   4800 |    2
 sales     |     3 |   4800 |    2
(10 rows)
```
As shown here, the rank function produces a numerical rank for each distinct `ORDER BY` value in the current row's partition, using the order defined by the `ORDER BY` clause. rank needs no explicit parameter, because its behavior is entirely determined by the `OVER` clause.

```sql
SELECT salary, sum(salary) OVER () FROM empsalary;
 salary |  sum
--------+-------
   5200 | 47100
   5000 | 47100
   3500 | 47100
   4800 | 47100
   3900 | 47100
   4200 | 47100
   4500 | 47100
   4800 | 47100
   6000 | 47100
   5200 | 47100
(10 rows)
```
Since there is no `ORDER BY` in the OVER clause, the window frame is the same as the partition, which for lack of `PARTITION BY` is the whole table; in other words each sum is taken over the whole table and so we get the same result for each output row. But if we add an `ORDER BY` clause, we get very different results.

```sql
SELECT salary, sum(salary) OVER (ORDER BY salary) FROM empsalary;
 salary |  sum
--------+-------
   3500 |  3500
   3900 |  7400
   4200 | 11600
   4500 | 16100
   4800 | 25700
   4800 | 25700
   5000 | 30700
   5200 | 41100
   5200 | 41100
   6000 | 47100
(10 rows)
```

### Inheritance

Let's create two tables: A table cities and a table capitals. Naturally, capitals are also cities, so you want some way to show the capitals implicitly when you list all cities. If you're really clever you might invent some scheme like this:
```sql
CREATE TABLE capitals (
  name       text,
  population real,
  elevation  int,    -- (in ft)
  state      char(2)
);

CREATE TABLE non_capitals (
  name       text,
  population real,
  elevation  int     -- (in ft)
);

CREATE VIEW cities AS
  SELECT name, population, elevation FROM capitals
    UNION
  SELECT name, population, elevation FROM non_capitals;
```

A better solution is this:
```sql
CREATE TABLE cities (
  name       text,
  population real,
  elevation  int     -- (in ft)
);

CREATE TABLE capitals (
  state      char(2) UNIQUE NOT NULL
) INHERITS (cities);
```
In this case, a row of capitals inherits all columns (name, population, and elevation) from its parent, cities. The type of the column name is text, a native PostgreSQL type for variable length character strings. The capitals table has an additional column, state, which shows its state abbreviation. In PostgreSQL, a table can inherit from zero or more other tables.

The following query finds the names of all cities, including state capitals, that are located at an elevation over 500 feet:
```sql
SELECT name, elevation
  FROM cities
  WHERE elevation > 500;
```
which returns:
```sql
   name    | elevation
-----------+-----------
 Las Vegas |      2174
 Mariposa  |      1953
 Madison   |       845
(3 rows)
```
The following query finds all the cities that are not state capitals and are situated at an elevation over 500 feet:
```sql
SELECT name, elevation
    FROM ONLY cities
    WHERE elevation > 500;
   name    | elevation
-----------+-----------
 Las Vegas |      2174
 Mariposa  |      1953
(2 rows)
```
Here the ONLY before cities indicates that the query should be run over only the cities table, and not tables below cities in the inheritance hierarchy. 
