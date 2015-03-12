# Lua Table Column Type #

**START WARNING->**(this functionality is brand new and only available from the "master" or "release\_0.2+" branch in [github](https://github.com/JakSprats/Alchemy-Database). It has been tested, but since it covers such an impossibly large test space, it has not been tested enough)**<-END WARNING**

**NOTE**: it may be worth reading a broad overview of the extent to which Lua has been integrated into AlchemyDB [here](http://code.google.com/p/alchemydatabase/wiki/LuaIntegration).

Alchemy Database has a column type called "LUATABLE" which contains a [Lua table](http://lua-users.org/wiki/TablesTutorial). [Lua](http://www.lua.org/about.html) is a turing complete  embedded language, and Lua tables can be efficiently used to store almost any data structure (even ones w/ functions, aka objects :)

Alchemy Database combines the relational logic of a RDBMS w/ the power of Lua tables and functions and the resulting data-store resembles both an Object-Database and a Document-Store.

To create a LUATABLEect Column, declare the column as type LUATABLE:
```
   CREATE TABLE users (userid INT, zipcode INT, info LUATABLE)
```

Inserting data into a LUATABLEect column is done in JSON. The JSON is decoded and stored as a Lua Table, (subsequent lua function calls access the data natively)
```
  INSERT INTO users "(userid, zipcode, info)" VALUES "(1,44555,{'fname':'BILL','lname':'DOE','age':32,'groupid':999,'height':70,'weight':180})"
  INSERT INTO users "(userid, zipcode, info)" VALUES "(2,44555,{'fname':'Jane','lname':'Smith','age':22,'groupid':888,'coupon':'XYZ123'})"
```
Note: in the 2nd INSERT `info.height & info.weight` are not defined, which is ok, as the Lua-table has no defined structure.

Elements within a LUATABLEect can be indexed, w/ Dot-Notation-Indexing (NOTE: the final argument specifies a data type for the index)
```
  CREATE INDEX i_users_i_a ON users (info.age) LONG
  CREATE INDEX i_users_i_g ON users (info.groupid) LONG
```

The following queries would use the previous indexes and return ['BILL' & 'Jane'] respectively
```
  SELECT info.name FROM users WHERE info.age = 32
  SELECT info.name FROM users WHERE info.groupid = 888
```
_In the author's humble opinion, Dot-Notation-Indexes should be viewed as slightly inferior when compared to normal indexes. It is recommended practice, that if you use a Dot-Notation-Index heavily, to break it out into a normal SQL index, which perform slightly better as the overhead of passing data between C & Lua is avoided and Dot-Notation-Indexes currently have limitations (e.g. no JOINs on DNIs)._

This is the basic formula for how Lua was mixed into SQL in AlchemyDB. AlchemyDB also allows direct Lua access (via the `LUAFUNC` call) and Lua functions can call the function `alchemy(,,,,)` which makes requests back into the database ... so the integration of Lua is cyclical on all levels.

This integration is best described w/ some examples, as it is a syntactic mixing of SQL, Lua Tables, and Lua functions.

The first step is to define a Lua function in AlchemyDB: a string of Lua can be passed to Alchemy, which is then interpreted (and remains defined in the Lua universe as byte-code (or even machine code w/ LuaJIT) for subsequent calls)
```
   INTERPRET LUA "function has_coupon(info, code) if (info.coupon ~= nil and info.coupon == code) then return 1; else return 0; end; end"
```
_Note: this function sees if a particular LUATABLEect column has the field "coupon\_code" defined, and then checks to see if it matches the coupon\_code from the function call._

The following query now becomes possible
```
  CREATE INDEX i_u_zip ON users (zipcode)
  SELECT info.name FROM users WHERE zipcode = 44555 AND has_coupon(info, 'XYZ123')
```
The function "has\_coupon()" will be run (at Lua-Stack-Call + byte-code speed) on every row w/ zipcode 44555, and ONLY those rows having "info.coupon\_code='XYZ123'" will be returned.

Besides adding Lua functionality in the where-clause, it is also possible to call Lua from:
  * SELECT Column Clause
```
    INTERPRET LUA "function format_name(t) return string.sub(t.fname, 1, 1) .. '. ' .. t.lname; end"
    SELECT format_name(info) FROM users WHERE zipcode = 44555 AND has_coupon(info, 'XYZ123')
```
  * UPDATE Set Clause
```
   INTERPRET LUA "function generate_coupon() return 'XYZ' .. math.random(1,999); end"
   UPDATE users SET info.coupon = generate_coupon() WHERE userid = 1
```
  * Complex UPDATEs via SELECTs: The function performs the update, the SELECT simply locates the relevant row.
```
    INTERPRET LUA "weight_gain_per_year={}; weight_gain_per_year[20]=1; weight_gain_per_year[25]=2; weight_gain_per_year[30]=3; weight_gain_per_year[35]=4; weight_gain_per_year[40]=5; weight_gain_per_year[45]=5; weight_gain_per_year[50]=4; weight_gain_per_year[55]=3; weight_gain_per_year[60]=2;"
    INTERPRET LUA "function update_weight(info) for k, v in pairs(weight_gain_per_year) do if (k > info.age) then info.weight = info.weight + v; return true; end; end; return false; end"
    SELECT update_weight(info) FROM users WHERE userid = 1
```
  * ORDER BY Clause
```
    SELECT format_name(info) FROM users WHERE zipcode = 44555 ORDER BY string.sub(info.lname,1,3)
```

It is **VERY** important to note that this implementation makes it impossible to have interpretable Lua functions at runtime, all functions must be defined beforehand (which converts them to byte-code or machine-code) and subsequent calls are done in C via Lua's stack, which is very efficient. Embedding a language into SQL that needs interpretation at runtime is **much** slower.

## EXAMPLES ##
The examples above are pretty lame, but hopefully it is clear that mixing Lua into SQL in such a way provides a highly flexible DML (implemented highly efficiently using Lua's virtual stack & byte-code function calls) that logically fits within the relational model. The goal here is to be able to do the exact operations you want to do on your data as efficiently as possible and not have to tweak & bend them to fit into SQL.

A better example of how powerful these constructs are is explained in [AlchemyDB's GraphDB](http://code.google.com/p/alchemydatabase/wiki/LuaGraphDB) implementation

## Lua Internals ##
### Direct Lua Access to Lua-Tables ###
AlchemyDB stores each LUATABLEect column in Lua's universe. Each row's LUATABLEect Column is stored as a Lua Table that is addressable as `ASQL[tblname][columnname][pk]`. If we took the above example, where the table "users" has the LUATABLEect "info", and userid=1, the LUATABLEect is stored in a table at `ASQL["users"]["info"][1]`. Manipulating the LUATABLE table's values directly in Lua is OK (e.g. setting `ASQL["users"]["info"][1].newfield=7` is OK)  and will not corrupt the database, but do not modify any data in the `ASQL[tblname][columnname][pk]` tree as this tree represents the raw database (e.g. do NOT do: `ASQL["users"]["info"]=nil`, as this deletes ALL `users.info` LUATABLE's).

### Direct Lua Access to Dot-Notation-Indexed Elements ###
AlchemyDB did some Lua voodoo (combo of index & newindex meta-method trickery) to ensure that if a Dot-Notation-Indexed value is changed directly in Lua, the change will be reflected in Alchemy's indexes. This can be reworded as: changing the value of a Dot-Notation-Indexed field directly in a Lua function will cause an update-index-trigger in Alchemy. An example of this would be (setting `ASQL["users"]["info"][1].age=55` would cause an update-trigger to the "i\_users\_i\_a" index, and a subsequent SQL query: `SELECT * FROM users WHERE info.age=55` would return the row w/ userid=1)

This ensures data consistency between Lua & the RDBMS and means the developer does not have to worry about where (in SQL or Lua) he changes data, his changes will be reflected in both Lua & via SQL.

### Persistence ###
Any data residing in the `ASQL[]` tree will be persisted to disk and there is also a special Lua Table called `UserData[]` that will be persisted to Disk (e.g. setting `UserData.sometable = {key='mkey'; value=33;};` will persist if you shutdown and start AlchemyDB). This is useful for users who use Lua itself to store data, (which is encouraged, _if the use-case fits_).

### ACID ###
`LUATABLE` column types have the same ACID property as any column in AlchemyDB. If an Alchemy write command involving many rows (e.g. Range-Update) encounters an error halfway, NONE of the `LUATABLE` operations will be applied, the same goes if an INSERT encounters a Unique-Constraint-Violation.

## Alternate Insert Syntaxes ##
`LUATABLE` columns can be initialised via Lua Function calls
```
  INSERT INTO doc VALUES (1,111, nested_lua(10,9.99,'text'))
```
The final column in the table "doc" will be the return value of the already interpreted Lua function: `nested_lua()`.

`LUATABLE` columns can also be initilaised by evaluating Lua statements. In the next example, the Lua is so complex, it makes sense to simply eval() it in Lua (instead of trying to parse it in C, and then pass over the parse tree to Lua via Lua's stack)
```
  INSERT INTO doc VALUES (2,111, nested_lua(45*77+100-math.sqrt(99)))
```

As w/ JSON `LUATABLE` initialisations, all data is ultimately stored in Lua Tables.

## Nested Updates ##
Nested UPDATEs and SELECTs are supported, here is an example:
```
  INTERPRET LUA "function cubed(n) return n * n * n; end" 
  INTERPRET LUA "function nested_table(num) local t = {}; t['DEEP']={}; t['DEEP']['a']={} t['DEEP']['a']['b']={}; t['DEEP']['a']['b']['c']=cubed(num); return t; end"
  CREATE TABLE users (userid INT, zipcode INT, info LUATABLE)
  INSERT INTO users VALUES (1, 3333, {'nest':{'x':{'y':{'z':5}}}});
  SELECT * FROM users WHERE userid=1
     RETURNS: 1,3333,{'nest':{'x':{'y':{'z':5}}}}
  UPDATE users SET info.nest.x.y.TABLE = nested_table(info.nest.x.y.z) WHERE userid=1
  SELECT * FROM users WHERE userid=1
     RETURNS: 1,3333,{'nest':{'x':{'y':{'TABLE':{'DEEP':{'a':{'b':{'c':125}}}},'z':5}}}}
  UPDATE users SET info.nest.x.y = {'K':{'L':777}} WHERE userid=1
  SELECT * FROM users WHERE userid=1
     RETURNS: 1,3333,{'nest':{'x':{'y':{'K':{'L':777}}}}}
  UPDATE users SET info.nest.x.y.z = 100 WHERE userid=1
  SELECT * FROM users WHERE userid=1
     RETURNS: 1,3333,{'nest':{'x':{'y':{'K':{'L':777},'z':100}}}}
  UPDATE users SET info.nest.x.y.z = info.nest.x.y.z + info.nest.x.y.z * 100 WHERE userid=1
  SELECT * FROM users WHERE userid=1
     RETURNS: 1,3333,'{'nest':{'x':{'y':{'K':{'L':777},'z':10100}}}}
```