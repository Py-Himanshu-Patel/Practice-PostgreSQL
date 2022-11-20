# Data Definition

Some of the frequently used data types are integer for whole numbers, numeric for possibly fractional numbers, text for character strings, date for dates, time for time-of-day values, and timestamp for values containing both date and time.

### Default Value
default values are listed after the column data type. For example:
```sql
CREATE TABLE products (
    product_no integer,
    name text,
    price numeric DEFAULT 9.99
);
```
A common example is for a timestamp column to have a default of `CURRENT_TIMESTAMP`.
```sql
CREATE TABLE products (
    product_no SERIAL,
    ...
);
```

### Generated Columns
Column that is always computed from other columns. Thus, it is for columns what a view is for tables. 
Rwo kinds of generated columns: 
- stored (computed when it is written (inserted or updated) and occupies storage as if it were a normal column)
- virtual (occupies no storage and is computed when it is read)
**PostgreSQL** currently implements only **stored generated columns**.

```sql
CREATE TABLE people (
    ...,
    height_cm numeric,
    height_in numeric GENERATED ALWAYS AS (height_cm / 2.54) STORED
);
```
```sql
gorm=# INSERT INTO people VALUES (2), (3), (4);
INSERT 0 3
gorm=# 
gorm=# select * from people;
 height_cm |       height_in        
-----------+------------------------
         2 | 0.78740157480314960630
         3 |     1.1811023622047244
         4 |     1.5748031496062992
(3 rows)
gorm=#
```

### Check Constraints
```sql
CREATE TABLE products (
    product_no integer,
    name text,
    price numeric CHECK (price > 0)
);
```
`CHECK` followed by an expression in parentheses

```sql
CREATE TABLE products (
    product_no integer,
    name text,
    price numeric CHECK (price > 0),
    discounted_price numeric CHECK (discounted_price > 0),
    CHECK (price > discounted_price)
);
```
Names can be assigned to table constraints in the same way as column constraints:
```sql
CREATE TABLE products (
    product_no integer,
    name text,
    price numeric,
    CHECK (price > 0),
    discounted_price numeric,
    CHECK (discounted_price > 0),
    CONSTRAINT valid_discount CHECK (price > discounted_price)
);
```

#### Note
PostgreSQL does not support CHECK constraints that reference table data other than the new or updated row being checked. While a CHECK constraint that violates this rule may appear to work in simple tests, it cannot guarantee that the database will not reach a state in which the constraint condition is false (due to subsequent changes of the other row(s) involved). This would cause a database dump and restore to fail. The restore could fail even when the complete database state is consistent with the constraint, due to rows not being loaded in an order that will satisfy the constraint. If possible, use UNIQUE, EXCLUDE, or FOREIGN KEY constraints to express cross-row and cross-table restrictions.

If what you desire is a one-time check against other rows at row insertion, rather than a continuously-maintained consistency guarantee, a custom trigger can be used to implement that. (This approach avoids the dump/restore problem because pg_dump does not reinstall triggers until after restoring data, so that the check will not be enforced during a dump/restore.)

#### Note
PostgreSQL assumes that CHECK constraints' conditions are immutable, that is, they will always give the same result for the same input row. This assumption is what justifies examining CHECK constraints only when rows are inserted or updated, and not at other times. (The warning above about not referencing other table data is really a special case of this restriction.)

An example of a common way to break this assumption is to reference a user-defined function in a CHECK expression, and then change the behavior of that function. PostgreSQL does not disallow that, but it will not notice if there are rows in the table that now violate the CHECK constraint. That would cause a subsequent database dump and restore to fail. The recommended way to handle such a change is to drop the constraint (using ALTER TABLE), adjust the function definition, and re-add the constraint, thereby rechecking it against all table rows.

### Not-Null Constraints
```sql
CREATE TABLE products (
    product_no integer NOT NULL,
    name text NOT NULL,
    price numeric
);
```
A not-null constraint is always written as a column constraint. A not-null constraint is functionally equivalent to creating a check constraint CHECK (column_name IS NOT NULL), but in PostgreSQL creating an explicit not-null constraint is more efficient. The drawback is that you cannot give explicit names to not-null constraints created this way.

### Unique Constraints
Unique constraints ensure that the data contained in a column, or a group of columns, is unique among all the rows in the table. 
```sql
-- column constraint
CREATE TABLE products (
    product_no integer UNIQUE,
    name text,
    price numeric
);
-- table constraint
CREATE TABLE products (
    product_no integer,
    name text,
    price numeric,
    UNIQUE (product_no)
);
```
To define a unique constraint for a group of columns
```sql
CREATE TABLE product (
    a integer,
    b integer,
    c integer,
    UNIQUE (a, c)
);
```
You can assign your own name for a unique constraint, in the usual way:
```sql
CREATE TABLE products (
    product_no integer CONSTRAINT must_be_different UNIQUE,
    name text,
    price numeric
);
```
Adding a unique constraint will automatically create a unique B-tree index on the column or group of columns listed in the constraint. A uniqueness restriction covering only some rows cannot be written as a unique constraint, but it is possible to enforce such a restriction by creating a unique partial index.

In general, a unique constraint is violated if there is more than one row in the table where the values of all of the columns included in the constraint are equal. 
By default, two null values are not considered equal in this comparison. That means even in the presence of a unique constraint it is possible to store duplicate rows that contain a null value in at least one of the constrained columns. This behavior can be changed by adding the clause `NULLS NOT DISTINCT`
```sql
CREATE TABLE products (
    product_no integer UNIQUE NULLS NOT DISTINCT,
    name text,
    price numeric
);
```

## Primary Keys
A primary key constraint indicates that a column, or group of columns, can be used as a unique identifier for rows in the table. This requires that the values be both unique and not null. 
```sql
CREATE TABLE products (
    product_no integer UNIQUE NOT NULL,
    name text,
    price numeric
);

OR

CREATE TABLE products (
    product_no integer PRIMARY KEY,
    name text,
    price numeric
);
```

