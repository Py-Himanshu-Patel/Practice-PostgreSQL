# SQL Syntax

### Calling Functions
```sql
CREATE FUNCTION concat_lower_or_upper(a text, b text, uppercase boolean DEFAULT false)
RETURNS text
AS
$$
 SELECT CASE
        WHEN $3 THEN UPPER($1 || ' ' || $2)
        ELSE LOWER($1 || ' ' || $2)
        END;
$$
LANGUAGE SQL IMMUTABLE STRICT;
```
#### Using Positional Notation
```SQL
gorm=# SELECT concat_lower_or_upper('Hello', 'World', true);
 concat_lower_or_upper 
-----------------------
 HELLO WORLD
(1 row)

gorm=# SELECT concat_lower_or_upper('Hello', 'World', false);
 concat_lower_or_upper 
-----------------------
 hello world
(1 row)
```
#### Using Named Notation
Each argument's name is specified using `=>` to separate it from the argument expression. For example:
```sql
SELECT concat_lower_or_upper(a => 'Hello', b => 'World');
 concat_lower_or_upper
-----------------------
 hello world
(1 row)
SELECT concat_lower_or_upper(a => 'Hello', uppercase => true, b => 'World');
 concat_lower_or_upper
-----------------------
 HELLO WORLD
(1 row)
```
An older syntax based on ":=" is supported for backward compatibility:
```sql
SELECT concat_lower_or_upper(a := 'Hello', uppercase := true, b := 'World');
 concat_lower_or_upper
-----------------------
 HELLO WORLD
(1 row)
```
#### Using Mixed Notation
```sql
SELECT concat_lower_or_upper('Hello', 'World', uppercase => true);
 concat_lower_or_upper
-----------------------
 HELLO WORLD
(1 row)
```
