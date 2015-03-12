# Alchemy Database Examples #

# TODO - this is horribly out of date, there are only 2 examples #

**[Jump to examples for Mysql users](http://code.google.com/p/alchemydatabase/wiki/Examples#MYSQL_EXAMPLES)**<br />
**[Jump to examples for Redis users](http://code.google.com/p/alchemydatabase/wiki/Examples#REDIS_EXAMPLES)**<br />
<br />

Alchemy Database provides a single home for your SQL and NOSQL data. Its event driven fast and you can effectively use all of your RAM (even go into swap if you arent scared) to store your data and have it retrieved consistently at RAM speed<br />

<br />
## MYSQL EXAMPLES ##
### MYSQL EXAMPLE ONE: SQL tables at NOSQL speed ###
If your database is bottlenecking and you need NOSQL speed, that is what Alchemy Database was built for<br />
Alchemy Database is a full relational database and is [VERY fast](http://code.google.com/p/alchemydatabase/#FAST_ON_COMMODITY_HARDWARE:)
  1. export table(s) to Alchemy Database (e.g. w/ Ruby function [import\_from\_mysql](https://github.com/JakSprats/redis-rb/blob/master/examples/schemaless.rb#L16-61)),
  1. cutover your application to use an Alchemy Database library (which speak SQL and support RDBMS functionality, meaning you simply switch libraries and maybe rename a few functions, if your current mysql library uses non standard function names)
  1. That's it, you are set
  1. The Alchemy Database server can even be on a different machine if you want to free up resources for your mysql-server
Best to [read](http://code.google.com/p/alchemydatabase/wiki/CommandReference) up on the SQL that Alchemy Database supports and what it does not support, if you are following best OLTP practices, you should be good to go.<br />
Also note that joining a Mysql table w/ a Alchemy Database during runtime is not yet supported
<br /><br />


## REDIS EXAMPLES ##
### REDIS EXAMPLE ONE: create a Mysql Cache for "old" redis data ###
In Memory Databases are fantastic, until they get low on RAM, which is somewhat of an eventuality.<br />
<br />
Its a good practice to archive data you probably will not access again on the frontend from your In Memory Database to a disk based Database (e.g. mysql) and then build a thin cache layer that will retrieve archived data from the disk based Database as needed.<br />
<br />
Building the thin cache layer is surprisingly easy to program. Here is a PHP script that creates a Mysql Cache for the redis ZSET. The object can be found [here](http://github.com/JakSprats/predis/blob/master/examples/ZsetCache.php) _(only 100 lines)_ and an example using tweets can be found [here](http://github.com/JakSprats/predis/blob/master/examples/tweet/tweet_archiver.php).<br />
<br />
With this ZSetCache class, you need only to define your own achiving criterion and scripts, (specific to your data - some peoples data is old after 30 minutes, some after 30 days)<br />
<br />
The approach: cache-old-data-to-disk is a fantastic way to ensure your In Memory Database stays well within its RAM's size and can also be used to avoid spending money on hardware by avoiding the need to scale horizontally.
<br />
<br />