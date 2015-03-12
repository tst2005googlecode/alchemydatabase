# ALCHEMY DATABASE COMMANDS #

Alchemy Database supports:
  1. [An OLTP optimised subset of SQL](http://code.google.com/p/alchemydatabase/wiki/CommandReference#Supported_SQL)
  1. [embedded LUA w/ client-server functionality](http://code.google.com/p/alchemydatabase/wiki/CommandReference#LUAFUNC_Command)
  1. [Extensions to SQL designed to optimise common OLTP use cases](http://code.google.com/p/alchemydatabase/wiki/CommandReference#Beyond_SQL)
  1. [ALL redis commands](http://code.google.com/p/alchemydatabase/wiki/CommandReference#REDIS_Commands)<br />

## NOTES ##
Supported SQL data types are currently UNSIGNED INT, UNSIGNED LONG, U128 (128 bit INTEGER), TEXT, and FLOAT.<br />
Datetime should be represented as a UNSIGNED INT/LONG or FLOAT<br />
TEXT uses variable length storage and compression and can be substituted for any VARCHAR(), CHAR(), BLOB, etc....<br />

## Supported SQL ##

### Create/Drop Operations: ###
  * `CREATE TABLE customer (id INT, group_id INT, name TEXT, phone text, age INT);`
  * `CREATE INDEX i_cg ON customer (group_id);`
  * `CREATE INDEX i_cga ON customer (group_id, age);` - Compound Indexes
  * `CREATE UNIQUE INDEX iu_cg ON customer (group_id);`
  * `DROP TABLE customer`
  * `DROP INDEX cust_group_ind`
  * `DESC customer` - lists columns & provides detailed memory usage info
  * `DUMP customer` - dumps the table
  * `DUMP customer TO MYSQL` - dumps the table in mysql format
  * `DUMP customer TO FILE fname` - dumps the table to the file "fname"
  * `ALTER TABLE customer ADD COLUMN state INT` - "Alter Table" is instantaneous, does not physically change rows (just the table's metadata)

### Single Row Operations ###
  * `INSERT INTO customer VALUES (1,300,’John Doe’,’301-933-1891’,30);`
  * `SELECT group_id, phone FROM customer WHERE id = 1`
  * `DELETE FROM customer WHERE id = 1`
  * `UPDATE customer SET phone = ‘703-933-1891’, group_id = 4 WHERE id = 1`
  * `INSERT INTO customer VALUES (1,300,’John Doe’,’301-933-1891’,30) RETURN SIZE;` - returns the sizes of the row, table, and indices
  * `REPLACE INTO customer VALUES (1,300,’John Doe’,’703-387-7777’,30)`
  * `INSERT INTO customer VALUES (1,300,’John Doe’,’703-387-7777’,30) ON DUPLICATE KEY UPDATE SET phone = ’703-387-7777’`
  * `INSERT INTO customer VALUES (1,300,’John Doe’,’301-933-1891’,30) (2,300,’Jane Smith’,’301-946-2222’,22) (1,300,’John Doe’,’202-933-444’,55)` - Bulk Insert
  * `INSERT INTO customer (id, name, phone) VALUES (1,’John Doe’,’301-933-1891’);` - Partial Insert, Alchemy rows are streams, so only the data in the row will be stored, only 1-4 bytes (depending on row's stream-length) for NULL placeholders

### Index Operations ###
  * `SELECT group_id, phone FROM customer WHERE group_id BETWEEN 300 AND 400 [ORDER BY column LIMIT n OFFSET m]`
  * `SELECT group_id, phone FROM customer WHERE group_id IN (300, 301, 307, 311) [ORDER BY column LIMIT n OFFSET m]`
  * `DELETE FROM customer WHERE group_id BETWEEN 300 AND 400 [ORDER BY column LIMIT n OFFSET m]`
  * `UPDATE customer SET phone = ‘703-933-1891’, group_id = 4 WHERE group_id = 300 [ORDER BY column LIMIT n OFFSET m]`

### SIMPLE Update Expressions ###
> UPDATE expressions currently support the following functionalities `(+=, -=, *=, /=, %=, ^=)`
```
UPDATE customer SET group_id = group_id * 4, age = age + 1 WHERE id = 1
```

> Simple UPDATEs are done in C, [complex UPDATEs](http://code.google.com/p/alchemydatabase/wiki/CommandReference?ts=1317055830&updated=CommandReference#Complex_UPDATEs_via_lua-rewriting) are done in LUA.

### Where Clause Grammar ###
```
WHERE {[pk/fk=X],[pk/fk BETWEEN X AND Y],[pk/fk IN (X,Y,Z)]} {AND [column=X],[column!=X],[column<X],[column<=X],[column>X],[column>=X],[column IN (X,Y,Z)]}*
```
> ORDER BY single and multiple columns with LIMIT OFFSET
```
WHERE fk = 7 ORDER BY col1 DESC, col2, col3 ASC LIMIT 10 OFFSET 100
```

### JOINS ###
Alchemy currently supports chained Table Joins, and will soon support tree joins
### Join via single Index ###
```
SELECT c.name, g.slogan
 FROM customer c, group g
 WHERE c.group_id = g.id AND g.id BETWEEN 300 AND 400
       [ORDER BY table.column LIMIT n OFFSET m]
```
### EXPLAIN Join-Plan ###
```
EXPLAIN
 SELECT customer.name, group.slogan FROM customer, group
 WHERE customer.group_id = group.id AND group.id BETWEEN 300 AND 400
```

### Create table from Join ###
```
CREATE TABLE new_table
AS SELECT customer.name, group.slogan
FROM customer, group
WHERE customer.group_id = group.id AND
      group.id IN (300, 301, 307, 311)
```

### Full Table Scan ###
Find unindexed data – _not recommended, should be avoided when possible_
```
SCAN id, phone FROM customer WHERE name = ‘bill’ AND age BETWEEN 18 AND 25
```
> _the SCAN command makes it **impossible** to accidentally do a full table scan w/ SELECT, which is worse than in a Disk-Based RDBMS because AlchemyDB is single-threaded, so SCANs block during their duration._

## LUAFUNC Command ##
Alchemy Database has embedded [LUA](http://www.lua.org/about.html) <br />
[IBM's lua description](http://www.ibm.com/developerworks/linux/library/l-embed-lua/index.html): _The Lua programming language is a small scripting language specifically designed to be embedded in other programs. Lua’s C API allows exceptionally clean and simple code both to call Lua from C, and to call C from Lua”_<br />
Alchemy Database has embedded Lua in its C-server and C in its embedded Lua :)<br />
The speed of embedded LUA commands in Alchemy Database is impressive: simple LUA routines run at about the same speed as simple Alchemy look read/write-ops.

### LUA alchemy() function ###
The lua function "alchemy()" will call the Alchemy Database server internally as if it were a client call.<br />
Function to set a user's last\_login and return his status
```
  function getset_last_login(userid, ts)
    alchemy('SET', 'user:' .. userid .. ':last_login', ts);
    return alchemy('GET', 'user:' .. userid .. ':status');"
  end
```

### Adding Lua functions at run-time ###
Loading lua files (containing lua functions) into the Alchemy Database server can be done either by loading lua files of individual lines of lua.
```
./alchemy-cli INTERPRET LUAFILE "helper.lua"
./alchemy-cli INTERPRET LUA "function echo(x) return x; end"
```

### calling Lua functions via LUAFUNC ###
once a lua function has been imported into Alchemy, it is simple to call it. <br />
As an example, suppose the following function has been added to AlchemyDB's Lua universe via a "INTERPRET" call:
```
  function getuserdata(arg1, arg2, arg3)
    ...
    return alchemy('GET', arg3');
  end
```
To then call the function 'getuserdata' w/ the 3 arguments it needs, the command "LUAFUNC" is used:
```
   LUAFUNC getuserdata 'session_777' 'user_123' 'last_login'
```
and the data returned in the function _getuserdata_ will be output in redis' protocol (to be read by the client)

## Beyond SQL ##
SQL is a powerful **standardised** language, adding specific use case optimisations to it, can greatly improve the performance & ease-of-programming of many common OLTP use cases.

### CREATE INDEX via BATCH ###
Creating an Index is a blocking op in AlchemyDB (like all other ops). Creating an index on a very large table (i.e. 20 million rows) can take a prohibitively long time (e.g. 7-30 secs (depending on index repetitiveness) for 20million rows on a 2.8GHz CPU), so Alchemy provides a mechanism to create a single index via a batch. Unlike
[Alchemy Cursors](http://code.google.com/p/alchemydatabase/wiki/CommandReference#Alchemy_Cursors) creating an index is an ACID compliant operation as the index is not active until the ENTIRE Index has been built.
To Create an Index via a batch job, simply call
```
CREATE INDEX i_t ON t (fk) LIMIT 1000 
```
repetitively until ZERO rows are returned. When zero rows are returned the Index has been created.
In practice, creating a HUGE Index via a batch will negatively effect (during its build) all other read/write ops by about 10-20% at full load (when the load is not full, creating an index via a batch job goes more or less unnoticed).

### ORDER BY INDEX ###
ORDER BY INDEX stores values sorted to the specified column's value (instead of sorted to the default PK ASC standard in RDBMSes). This is very useful for dashboard updates as the ORDER BY sorting is done at INSERT time (w/ zero overhead), instead of query time (requires row-buffering + qsort). Example:
```
  CREATE TABLE tweets (pk INT, userid INT, unix_ts INT, message TEST);
  CREATE INDEX i_tweets ON tweets (userid) ORDER BY unix_ts
```
Querying this table, using the `i_tweets` index will return rows PRE-sorted to `unix_ts` (unix timestamp), so specifying `unix_ts` in the ORDER BY clause will not require any server side sorting. Here is a dashboard query that will require zero row-buffering and zero sorting.
```
    SELECT * FROM tweets WHERE userid = 999 ORDER BY unix_ts DESC LIMIT 10
```

### Complex UPDATEs via lua-rewriting ###
Complex UPDATEs are rewritten into lua functions and the computation of the UPDATE is done in LUA ... and this is done transparently.

Here is an example (because this is totally confusing).Given the complex UPDATE: (NOTE: the function SHA1() is baked into AlchemyDB (coded in C))
```
UPDATE table SET col2 = math.random(1000,2000),
                 col3 = SHA1((col3 + (col2 * col4))),
                 col4 = col4 * 10000
WHERE fk = 2
```
The 3 update column statements (col2, col3, col4) are analysed for complexity.
  * `col2=math.random(1000,2000)`"   -> complex
  * `col3=SHA1((col3+(col2*col4)))`" -> complex
  * `col4=col4*10000`"               -> simple.

So the update to col4 (the simple one) is done in C (just like in any other RDBMS). The complex updates (col2,col3) are rewritten into lua functions.
```
function luf_2(...) return math.random(1000,2000); end
function luf_3(...)
  local col3 = arg[1];
  local col2 = arg[2];
  local col4 = arg[3];
  return SHA1(((col3 + (col2 * col4))));
end
```
And these LUA functions `luf_2() & luf_3()` are applied as UPDATEs to each ROW matching the Where-Clause.

This sounds both stupid and lazy, but now any UPDATE that can be written in LUA, can be done to a Relational Database. Writing a User-Defined-Function, means writing it in LUA, [adding the function](http://code.google.com/p/alchemydatabase/wiki/CommandReference?ts=1317055830&updated=CommandReference#Adding_Lua_functions_at_run-time) to AlchemyDB, and then calling it in an UPDATE. `ComplexUpdatesViaLuaFunctionRewriting` are significantly slower than `SimpleUpdatesViaCFunctions` (mostly due to interpreting the rewritten LUA function, which is done once per command, **not** once per updated row), but these UPDATES are truly flexible.

### HASHABILITY ###
Alchemy DB supports relational tables that can be used as hash-tables. Since Alchemy's rows are stored as streams, empty columns take up minimal memory overhead, and storing sparse rows can be done efficiently.
The command
```
ALTER TABLE tablename ADD HASHABILITY
```
will allow any column specified in the INSERT declaration to be a new column.

For example give the table `t2` w/ columns `(pk,fk1,fk2)`. Issuing the following 2 commands (after `ADD HASHABILITY`)
```
  INSERT INTO t2 (pk, fk1, col1)   VALUES (1, 222, 8888)
  INSERT INTO t2 (pk, fk1, col999) VALUES (1, 222, 99.9999)
```
Would add the columns `col1 & col999` to the table `t2`. The newly added columns values determine their types, in this example `col1 -> BIGINT UNSIGNED` & `col999 -> FLOAT`

This means Alchemy uses SQL to store/retrieve unstructured data (also referred to as Poly Structured Data) and all relational properties/operations can be applied to this data (this is another hybrid property of AlchemyDB)

### Alchemy Cursors ###
Since Alchemy is a **single threaded datastore**, long running queries block other queries during their execution.

For this reason Alchemy has cursors, similar (but simpler) to SQL cursors.<br />
Alchemy Cursors use the construct _"ORDER BY pk/fk LIMIT X OFFSET **variable**"_ to store cursor state in _**variable**_ between cursor calls.

An example of a long-running-update could be
```
UPDATE tbl SET col=X WHERE fk = 1
```
which can be simply transformed into a cursor w/ client side logic (written in pseudocode):
```
ret=1000
while (ret == 1000) {
  ret = UPDATE tbl SET col=X WHERE fk = 1 ORDER BY fk LIMIT 1000 OFFSET CursorUpd8tbl
  sleep(0.01);
}
```
which will update the variable `CursorUpd8tbl` on each call (0->1000->2000->3000->...) and will erase the variable `CursorUpd8tbl` when the final row has been updated.<br />
So calling the above command repeatedly, until the return value is less than _1000_ (which signals the final row matching _fk = 1_ has been updated) will complete the long-running-update described above, but in non-blocking chunks of 1000 rows (w/ 1ms sleeps between chunks).

### LUATRIGGER ###
Luatriggers are lua functions that are run on every SQL INSERT/UPDATE/DELETE.

The syntax for Luatriggers is
```
  CREATE LUATRIGGER triggername ON table INSERT add_trigger(cols,,,,)
  CREATE LUATRIGGER triggername ON table DELETE del_trigger(cols,,,,)
  CREATE LUATRIGGER triggername ON table PREUPDATE preup_trigger(cols,,,,)
  CREATE LUATRIGGER triggername ON table POSTUPDATE postup_trigger(cols,,,,)
```
These 4 cases cover all state changes that take place in an indexes lifetime.

  * On INSERT `add_trigger(cols,,,,)` is called and the values of the inserted columns replace "(cols,,,,)" in the function call
  * On DELETE `del_triggers(cols,,,)` is called
  * On UPDATE the PREUPDATE function is called before any changes are applied (i.e. w/ the old row's values) and then the changes are applied and the POSTUPDATE function is called (w/ the new row's values).

The following example lua function, does:
  * `SELECT COUNT(*) FROM tbl WHERE fk1 = XXX`
  * if this count is greater than 100, `DELETE FROM tbl WHERE fk1 = XXX LIMIT (count-100)` -> this delete caps the table at 100 rows per fk1
  * NOTE: using "table" as the 1st function argument, will pass in the tablename
```
function add_cap(table, fk1)
  cnt = select("count(*)", table, "fk1 = " + fk1); 
  if (cnt > 100) then delete(table, "fk1 = " + fk1 + " LIMIT " + (cnt - 100)); end
  return 1;
end
```
And the following SQL declarations
```
  CREATE TABLE tbl (pk INT, fk1 INT, t TEXT)
  CREATE INDEX      i_cap  ON tbl (fk1)
  CREATE LUATRIGGER lt_cap ON tbl INSERT add_cap(table,fk1)
```
Is a very simple way to implement a per fk number-of-rows cap that runs at almost the same speed as a straight INSERT

### LRUINDEX ###
the LRUINDEX adds a column to a SQL table, called "LRU" (stands for `LeastRecentlyUsed`), which gets updated w/ the current timestamp everytime the commands INSERT,REPLACE,UPDATE,SELECT are called on the row.

The syntax for the LRUINDEX is:
```
 CREATE TABLE tbl (pk INT, c1 INT, c2 INT)
 CREATE LRUINDEX ON tbl
```
After the LRUINDEX command is called, the tables columns are `pk INT, c1 INT, c2 INT, LRU INT)`

Combining a LRUINDEX and a LUATRIGGER is an easy way to add LRU-caching to a relational table.

NOTE: the LRUINDEX has a minimum granularity of 1 second under high load and 5 seconds under low load -> to minimise sparse indexing.

### LFUINDEX ###
the LFUINDEX adds a column to a SQL table, called "LFU" (stands for `LeastFrequentlyUsed`), which is incremented everytime the commands INSERT,REPLACE,UPDATE,SELECT are called on the row.

The syntax for the LFUINDEX is:
```
 CREATE TABLE tbl (pk INT, c1 INT, c2 INT)
 CREATE LFUINDEX ON tbl
```
After the LFUINDEX command is called, the tables columns are `pk INT, c1 INT, c2 INT, LFU INT)`

NOTE: the LFUINDEX stores log2(num\_accesses) -> to minimise sparse indexing.

### Pure Lua Function Index ###
With the depth of Lua's integration, it makes sense to have indexes that are populated (add,delete,update) solely in Lua. In other words it is up to the developer to populate the indexes. With the (via lua functions) populated indexes, it is possible to make SQL queries against them. This provides for highly flexible (albeit development intensive indexes).

To create a Pure Lua Function Index, run the following command:
```
   CREATE INDEX indexname ON tablename "(whereclausetoken())" TYPE constructUserGraphHooks destructUserGraphHooks
```
TYPE can be any column type: (e.g. INT,TEXT,FLOAT,etc..)

`whereclausetoken()` is used in SQL where clauses to directly access the Pure-Lua-Function-Index.

Examples:
```
  SELECT \* FROM tablename WHERE whereclausetoken() = 7
  SELECT \* FROM tablename WHERE whereclausetoken() BETWEEN 2 AND 5
  SELECT \* FROM tablename WHERE whereclausetoken() IN (3,7,9)
```

`constructUserGraphHooks` is a lua function that should place hooks into your Lua environment and "Construct" the Index.

`destructUserGraphHooks` is a lua function responsible for destructing the Index.

The Lua functions that the hooks can use to call into Alchemy's RDBMS (to populate the index) are:
```
  alchemySetIndexByName(), alchemyUpdateIndexByName(), alchemyDeleteIndexByName()
```

Indexes populated in Lua can contain complicated logic, and can be used to index nested data structures.

### PREPARED\_STATEMENTS ###
Prepared statements have 2 main benefits in SQL
  1. reduce network I/O for repetitive queries
  1. boost performance by skipping SQL parsing
```
  PREPARE P_RQ AS SELECT id,name,salary,division FROM employee WHERE division = $1 ORDER BY name
  EXECUTE P_RQ 22
```
> will run:
```
  SELECT id,name,salary,division FROM employee WHERE division = 22 ORDER BY name
```
The intended use-case for Prepared Statements in AlchemyDB is to give that last little bit of performance boost in Alchemy's embedded version.

## REDIS Commands ##
redis supports commands to deal with many data structures, including (strings, sets, lists, sorted-sets, and hash tables)<br />
redis' project home page **[here](http://redis.io/)**<br />
Extensive documentation on redis commands can be found http://redis.io/commands<br />