Primary keys can span more than one column.
```sql
CREATE TABLE example (
    a integer,
    b integer,
    c integer,
    PRIMARY KEY (a, c)
);
```

## Foreign Keys
```sql
CREATE TABLE products (
    product_no integer PRIMARY KEY,
    name text,
    price numeric
);

CREATE TABLE orders (
    order_id integer PRIMARY KEY,
    product_no integer REFERENCES products (product_no),
    quantity integer
);

-- OR 

CREATE TABLE orders (
    order_id integer PRIMARY KEY,
    product_no integer REFERENCES products,
    quantity integer
);
```
Because in absence of a column list the primary key of the referenced table is used as the referenced column(s).

Of course, the number and type of the constrained columns need to match the number and type of the referenced columns.

A foreign key can also constrain and reference a group of columns.

```sql
CREATE TABLE t1 (
  a integer PRIMARY KEY,
  b integer,
  c integer,
  FOREIGN KEY (b, c) REFERENCES other_table (c1, c2)
);
```

Sometimes it is useful for the “other table” of a foreign key constraint to be the same table; this is called a `self-referential foreign key`. If rows of a table to represent nodes of a tree structure.

```sql
CREATE TABLE tree (
    node_id integer PRIMARY KEY,
    parent_id integer REFERENCES tree,
    name text,
    ...
);
```
A top-level node would have `NULL` `parent_id`, while non-NULL `parent_id` entries would be constrained to reference valid rows of the table.

A table can have more than one foreign key constraint. This is used to implement `many-to-many` relationships between tables. The tables about products and orders, but now you want to allow one order to contain possibly many products.

```sql
CREATE TABLE products (
    product_no integer PRIMARY KEY,
    name text,
    price numeric
);

CREATE TABLE orders (
    order_id integer PRIMARY KEY,
    shipping_address text,
    ...
);

CREATE TABLE order_items (
    product_no integer REFERENCES products,
    order_id integer REFERENCES orders,
    quantity integer,
    PRIMARY KEY (product_no, order_id)
);
```
Primary key overlaps with the foreign keys in the last table.

What if a referrenced record is deleted?

- Disallow deleting a referenced product
- Delete the referring record as well
- Something else?

1. When someone wants to remove a product that is still referenced by an order (via order_items), we disallow it. If someone removes an order, the order items are removed as well:
```sql
CREATE TABLE products (
    product_no integer PRIMARY KEY,
    name text,
    price numeric
);

CREATE TABLE orders (
    order_id integer PRIMARY KEY,
    shipping_address text,
    ...
);

CREATE TABLE order_items (
    product_no integer REFERENCES products ON DELETE RESTRICT,
    order_id integer REFERENCES orders ON DELETE CASCADE,
    quantity integer,
    PRIMARY KEY (product_no, order_id)
);
```

`RESTRICT` prevents deletion of a referenced row. 

`NO ACTION` means that if any referencing rows still exist when the constraint is checked (that is while inserting or updating a referred record), an error is raised; this is the default behavior if you do not specify anything. (The essential difference between these two choices is that `NO ACTION` allows the check to be deferred until later in the transaction, whereas `RESTRICT` does not.) Means If you insert or update a referred record and we have `RESTRICT` constraint then the update/insert will fail immediately while if we have `NO ACTION` then the update/insert will fail then the transaction commit. This difference become visible in case when you need to update the referred record and then update referring record and then commint the transaction.

`CASCADE` specifies that when a referenced row is deleted, row(s) referencing it should be automatically deleted as well. 

`SET NULL` and `SET DEFAULT`. These cause the referencing column(s) in the referencing row(s) to be set to nulls or their default values, respectively, when the referenced row is deleted. If an action specifies `SET DEFAULT` but the default value would not satisfy the foreign key constraint, the operation will fail.

A foreign key must reference columns that either are a primary key or form a unique constraint. This means that the referenced columns always have an index (the one underlying the primary key or unique constraint).

Since a DELETE of a row from the referenced table or an UPDATE of a referenced column will require a scan of the referencing table for rows matching the old value, it is often a good idea to index the referencing columns too.


## System Columns
Every table has several system columns that are implicitly defined by the system. Therefore, these names cannot be used as names of user-defined columns. You do not really need to be concerned about these columns; just know they exist.

## Modifying Tables

### Adding a Column

```sql
gorm=# select * from people;
 height_cm |     height_in      
-----------+--------------------
         3 | 1.1811023622047244
         4 | 1.5748031496062992
         5 | 1.9685039370078740
```

Add a text column to above table
```sql
gorm=# ALTER TABLE people ADD COLUMN description text;
```

```sql
gorm=# select * from people;
 height_cm |     height_in      | description 
-----------+--------------------+-------------
         3 | 1.1811023622047244 | 
         4 | 1.5748031496062992 | 
         5 | 1.9685039370078740 | 
```

