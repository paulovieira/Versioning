# pg-versioning

**NOTE:** this is a forked version of https://github.com/depesz/Versioning

This project strives to provide a simple way to manage changes to a Postgres 
database.

Instead of making changes on the development server, then finding
differences between production and development, deciding which ones
should be installed on production, and finding a way to install them -
you start with writing diffs directly!

More detailed description is (so far) in here:
http://www.depesz.com/index.php/2010/08/22/versioning/


#### Usage example

Suppose the initial design of the database was badly done. For instance:
```sql
DO $$

DECLARE
patch_exists int := _v.register_patch('first-patch');

BEGIN

IF patch_exists THEN
    RETURN;
END IF;

/* the actual code to change to the database starts here */

CREATE TABLE stuff(id int primary key, email text unique);

END;
$$ 
```

Meanwhile we have some data inserted by the application:
```sql
INSERT INTO stuff VALUES (10, 'c'), (5, 'b'), (20, 'a');
```

Sometime in the future we conclude that a new separate `users` table is needed. It should contain the data in `stuff.email` (plus other new columns). A new patch is created to handle these changes:
```sql
DO $$

DECLARE
patch_exists int := _v.register_patch('second-patch');

BEGIN

IF patch_exists THEN
    RETURN;
END IF;

/* the actual code to change to the database starts here */

-- add a new "users" table and fill it with the existing data
CREATE TABLE users(id serial primary key, email text unique);

INSERT INTO users(email)
    SELECT email FROM stuff 
    ORDER BY id;

-- add a new column (fk) in the old "stuff" table and fill with the 
-- correct reference
ALTER TABLE stuff 
    ADD COLUMN user_id int REFERENCES users(id);

UPDATE stuff 
    SET user_id = u.id
    FROM ( SELECT * FROM users ) u
    WHERE stuff.email = u.email;

-- delete the "email" column in the old table (not needed anymore)
ALTER TABLE stuff 
    DROP COLUMN email;

END;
$$ 
```

The statements relative to 'second-patch' will be executed only once because we return early if the patch has been already applied.

#### Relation to the IF NOT EXISTS clause

For simple operations like `CREATE TABLE` we could simply use the `IF NOT EXISTS` clause. The usefulness of pg-versioning is seen when there are more complex changes involved, in particular when there's data that has to be moved (an operation that should be done only once).

But even for simple operations it might make sense to use pg-versioning instead of relying on `IF NOT EXISTS`. For instance, suppose a existing table is not needed anymore. We could use a `DROP TABLE IF EXISTS` in the scripts. Without pg-versioning we will be creating and droppping that table everytime the scripts are run, without any purpose.

#### Installation

To install pg-versioning simply run the `pg-versioning.sql` script in your
database (all of them: production, stage, test, devel, ...).

To uninstall, just run 
```sql
DROP SCHEMA _v CASCADE;
```


#### Differences relatively to the original project

The original project was started by [depesz](https://www.depesz.com/) and lives here: https://github.com/depesz/Versioning

This fork has been changed a little in  the `register_patch` function: it returns an `INT4` instead of `setof INT4`:
* If the patch is already registered, returns 1 and won't register it (return early). Can be used to skip the execution of the patch, as shown in the example above.
* If the patch does not exist:
    - If there are conflicting patches already registered, an exception is raised (as in the original version)
    - If there are pre-requisites patches not registered, an exception is raised (as in the original version)
    - otherwise, the patch is registered and returns 0

The `unregister_patch` function was changed similarly:
* if the patch is required by some other patch, an exception is raised (as in the original version)
* if the patch is not registered, an exception is raised (as in the original version)
* otherwise, the patch is unregistered (deleted) and returns 0