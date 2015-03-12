# AlchemyDB is now Aerospike - visit us at [www.aerospike.com](http://www.aerospike.com) #

# Alchemy Database: A Hybrid RDBMS/NOSQL-Datastore #

Alchemy Database is a low-latency high-TPS NewSQL RDBMS embedded in the NOSQL datastore [redis](http://redis.io/). Extensive [datastore-side-scripting](http://jaksprats.wordpress.com/2010/11/15/datastore-side-scripting-can-prevent-bottlenecks/) is provided via deeply embedded [Lua](http://www.lua.org/about.html). Unstructured data, can also be stored, as there are no limits on #tables, #indexes, #columns, and [sparsely populated rows](http://code.google.com/p/alchemydatabase/wiki/HighLights?ts=1317936394&updated=HighLights#Memory_optimisations) use minimal memory.

AlchemyDB believes OLTP traffic's needs are best served by extending SQL and has recently added the following experimental functionalities:
  * [LuaTable](http://code.google.com/p/alchemydatabase/wiki/LuaTableColumnType) - A column type that is a [Lua Table](http://lua-users.org/wiki/TablesTutorial), that single handedly adds both Document-Store & Object-DB functionality by mixing Lua into SQL.
  * [GraphDB](http://code.google.com/p/alchemydatabase/wiki/LuaGraphDB) - so brand new it is not even fully documented :) A GraphDB was created on top of AlchemyDB using SQL for indexes and Lua for graph-traversal logic. AlchemyDB is a customizable data platform.
  * [AppStack](http://code.google.com/p/alchemydatabase/wiki/AppStack) - AlchemyDB uses a REST API and already had Lua embedded, creating a dynamic HTTP server, serving Lua webpages was a logical step, and it is the fastest dynamic webserver I have ever benchmarked (probably because it can only make internal AlchemyDB calls, i.e. NO  backend calls :)

Alchemy Database is optimised for top notch memory efficiency and top notch TPS for OLTP requests:
  * Speed is achieved by being an event driven network server that stores 100% of data in RAM, achieving disk persistence by using a spare cpu-core to periodically log data changes (i.e. no threads, no locks, no undo-logs, no disk-seeks, serving data over a network at RAM speed)
  * Storage data structures w/ very low memory overhead and data compression, via algorithms w/ insignificant performance hits, greatly increase the amount of data you can fit in RAM
  * Optimising to the [SQL statements](http://code.google.com/p/alchemydatabase/wiki/CommandReference#Supported_SQL) most commonly used in OLTP workloads yields a lightweight RDBMS designed for [low latency at high concurrency](http://jaksprats.wordpress.com/2010/09/27/next-generation-web-server-pole-low-latency-at-high-concurrency/) (i.e. world class speed/thruput).

### RAM is CHEAP these days ###
RAM is now affordable enough to be able to store ENTIRE OLTP Databases in a single machine's RAM (e.g. [Wikipedia's English DB](http://en.wikipedia.org/wiki/Wikipedia:Database_download) is 30GB and a
[Dell T610](http://configure.us.dell.com/dellstore/config.aspx?oc=becwek1&c=us&l=en&s=bsd&cs=04&model_id=poweredge-t610) w/ 32GB RAM costs $2100). Data can be asynchronously replicated over the wire (providing high availability) and written to disk via snapshots and appending log files (providing durability) and data I/O is done at RAM speed.

### FAST ON COMMODITY HARDWARE: ###
Client/Server using 1GbE LAN to/from a **single** core running at 3.0GHz, RAM PC3200 (400MHz)
  * 95K INSERT/sec, 95K SELECT/sec, 90K UPDATE/sec, 100K DELETE/sec
  * Range Queries returning 10 rows: 40K/sec
  * 2 Table Joins returning 10 rows: 20K/sec
  * Lua script performing read and write: 85K/sec

### MEMORY EFFICIENT: ###
  1. Each row has very little overhead when stored (20-30bytes) and Insert speed does not significantly degrade as more indices are added
    * Simple row (PK+TEXT->16 bytes), 1GB stores 40 million rows, insert speed: 70K/sec
    * Complex row (10 Indices+TEXT->48 bytes), 1GB stores 9 million rows, insert speed: 40K/s
    * TEXT fields are compressed. If a 100 character column compresses down to 80 bytes, the row can be stored w/ ZERO storage overhead (e.g. 1million rows of 100 chars will take up 100MB)
  1. Sparse-Rows: tables w/ 1000s of columns, that are sparsely populated, use a serialised hash table in the row's stream to store column offsets. Sparse-Rows can be Orders-Of-Magnitude smaller than full rows ... [more info](http://code.google.com/p/alchemydatabase/wiki/HighLights)

### EASY TO USE: ###
  * Its [SQL](http://code.google.com/p/alchemydatabase/wiki/CommandReference#Supported_SQL) ... you already know it :)
  * [Redis commands](http://redis.io/commands) are VERY easy to learn :)
  * Need more, [Lua is embedded](http://code.google.com/p/alchemydatabase/wiki/CommandReference#LUA_Command), opening up ... EVERYTHING
  * Alchemy's [SQL extensions](http://code.google.com/p/alchemydatabase/wiki/CommandReference#Beyond_SQL) enhance SQL and fix some of its common shortcomings

### INSTALL ###
**BUILD:**
  1. Download the code from git via `"git clone git://github.com/JakSprats/Alchemy-Database.git"`
  1. type `"cd Alchemy-Database"`
  1. type `"make"`
**RUN:**
> type `"cd ./redis/src; ./alchemy-server"`
**CONFIG:**
> Config is done in the config file called "./redis/redis.conf" and is documented at redis' [website](http://redis.io/documentation) as well as in the [file](https://github.com/JakSprats/Alchemy-Database/blob/master/redis_unstable/redis.conf)

## CLIENTS: ##
Starting with Release 0.2, AlchemyDB has recently switched to a REST API, documentation can be found [here](http://code.google.com/p/alchemydatabase/wiki/RestAPI)

## LICENSE: ##
Alchemy Database is both AGPL licensed & commercially licensed. A GPL'ed Community Edition is in the works.