The new column is initially filled with whatever default value is given (null if you don't specify a DEFAULT clause).

#### TIP
From PostgreSQL 11, adding a column with a constant default value no longer means that each row of the table needs to be updated when the ALTER TABLE statement is executed. Instead, the default value will be returned the next time the row is accessed, and applied when the table is rewritten, making the ALTER TABLE very fast even on large tables.

However, if the default value is volatile (e.g., clock_timestamp()) each row will need to be updated with the value calculated at the time ALTER TABLE is executed. To avoid a potentially lengthy update operation, particularly if you intend to fill the column with mostly nondefault values anyway, it may be preferable to add the column with no default, insert the correct values using UPDATE, and then add any desired default as described below.

You can also define constraints on the column at the same time, using the usual syntax:
```sql
ALTER TABLE products ADD COLUMN description text CHECK (description <> '');
```

### Removing a Column
```sql
gorm=# ALTER TABLE people DROP COLUMN DESCRIPTION;
ALTER TABLE
gorm=# 
gorm=# select * from people;
 height_cm |     height_in      
-----------+--------------------
         3 | 1.1811023622047244
         4 | 1.5748031496062992
         5 | 1.9685039370078740
(3 rows)
```
Table constraints involving the column are dropped, too. However, if the column is referenced by a foreign key constraint of another table, PostgreSQL will not silently drop that constraint. You can authorize dropping everything that depends on the column by adding `CASCADE`:
```sql
ALTER TABLE people DROP COLUMN description CASCADE;
```

### Adding a Constraint
```sql
ALTER TABLE products ADD CHECK (name <> '');
ALTER TABLE products ADD CONSTRAINT some_name UNIQUE (product_no);
ALTER TABLE products ADD FOREIGN KEY (product_group_id) REFERENCES product_groups;
```
To add a not-null constraint, which cannot be written as a table constraint, use this syntax:
```sql
ALTER TABLE products ALTER COLUMN product_no SET NOT NULL;
```

When ever we put any constaint it is verified then and there. So data should be clean up before we apply that constaint.

### Removing a Constraint
To remove a constraint you need to know its name. If you gave it a name then that's easy. Otherwise the system assigned a generated name, which you need to find out. 

The psql command `\d tablename` can be helpful here
```sql
gorm=# \d cities
                       Table "public.cities"
  Column  |         Type          | Collation | Nullable | Default 
----------+-----------------------+-----------+----------+---------
 name     | character varying(80) |           | not null | 
 location | point                 |           |          | 
Indexes:
    "cities_pkey" PRIMARY KEY, btree (name)
Referenced by:
    TABLE "weather" CONSTRAINT "weather_city_fkey" FOREIGN KEY (city) REFERENCES cities(name)
```
Syntax is
```sql
ALTER TABLE products DROP CONSTRAINT some_name;
```

(If you are dealing with a generated constraint name like $2, don't forget that you'll need to double-quote it to make it a valid identifier.)

As with dropping a column, you need to add CASCADE if you want to drop a constraint that something else depends on. An example is that a foreign key constraint depends on a unique or primary key constraint on the referenced column(s).

```sql
gorm=# ALTER TABLE cities DROP CONSTRAINT cities_pkey;
ERROR:  cannot drop constraint cities_pkey on table cities because other objects depend on it
DETAIL:  constraint weather_city_fkey on table weather depends on index cities_pkey
HINT:  Use DROP ... CASCADE to drop the dependent objects too.
```
Then go for this
```sql
gorm=# ALTER TABLE cities DROP CONSTRAINT cities_pkey CASCADE;
NOTICE:  drop cascades to constraint weather_city_fkey on table weather
ALTER TABLE
```

### Changing a Column's Data Type
```sql
ALTER TABLE products ALTER COLUMN price TYPE numeric(10,2);
```
This will succeed only if each existing entry in the column can be converted to the new type by an implicit cast. If a more complex conversion is needed, you can add a USING clause that specifies how to compute the new values from the old.

```sql
gorm=# select * from cities;
   name    | location 
-----------+----------
 Hyderabad | (1,2)
 Jaipur    | (1,1)
 Hyderabad | (5,6)
```
```sql
gorm=# ALTER TABLE cities ALTER COLUMN location TYPE INTEGER;
ERROR:  column "location" cannot be cast automatically to type integer
HINT:  You might need to specify "USING location::integer".
```
```sql
gorm=# ALTER TABLE cities ALTER COLUMN location TYPE INTEGER USING location::integer;
ERROR:  cannot cast type point to integer
LINE 1: ... alter column location type integer using location::integer;
```
```sql
gorm=# 
gorm=# ALTER TABLE cities ALTER COLUMN location TYPE INTEGER USING location[0]+location[1]::integer;
ALTER TABLE
gorm=# select * from cities;
   name    | location 
-----------+----------
 Hyderabad |        3
 Jaipur    |        2
 Hyderabad |       11
(3 rows)
```

### Renaming a Column
```sql
ALTER TABLE cities RENAME COLUMN location TO distance;
ALTER TABLE
gorm=# select * from cities;
   name    | distance 
-----------+----------
 Hyderabad |        3
 Jaipur    |        2
 Hyderabad |       11
(3 rows)
```

### Renaming a Table
```sql
ALTER TABLE cities RENAME TO route;
gorm=# select * from route;
   name    | distance 
-----------+----------
 Hyderabad |        3
 Jaipur    |        2
 Hyderabad |       11
(3 rows)
```

## Privileges
When an object is created, it is assigned an owner. The owner is normally the role that executed the creation statement. For most kinds of objects, the initial state is that only the owner (or a superuser) can do anything with the object. To allow other roles to use it, privileges must be granted.

There are different kinds of privileges: SELECT, INSERT, UPDATE, DELETE, TRUNCATE, REFERENCES, TRIGGER, CREATE, CONNECT, TEMPORARY, EXECUTE, USAGE, SET and ALTER SYSTEM. 

The right to modify or destroy an object is inherent in being the object's owner, and cannot be granted or revoked in itself. (However, like all privileges, that right can be inherited by members of the owning role

An object can be assigned to a new owner with an ALTER command of the appropriate kind for the object, for example
```sql
ALTER TABLE table_name OWNER TO new_owner;
```
Superusers can always do this; ordinary roles can only do it if they are both the current owner of the object (or a member of the owning role) and a member of the new owning role.

To assign privileges, the `GRANT` command is used. For example, if joe is an existing role, and accounts is an existing table, the privilege to update the table can be granted with:
```sql
GRANT UPDATE ON accounts TO joe;
```
Writing `ALL` in place of a specific privilege grants all privileges that are relevant for the object type. The special “role” name PUBLIC can be used to grant a privilege to every role on the system. 
```sql
REVOKE ALL ON accounts FROM PUBLIC;
```
Ordinarily, only the object's owner (or a superuser) can grant or revoke privileges on an object. However, it is possible to grant a privilege “with grant option”, which gives the recipient the right to grant it in turn to others. If the grant option is subsequently revoked then all who received the privilege from that recipient (directly or through a chain of grants) will lose the privilege.

```sql
GRANT SELECT ON mytable TO PUBLIC;
GRANT SELECT, UPDATE, INSERT ON mytable TO admin;
GRANT SELECT (col1), UPDATE (col1) ON mytable TO miriam_rw;
```
Then psql's `\dp` command would show:
```sql
=> \dp mytable
                                  Access privileges
 Schema |  Name   | Type  |   Access privileges   |   Column privileges   | Policies
--------+---------+-------+-----------------------+-----------------------+----------
 public | mytable | table | miriam=arwdDxt/miriam+| col1:                +|
        |         |       | =r/miriam            +|   miriam_rw=rw/miriam |
        |         |       | admin=arw/miriam      |                       |
(1 row)
```

---

### Look Roles
```sql
gorm=# select * from pg_roles;
          rolname          | rolsuper | rolinherit | rolcreaterole | rolcreatedb | rolcanlogin | rolreplication | rolconnlimit | rolpassword | rolvaliduntil | rolbypassrls | rolconfig |  oid  
---------------------------+----------+------------+---------------+-------------+-------------+----------------+--------------+-------------+---------------+--------------+-----------+-------
 pg_database_owner         | f        | t          | f             | f           | f           | f              |           -1 | ********    |               | f            |           |  6171
 pg_read_all_data          | f        | t          | f             | f           | f           | f              |           -1 | ********    |               | f            |           |  6181
 pg_write_all_data         | f        | t          | f             | f           | f           | f              |           -1 | ********    |               | f            |           |  6182
 pg_monitor                | f        | t          | f             | f           | f           | f              |           -1 | ********    |               | f            |           |  3373
 pg_read_all_settings      | f        | t          | f             | f           | f           | f              |           -1 | ********    |               | f            |           |  3374
 pg_read_all_stats         | f        | t          | f             | f           | f           | f              |           -1 | ********    |               | f            |           |  3375
 pg_stat_scan_tables       | f        | t          | f             | f           | f           | f              |           -1 | ********    |               | f            |           |  3377
 pg_read_server_files      | f        | t          | f             | f           | f           | f              |           -1 | ********    |               | f            |           |  4569
 pg_write_server_files     | f        | t          | f             | f           | f           | f              |           -1 | ********    |               | f            |           |  4570
 pg_execute_server_program | f        | t          | f             | f           | f           | f              |           -1 | ********    |               | f            |           |  4571
 pg_signal_backend         | f        | t          | f             | f           | f           | f              |           -1 | ********    |               | f            |           |  4200
 postgres                  | t        | t          | t             | t           | t           | t              |           -1 | ********    |               | t            |           |    10
```

Another way to get user roles in PSQL
```sql
gorm=# \du
                                   List of roles
 Role name |                         Attributes                         | Member of 
-----------+------------------------------------------------------------+-----------
 postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS | {}
```
Noice that the roles that start with with `pg_` are system roles.

### Create Roles

Useful Link : [AWS Doc on how to create and assign roles with different permission to users](https://aws.amazon.com/blogs/database/managing-postgresql-users-and-roles/#:~:text=Users%2C%20groups%2C%20and%20roles%20are,for%20the%20CREATE%20ROLE%20statement.)

CREATE USER = CREATE ROLE + LOGIN PERMISSSION

`CREATE ROLE` adds a new role to a PostgreSQL database cluster. A role is an entity that can own database objects and have database privileges. You must have CREATE ROLE privilege or be a database superuser to use this command.

```sql
# CREATE ROLE bob;
```
```sql
gorm=# \du
                                   List of roles
 Role name |                         Attributes                         | Member of 
-----------+------------------------------------------------------------+-----------
 bob       | Cannot login                                               | {}
 postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS | {}
```
```
root@fd01a645f479:/# psql -U bob
psql: error: connection to server on socket "/var/run/postgresql/.s.PGSQL.5432" 
failed: FATAL:  role "bob" is not permitted to log in
```

#### Create login roles
```sql
gorm=# CREATE ROLE alice LOGIN;
CREATE ROLE
gorm=# \du
                                   List of roles
 Role name |                         Attributes                         | Member of 
-----------+------------------------------------------------------------+-----------
 alice     |                                                            | {}
 bob       | Cannot login                                               | {}
 postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS | {}
```
Try login 
```bash
$ psql -U alice
psql: error: connection to server on socket "/var/run/postgresql/.s.PGSQL.5432" 
failed: FATAL:  database "alice" does not exist
# error because by default the command above will try connecting to DB named after the user 
# i.e. alice DB.Since we don't have one, either we can create one (befoe alice connect) or 
# we can specify the DB to connect to.
$ psql -U alice -d postgres
psql (14.5 (Debian 14.5-1.pgdg110+1))
Type "help" for help.
postgres=> 
```
To put password auth on user login. First drop the role we created and then recreate as follow.
```sql
gorm=# CREATE ROLE alice LOGIN PASSWORD 'alicepass';
CREATE ROLE
gorm=# \du
                                   List of roles
 Role name |                         Attributes                         | Member of 
-----------+------------------------------------------------------------+-----------
 alice     |                                                            | {}
 postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS | {}
```
But this is not enough to make sure user is prompted for password. We need to ensure 3 things.
1. Make `PGPASSWORD` env variable as empty (in docker container). or else we can save the user password in this env var which get used to each login without password prompt.
```
echo $PGPASSWORD

```
2. Remove `.pgpass` in `/home` location. This file fetch the password store in it without password prompt.
```
# hostname:port:database:username:password
*:*:*:postgres:admin
```
Pass here is stored as plain text.
3. Make `md5` entry instead of `trust` in file `pg_hba.conf`.
```
root@fd01a645f479:/# cd /var/lib/postgresql/data/
root@fd01a645f479:/var/lib/postgresql/data# ls
base	      pg_hba.conf    pg_notify	   pg_stat	pg_twophase  postgresql.auto.conf
global	      pg_ident.conf  pg_replslot   pg_stat_tmp	PG_VERSION   postgresql.conf
pg_commit_ts  pg_logical     pg_serial	   pg_subtrans	pg_wal	     postmaster.opts
pg_dynshmem   pg_multixact   pg_snapshots  pg_tblspc	pg_xact      postmaster.pid
root@fd01a645f479:/var/lib/postgresql/data# vi pg_hba.conf
```
The rules are matched from top. First one that match is used. Change for local Unix Domain from truct to md5.
```
# "local" is for Unix domain socket connections only
local   all             all                                     md5
# IPv4 local connections:
host    all             all             127.0.0.1/32            trust
# IPv6 local connections:
host    all             all             ::1/128                 trust
# Allow replication connections from localhost, by a user with the
# replication privilege.
local   replication     all                                     trust
host    replication     all             127.0.0.1/32            trust
host    replication     all             ::1/128                 trust

host all all all scram-sha-256
```
Then either restart the postgres service
```
sudo service postgresql restart 
```
OR, just resource the config inside the Database
Login without password and resource config
```
# select pg_reload_conf();
 pg_reload_conf 
----------------
 t
(1 row)
```
After this logout and try login again. You will be asked for user password.
```
root@fd01a645f479:/# psql -U alice -d postgres
Password for user alice: 
psql (14.5 (Debian 14.5-1.pgdg110+1))
Type "help" for help.
```

##### Some actions result

Alice can't create database
```sql
gorm=> CREATE DATABASE alice;
ERROR:  permission denied to create database
```
Alice can create table. Because a role when created get public role and public role can create table in public schema of any database. 
```sql
gorm=> create table gorm_table (
gorm(>     name text
gorm(> );
CREATE TABLE
```
But alice can't read, update or delete other tables not owned by it.
```sql
gorm=> \d
           List of relations
 Schema |    Name    | Type  |  Owner   
--------+------------+-------+----------
 public | empsalary  | table | postgres
 public | gorm_table | table | alice
 public | people     | table | postgres
 public | route      | table | postgres
 public | weather    | table | postgres
(5 rows)
gorm=> SELECT * FROM empsalary;
ERROR:  permission denied for table empsalary
```

#### Drop roles
```sql
DROP ROLE role_name;
```
Sometime due to dependency of some schemas, tables on the role. We can't drop it directly. Then we can drop the dependent data objects before dropping the role.
```sql
DROP OWNER BY role_name;
DROP ROLE role_name;
```

#### Check Current User and Change Roles
```sql
postgres=> SELECT current_user;
 current_user 
--------------
 alice
(1 row)
```
Another way is 
```sql
postgres=> \conninfo
You are connected to database "postgres" as user "alice" via socket in "/var/run/postgresql" at port "5432".
```
Change to other user as
```sql
postgres=> \c - bob
Password for user bob: 
You are now connected to database "postgres" as user "bob".
```

#### Practice Role and User creation
Generally when we create a table it is created under public schema of the connected database.

```sql
SELECT 
      r.rolname, 
      ARRAY(SELECT b.rolname
            FROM pg_catalog.pg_auth_members m
            JOIN pg_catalog.pg_roles b ON (m.roleid = b.oid)
            WHERE m.member = r.oid) as memberof
FROM pg_catalog.pg_roles r
WHERE r.rolname NOT IN ('pg_signal_backend','rds_iam',
                        'rds_replication','rds_superuser',
                        'rdsadmin','rdsrepladmin')
ORDER BY 1;
          rolname          |                           memberof                           
---------------------------+--------------------------------------------------------------
 alice                     | {}
 bob                       | {}
 pg_database_owner         | {}
 pg_execute_server_program | {}
 pg_monitor                | {pg_read_all_settings,pg_read_all_stats,pg_stat_scan_tables}
 pg_read_all_data          | {}
 pg_read_all_settings      | {}
 pg_read_all_stats         | {}
 pg_read_server_files      | {}
 pg_stat_scan_tables       | {}
 pg_write_all_data         | {}
 pg_write_server_files     | {}
 postgres                  | {}
(13 rows)


                                 List of databases
   Name    |  Owner   | Encoding |  Collate   |   Ctype    |   Access privileges   
-----------+----------+----------+------------+------------+-----------------------
 mydb      | postgres | UTF8     | en_US.utf8 | en_US.utf8 | 
 postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 | 
 template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
(4 rows)
-- remove create access of PUBLIC from public schema of database mydb
mydb=# REVOKE CREATE ON SCHEMA public FROM PUBLIC;
REVOKE
-- remove all permission on database mydb from public
mydb=# REVOKE ALL ON DATABASE mydb FROM PUBLIC;
REVOKE
```
If you are trying to create a `read-only` user. Even if you restrict all privileges, the permissions inherited via the `public` role allow the user to create objects in the public schema.

To fix this, you should revoke the default create permission on the `public schema` from the `public role` using the following SQL statement `REVOKE CREATE ON SCHEMA public FROM PUBLIC;`
Make sure that you are the owner of the public schema or are part of a role that allows you to run this SQL statement.

The following statement revokes the public role’s ability to connect to the database: `REVOKE ALL ON DATABASE mydb FROM PUBLIC;`

#### Privilages are given in below order
Database > Schema > Table > Row

#### Create Read Only Role
```sql
CREATE ROLE readonly;
```
This is a base role with no permissions and no password. It cannot be used to log in to the database. Grant this role permission to connect to your target database named `mydb`:
```sql
GRANT CONNECT ON DATABASE mydb TO readonly;
```
The next step is to grant this role usage access to your schema. Let’s assume the schema is named `myschema`. This step grants the readonly role permission to perform some activity inside the schema.
```sql
GRANT USAGE ON SCHEMA myschema TO readonly;

mydb=# \du
                                   List of roles
 Role name |                         Attributes                         | Member of 
-----------+------------------------------------------------------------+-----------
 postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS | {}
 readonly  | Cannot login                                               | {}
```
Look the role can't login. Because the roles are not for login purpose but users are. So this role is just to define a policy and user will use this roles access.

The next step is to grant the `readonly` role access to run select on the required tables.
```sql
mydb=# CREATE TABLE mytable1 (name text);
CREATE TABLE
mydb=# CREATE TABLE mytable2 (name text);
CREATE TABLE
mydb=# 
mydb=# \d
          List of relations
 Schema |   Name   | Type  |  Owner   
--------+----------+-------+----------
 public | mytable1 | table | postgres
 public | mytable2 | table | postgres
(2 rows)
```
```sql
GRANT SELECT ON TABLE mytable1, mytable2 TO readonly;
```
The preceding SQL statement grants SELECT access to the readonly role on all the existing tables and views in the schema myschema. Note that any new tables that get added in the future will not be accessible by the readonly user. To help ensure that new tables and views are also accessible, run the following statement to grant permissions automatically:
```sql
ALTER DEFAULT PRIVILEGES IN SCHEMA myschema GRANT SELECT ON TABLES TO readonly;
```

#### Create Read Write Role
```sql
CREATE ROLE readwrite;
GRANT CONNECT ON DATABASE mydb TO readwrite;
GRANT USAGE, CREATE ON SCHEMA myschema TO readwrite;    -- for access and create of object in schema
GRANT SELECT, INSERT, UPDATE, DELETE ON TABLE mytable1, mytable2 TO readwrite;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA myschema TO readwrite;
ALTER DEFAULT PRIVILEGES IN SCHEMA myschema GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO readwrite;
```

#### Creating DB User
```sql
mydb=# CREATE USER myuser1 WITH PASSWORD 'secret_passwd';
CREATE ROLE
mydb=# 
mydb=# GRANT readonly TO myuser1;
GRANT ROLE
```

```sql
          rolname          |                           memberof                           
---------------------------+--------------------------------------------------------------
 myuser1                   | {readonly}
 postgres                  | {}
 readonly                  | {}
 readwrite                 | {}
```

## Check and Kill the stale process and connection
Check all the process running in DB
```sql
gorm=# SELECT pid, usename, datname, query FROM pg_stat_activity; 
 pid  | usename  | datname |                           query                            
------+----------+---------+------------------------------------------------------------
   31 |          |         | 
   33 | postgres |         | 
 2022 | alice    | alice   | 
   72 | postgres | gorm    | SELECT pid, usename, datname, query FROM pg_stat_activity;
 2136 | alice    | alice   | 
   29 |          |         | 
   28 |          |         | 
   30 |          |         | 
(8 rows)
```
Each time we fire above query this is also a process
```sql
gorm=# select pg_backend_pid();
 pg_backend_pid 
----------------
             72
(1 row)
```
Lets get all the process except this one
```sql
gorm=# SELECT pid, usename, datname, query FROM pg_stat_activity WHERE pid <> pg_backend_pid(); 
 pid  | usename  | datname | query 
------+----------+---------+-------
   31 |          |         | 
   33 | postgres |         | 
 2022 | alice    | alice   | 
 2136 | alice    | alice   | 
   29 |          |         | 
   28 |          |         | 
   30 |          |         | 
(7 rows)
```
Kill one process
```sql
gorm=# SELECT pg_terminate_backend(2136);
 pg_terminate_backend 
----------------------
 t
(1 row)
```
Similarly Kill all irrelevant process
```sql
gorm=# SELECT pid, usename, datname, query, pg_terminate_backend(pid) FROM pg_stat_activity WHERE pid <> pg_backend_pid(); 
WARNING:  PID 29 is not a PostgreSQL server process
WARNING:  PID 28 is not a PostgreSQL server process
WARNING:  PID 30 is not a PostgreSQL server process
 pid  | usename  | datname | query | pg_terminate_backend 
------+----------+---------+-------+----------------------
 2261 |          |         |       | t
 2263 | postgres |         |       | t
   29 |          |         |       | f
   28 |          |         |       | f
   30 |          |         |       | f
(5 rows)
```

This will remove the old stale connection
```sql
gorm=# DROP DATABASE alice;

ERROR:  database "alice" is being accessed by other users
DETAIL:  There are 2 other sessions using the database.
```
```sql
gorm=# DROP DATABASE alice;
DROP DATABASE
```

### Getting Schema name and table/relation in those schema
```sql
gorm=# \z
                               Access privileges
 Schema |   Name    | Type  | Access privileges | Column privileges | Policies 
--------+-----------+-------+-------------------+-------------------+----------
 public | empsalary | table |                   |                   | 
 public | people    | table |                   |                   | 
 public | route     | table |                   |                   | 
 public | weather   | table |                   |                   | 
(4 rows)

-- Get all schemas name
mydb=# \dn
   List of schemas
   Name   |  Owner   
----------+----------
 myschema | postgres
 public   | postgres
(2 rows)
```

## Row Security Policies
Row-Level Security (RLS). It's a way for PostgreSQL to limit what rows of a table are visible to a query. 

When row security is enabled on a table (with `ALTER TABLE ... ENABLE ROW LEVEL SECURITY`), all normal access to the table for selecting rows or modifying rows must be allowed by a row security policy. (However, the table's owner is typically not subject to row security policies.) If no policy exists for the table, a default-deny policy is used, meaning that no rows are visible or can be modified. Operations that apply to the whole table, such as `TRUNCATE` and `REFERENCES`, are not subject to row security.

Row security policies can be specific to commands, or to roles, or to both. A policy can be specified to apply to ALL commands, or to SELECT, INSERT, UPDATE, or DELETE. Multiple roles can be assigned to a given policy, and normal role membership and inheritance rules apply.

Superusers and roles with the `BYPASSRLS` attribute always bypass the row security system when accessing a table. Table owners normally bypass row security as well, though a table owner can choose to be subject to row security with `ALTER TABLE ... FORCE ROW LEVEL SECURITY`.

Enabling and disabling row security, as well as adding policies to a table, is always the privilege of the table owner only.

Policies are created using the `CREATE POLICY` command, altered using the `ALTER POLICY` command, and dropped using the `DROP POLICY` command. To enable and disable row security for a given table, use the `ALTER TABLE` command.

Here is how to create a policy on the account relation to allow only members of the managers role to access rows, and only rows of their accounts:
```sql
CREATE TABLE accounts (manager text, company text, contact_email text);

ALTER TABLE accounts ENABLE ROW LEVEL SECURITY;

CREATE POLICY account_managers ON accounts TO managers
    USING (manager = current_user);
```
So a manager cannot SELECT, UPDATE, or DELETE existing rows belonging to a different manager.
So rows belonging to a different manager cannot be created via `INSERT` or `UPDATE`.

If no role is specified, or the special user name PUBLIC is used, then the policy applies to all users on the system. To allow all users to access only their own row in a users table, a simple policy can be used:
```sql
CREATE POLICY user_policy ON users
    USING (user_name = current_user);
```
To use a different policy for rows that are being added to the table compared to those rows that are visible, multiple policies can be combined. This pair of policies would allow all users to view all rows in the users table, but only modify their own:
```sql
CREATE POLICY user_sel_policy ON users
    FOR SELECT
    USING (true);
CREATE POLICY user_mod_policy ON users
    USING (user_name = current_user);
```
Row security can also be disabled with the `ALTER TABLE` command. Disabling row security does not remove any policies that are defined on the table; they are simply ignored

The table passwd emulates a Unix password file:
```sql
CREATE TABLE passwd (
    user_name text UNIQUE NOT NULL,
    pwhash text,
    uid int PRIMARY KEY,
    gid int NOT NULL,
    real_name text NOT NULL,
    home_phone text,
    extra_info text,
    home_dir text NOT NULL,
    shell text NOT NULL
);


CREATE ROLE admin;  -- Administrator
CREATE ROLE bob;    -- Normal user
CREATE ROLE alice;  -- Normal user

-- Populate the table
INSERT INTO passwd VALUES
  ('admin','xxx',0,0,'Admin','111-222-3333',null,'/root','/bin/dash');
INSERT INTO passwd VALUES
  ('bob','xxx',1,1,'Bob','123-456-7890',null,'/home/bob','/bin/zsh');
INSERT INTO passwd VALUES
  ('alice','xxx',2,1,'Alice','098-765-4321',null,'/home/alice','/bin/zsh');

-- Be sure to enable row-level security on the table
ALTER TABLE passwd ENABLE ROW LEVEL SECURITY;

-- At this point we can't access table via any of the roles 
new=# SET ROLE admin;
SET
new=> SELECT * FROM passwd ;
ERROR:  permission denied for table passwd
new=> SET ROLE alice;
SET
new=> SELECT * FROM passwd ;
ERROR:  permission denied for table passwd
new=> SET ROLE bob;
SET
new=> SELECT * FROM passwd ;
ERROR:  permission denied for table passwd

-- Create policies
-- Administrator can see all rows and add any rows
CREATE POLICY admin_all ON passwd TO admin USING (true) WITH CHECK (true);
-- This will need you to be in role of owner (postges here) - SET ROLE postgres
-- Now we can access the table by setting role to admin
SELECT * FROM passwd ;

-- Normal users can view all rows
CREATE POLICY all_view ON passwd FOR SELECT USING (true);
-- Normal users can update their own records, but
-- limit which shells a normal user is allowed to set
CREATE POLICY user_mod ON passwd FOR UPDATE
  USING (current_user = user_name)
  WITH CHECK (
    current_user = user_name AND
    shell IN ('/bin/bash','/bin/sh','/bin/dash','/bin/zsh','/bin/tcsh')
  );

-- Check the list of policies as
new=# SELECT schemaname,tablename,policyname,permissive,roles,cmd FROM pg_policies;
 schemaname | tablename | policyname | permissive |  roles   |  cmd   
------------+-----------+------------+------------+----------+--------
 public     | passwd    | user_mod   | PERMISSIVE | {public} | UPDATE
 public     | passwd    | all_view   | PERMISSIVE | {public} | SELECT
 public     | passwd    | admin_all  | PERMISSIVE | {admin}  | ALL
(3 rows)

-- But Role of alice and bob still can't access the table passwd as we yet not 
-- granted the policy created to respective roles. This can be seen in above policies list

-- Allow admin all normal rights
GRANT SELECT, INSERT, UPDATE, DELETE ON passwd TO admin;
-- Users only get select access on public columns
GRANT SELECT
  (user_name, uid, gid, real_name, home_phone, extra_info, home_dir, shell)
  ON passwd TO public;
-- Allow users to update certain columns
GRANT UPDATE
  (pwhash, real_name, home_phone, extra_info, shell)
  ON passwd TO public;
```
Let's check how it behaves
```sql
-- admin can view all rows and fields
postgres=> set role admin;
SET
postgres=> table passwd;
 user_name | pwhash | uid | gid | real_name |  home_phone  | extra_info | home_dir    |   shell
-----------+--------+-----+-----+-----------+--------------+------------+-------------+-----------
 admin     | xxx    |   0 |   0 | Admin     | 111-222-3333 |            | /root       | /bin/dash
 bob       | xxx    |   1 |   1 | Bob       | 123-456-7890 |            | /home/bob   | /bin/zsh
 alice     | xxx    |   2 |   1 | Alice     | 098-765-4321 |            | /home/alice | /bin/zsh
(3 rows)

-- Test what Alice is able to do
postgres=> set role alice;
SET
postgres=> table passwd;
ERROR:  permission denied for relation passwd
postgres=> select user_name,real_name,home_phone,extra_info,home_dir,shell from passwd;
 user_name | real_name |  home_phone  | extra_info | home_dir    |   shell
-----------+-----------+--------------+------------+-------------+-----------
 admin     | Admin     | 111-222-3333 |            | /root       | /bin/dash
 bob       | Bob       | 123-456-7890 |            | /home/bob   | /bin/zsh
 alice     | Alice     | 098-765-4321 |            | /home/alice | /bin/zsh
(3 rows)

postgres=> update passwd set user_name = 'joe';
ERROR:  permission denied for relation passwd
-- Alice is allowed to change her own real_name, but no others
postgres=> update passwd set real_name = 'Alice Doe';
UPDATE 1
postgres=> update passwd set real_name = 'John Doe' where user_name = 'admin';
UPDATE 0
postgres=> update passwd set shell = '/bin/xx';
ERROR:  new row violates WITH CHECK OPTION for "passwd"
postgres=> delete from passwd;
ERROR:  permission denied for relation passwd
postgres=> insert into passwd (user_name) values ('xxx');
ERROR:  permission denied for relation passwd
-- Alice can change her own password; RLS silently prevents updating other rows
postgres=> update passwd set pwhash = 'abc';
UPDATE 1
```

All of the policies constructed thus far have been permissive policies, meaning that when multiple policies are applied they are combined using the “OR” Boolean operator. While permissive policies can be constructed to only allow access to rows in the intended cases, it can be simpler to combine permissive policies with restrictive policies (which the records must pass and which are combined using the “AND” Boolean operator).

We add a restrictive policy to require the administrator to be connected over a **local Unix socket** to access the records of the passwd table:
```sql
CREATE POLICY admin_local_only ON passwd AS RESTRICTIVE TO admin
    USING (pg_catalog.inet_client_addr() IS NULL);
```
Connecting from Unix socket
```sql
root@fd01a645f479:/# psql -U postgres
Password for user postgres:
postgres=> \c new 
You are now connected to database "new" as user "postgres".
new=# set role admin;
SET
new=> select * from passwd ;
 user_name | pwhash | uid | gid | real_name |  home_phone  | extra_info |  home_di
r   |   shell   
-----------+--------+-----+-----+-----------+--------------+------------+---------
----+-----------
 admin     | xxx    |   0 |   0 | Admin     | 111-222-3333 |            | /root   
    | /bin/dash
 bob       | xxx    |   1 |   1 | Bob       | 123-456-7890 |            | /home/bo
b   | /bin/zsh
 alice     | xxx    |   2 |   1 | Alice     | 098-765-4321 |            | /home/al
ice | /bin/zsh
```
Connectinc over IP
```sql
root@fd01a645f479:/# psql -U postgres -h localhost
postgres=# \c new
You are now connected to database "new" as user "postgres".
new=# set role admin;
SET
new=> select * from passwd ;
 user_name | pwhash | uid | gid | real_name | home_phone | extra_info | home_dir |
 shell 
-----------+--------+-----+-----+-----------+------------+------------+----------+
-------
(0 rows)
```

In some contexts it is important to be sure that row security is not being applied. For example, when taking a backup, it could be disastrous if row security silently caused some rows to be omitted from the backup. In such a situation, you can set the row_security configuration parameter to off.

```sql
-- definition of privilege groups
CREATE TABLE groups (group_id int PRIMARY KEY,
                     group_name text NOT NULL);

INSERT INTO groups VALUES
  (1, 'low'),
  (2, 'medium'),
  (5, 'high');

GRANT ALL ON groups TO alice;  -- alice is the administrator
GRANT SELECT ON groups TO public;

-- definition of users' privilege levels
CREATE TABLE users (user_name text PRIMARY KEY,
                    group_id int NOT NULL REFERENCES groups);

INSERT INTO users VALUES
  ('alice', 5),
  ('bob', 2),
  ('mallory', 2);

GRANT ALL ON users TO alice;
GRANT SELECT ON users TO public;

-- table holding the information to be protected
CREATE TABLE information (info text,
                          group_id int NOT NULL REFERENCES groups);

INSERT INTO information VALUES
  ('barely secret', 1),
  ('slightly secret', 2),
  ('very secret', 5);

ALTER TABLE information ENABLE ROW LEVEL SECURITY;

-- a row should be visible to/updatable by users whose security group_id is
-- greater than or equal to the row's group_id
CREATE POLICY fp_s ON information FOR SELECT
  USING (group_id <= (SELECT group_id FROM users WHERE user_name = current_user));
CREATE POLICY fp_u ON information FOR UPDATE
  USING (group_id <= (SELECT group_id FROM users WHERE user_name = current_user));

-- we rely only on RLS to protect the information table
GRANT ALL ON information TO public;
```
Now suppose that alice wishes to change the “slightly secret” information, but decides that mallory should not be trusted with the new content of that row, so she does:
```sql
BEGIN;
UPDATE users SET group_id = 1 WHERE user_name = 'mallory';
UPDATE information SET info = 'secret from mallory' WHERE group_id = 2;
COMMIT;
```
