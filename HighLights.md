# DRAFT #

## Advanced Features ##
  * [LUA command](http://code.google.com/p/alchemydatabase/wiki/CommandReference#LUA_Command) and [DEEP lua embedding](http://code.google.com/p/alchemydatabase/wiki/LuaIntegration) provides a turing complete language to manipulate data (far more flexible than RDBMS's procedural languages)
  * [CREATE INDEX via BATCH](http://code.google.com/p/alchemydatabase/wiki/CommandReference#CREATE_INDEX_via_BATCH) allow the "CREATE INDEX" call to be called in batches. The Index is not active until it is completely built. This allows the non-blocking building of HUGE indexes.
  * [ORDER BY INDEX](http://code.google.com/p/alchemydatabase/wiki/CommandReference#ORDER_BY_INDEX) store values sorted to another column in the table, instead of to the default PK. Very useful for dashboard updates as ORDER BY sorting is done (w/ zero overhead) at INSERT time. If you want a users last 10 tweets, this index returns them w/o buffering or sorting.
  * [Complex Updates via Lua](http://code.google.com/p/alchemydatabase/wiki/CommandReference?ts=1317055830&updated=CommandReference#Complex_UPDATEs_via_lua-rewriting). Complex updates are rewritten transparently & on-the-fly to LUA functions and then applied to all rows matching the where-cluase. This means UPDATE operations are limited in complexity by what you can program in LUA.
  * [HASHABILITY](http://code.google.com/p/alchemydatabase/wiki/CommandReference#HASHABILITY) - relational tables can be declared as having "HASHABILITY" which means any column declared in the INSERT column declaration that is not present in the table will be added to the table transparently. This effectively turns a relational table into a hash-table, preserving all relational properties, and (using Alchemy's rows-as-streams and sparse-rows) has minimal memory overhead. This means Alchemy uses SQL to store/retrieve unstructured data (also referred to as Poly Structured Data)
  * [Alchemy Cursors](http://code.google.com/p/alchemydatabase/wiki/CommandReference#Alchemy_Cursors) allow a long running operation (e.g. a full table scan/update/etc...) to be broken up into batch process. This negates Isolation in ACID, but is often acceptable in given use cases.
  * [LUATRIGGER](http://code.google.com/p/alchemydatabase/wiki/CommandReference#LUATRIGGER) calls a LUA function containing the INSERTED/DELETED/UPDATED rows contents per INSERT/UPDATE/DELETE
  * [LRUINDEX](http://code.google.com/p/alchemydatabase/wiki/CommandReference#LRUINDEX) adds an indexed column (called "LRU") to a table which gets updated w/ the current timestamp every time the row is accessed. (perfect for getting rid of "old" data -> use a LUATRIGGER to do it automatically).
  * [LFUINDEX](http://code.google.com/p/alchemydatabase/wiki/CommandReference#LFUINDEX) adds an indexed column (called "LFU") to a table which gets incremented every time the row is accessed. (perfect for getting rid of "unpopular" data).
  * 128 bit INTEGER support. These 16 BYTE integers are best utilised to store 2 LONGs or 4 INTs and then indexed and used instead of a multi-column ORDER BY statement. Using U128s combined w/  [ORDER BY INDEX](http://code.google.com/p/alchemydatabase/wiki/CommandReference#ORDER_BY_INDEX) , ORDER BY statements w/ up to 8 columns can be done withOUT sorting. These indexed U128s use 68-96 bytes per row, so you can put more than one on a table, and do away w/ ORDER BY sorting (avoiding a large/repetitive performance hit). U128 columns can also place unique indexes on UUIDs w/ minimal overhead.
  * "SELECT indexname.pos" - for use-cases where the position of the index determines a ranking. Used on U128 indexes provides highly optimised multiple variable leaderboards/scoreboards.
  * `LuaCronJob`- a lua script can be run every second w/in the server. Use of this functionality comes w/ a big fat warning sticker: **the cron job is BLOCKING**
  * SUBPIPE (**TODO: document**) is an external process that subscribes to a Alchemy channel and forwards any messages it receives to another Alchemy instance. Combined w/ LUA & Cursors, long blocking operations can be broken up into batches that are performed iteratively and echoed from the Alchemy Server to the SUBPIPE and back, creating a feedback loop.
  * Advanced Messaging (**TODO: document**)
    * `MESSAGE` Command + Lua `Redisify()`
    * Slave Knowledge: Lua `IsConnectedToMaster()`
    * Lua commands: `RemoteMessage()`, `RemotePipe()`
    * Lua Pub/Sub file-descriptor access `SubscribeFD()`, `UnsubscribeFD()`, `GetFDForChannel()`, `CloseFD()`,

## Memory optimisations ##
  * Sparse-Rows: tables w/ 1000s of columns, store sparsely populated rows in an optimised format, using a serialised hash table in the row's stream to store column offsets. Sparse-Rows can be Orders-Of-Magnitude smaller than full rows (NOTE: useful for unstructured data: just keep adding columns, and Sparse-Rows limit memory usage)
  * TEXT columns are lzf compressed if over 20 characters, under 20 characters a homemade compression algorithm (dubbed: sixbit) sometimes yields modest reductions.
  * INT + LONG fields are bit packed and algorithms work on the compressed values
  * INT, LONG, & U128 Indexes are stored inside their respective Btrees (this includes Multiple Column Indexes and Unique Indexes) - also 2 column [INT|LONG|U128] tables have been similarly optimised.
  * When possible UPDATEs are done in place (if an update does not change the size of a row, there is no allocation/deallocation needed)
  * Rows are streams, ALTER TABLE ADD COLUMN is instantaneous, empty columns take up almost no space (1-4 bytes, 0 when the row itself has been converted to a hash-table)
  * Btree dynamic resizing: Btrees (indexes also) start tiny and then dynamically resize once they have reached a threshold size (helps w/ small tables and indexes w/ few entries)
  * jemalloc is compiled in, which has shown far less memory fragmentation (than glibc malloc, tcmalloc) and even gives freed memory back to the OS
  * COMING SOON: slab allocator for all btree nodes

## Miscellaneous ##
  * Unlimited number of tables, indexes, & columns (allows for NOSQL schema-less like data, add in [HASHABILITY](http://code.google.com/p/alchemydatabase/wiki/CommandReference#HASHABILITY) and using lua functions can yield a  very fast single-node in-memory map-reduce building block)
  * sqlappendonly writes ALL SQL CRUD to a file in MYSQL format (this file can be piped directly to a mysql-server, add a LUATRIGGER that evicts "cold" data via a LRUINDEX, and Alchemy becomes a RDBMS-Cache w/ relational logic).
  * SCION iterator - _SELECT ORDER BY fk/pk LIMIT X OFFSET Y_ has 2 specialised iterators, which finds ANY OFFSET in O(log(N)) time. This allows [Alchemy Cursors](http://code.google.com/p/alchemydatabase/wiki/CommandReference#Alchemy_Cursors) to run at about the same speed as large blocking operations (SELECT,UPDATE,DELETE), but Alchemy Cursors do not block, they can be tuned to have about a 20% performance hit for other queries when the system is under full load.
  * Embeddable - Alchemy Database can be [embedded](https://github.com/JakSprats/Alchemy-Database/blob/master/redis_unstable/src/embed2.c) into a C program (or other languages thru C), BUT it is single threaded, you need to write your own thread-multiplexing layer or run from a single thread. The embedded version is still being polished, but using [PREPARED STATEMENTS](http://code.google.com/p/alchemydatabase/wiki/CommandReference?ts=1324530844&updated=CommandReference#PREPARED_STATEMENTS) it can already boast speeds of over 400K TPS for simple SQL transactions and redis SET/GET run at 1M & 1.5M TPS respectively.
  * Multiple Output formats (normal-sql-rows, redis-output, stream-output _coming soon_)
  * PURGE command (**TODO: document**) guarantees a master has successfully flushed his replication stream to his slave(s) and then shuts down. Can be used to move Alchemy Instances (slave-promotion) or to do Mitosis on a particular node (i.e. split into 2 to scale better) and the slave is guaranteed up-to-date after the PURGE command.

## Architecture ##
  * Event-Driven, Non-blocking I/O (includes frontend, replication, publishing, periodic snapshots, aof-writing, etc....)
  * Detailed table memory usage (per index stats, data stats, btree-overhead stats, per-row-@insert-size stats)
  * [Webserver](http://code.google.com/p/alchemydatabase/wiki/AppStack) - Distributed framework (**TODO: document**)