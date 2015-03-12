# Alchemy Database Benchmark results #

**Hardware**<br />
2 Machines were used as Client and Server<br />
Both: 3.0GHz Phenom X4 CPU w/ 6MB L2 and 8GB RAM @400 MHz (PC3200) w/ 1GbE LAN<br />
(only one core per machine was used)

**Table Size**<br />
All tables have 1 million rows w/ sequential primary keys<br />
The rows returned from queries are random ranges w/ PKs ranging from 0 to 1million.

## Benchmark ONE ##
This benchmarks show how the requests per second the server can handle varies as the number of rows returned from range queries and table-joins varies.<br />

This graph is OLTP DB relevant (2-20 rows returned) as it measures the performance or range queries and joins returning only a small number of rows, which is an acceptable methodology in OLTP<br />
<br />
Y-Axis is requests/second .................. X-axis is number of rows returned<br />
![http://allinram.info/alsosql/PC_RQ-20.png](http://allinram.info/alsosql/PC_RQ-20.png)

The difference between 2-table and 3-table joins becomes less as number-of-rows increases, as can be seen on the [Extended Benchmarks](http://code.google.com/p/redisql/wiki/ExtendedBenchmarks)

<br />
## Benchmark 2 ##
Same as benchmark one but graphed logarithmically and showing how the server performs when it is returning range queries or joins w/ 2 through 50,000 result rows.

This also shows how joins scale as number of tables increase. This is OLAP territory, not OLTP, and the server is not designed for such usage, still, the server holds up well, and such tests are necessary for completeness<br />
<br />
NOTE: both axis are logarithmic and a 10-way table join is introduced `./Benchmark_Range_Query_Lengths.sh 10WAY`<br />
<br />
Y-Axis is requests/second .................. X-axis is number of rows returned<br />
![http://allinram.info/alsosql/PC_ALL_log_w_axisnames.png](http://allinram.info/alsosql/PC_ALL_log_w_axisnames.png)
<br />
<br />
## Benchmark 3 ##
[Lua benchmarks](http://groups.google.com/group/redisql-dev/msg/e24027e82e5c2094) .. informal writeup