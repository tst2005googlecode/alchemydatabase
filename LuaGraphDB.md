# Alchemy's Graph Database #

**START WARNING->**(this functionality is brand new and only available from the "master" or "release\_0.2+" branch in [github](https://github.com/JakSprats/Alchemy-Database). It has been tested, but since it covers such an impossibly large test space, it has not been tested enough)**<-END WARNING**

**NOTE**: it may be worth reading a broad overview of the extent to which Lua has been integrated into AlchemyDB [here](http://code.google.com/p/alchemydatabase/wiki/LuaIntegration).

A [Graph Database](http://en.wikipedia.org/wiki/Graph_database) uses graph structures with nodes, edges, and properties to represent and store data. AlchemyDB modeled its Graph Database implementation after [Neo4j](http://neo4j.org/), the best of breed in GraphDBs (for an excellent intro to Neo4j, read [this](http://docs.neo4j.org/chunked/stable/graphdb-neo4j.html)).

The purpose of building a GraphDB into AlchemyDB was two fold:
  1. it is nice to be able to do graph traversals on your data, so "embedding" a GraphDB is a wonderful add-on,
  1. it provides an example of how AlchemyDB's mix of Lua & SQL is simple-to-use, highly-flexible, very efficient, & can be used as a data-platform.
A Data Platform is a data-store that is made to be extensible. Adding a GraphDB to AlchemyDB was done using AlchemyDB's existing tools, AlchemyDB needs no additional functionality to be able to build a powerful GraphDB.

## GraphDB Basics ##
A `Node` stores data. A `Node` can have `Relationships` to other Nodes. An example would be `James -> KNOWS -> Jill`. Here, "James" & "Jill" are the `Nodes` and `KNOWS` is the `Relationship`.

`Node`s can also have `Properties`, which are unstructured key-value data attached to the Node. `James` could have the `Property: ['job-title'='manager']`. 'Jill' could have the `Property: ['hobby'='swimming']`.

GraphDB's excel at queries like: _Find ALL the managers who know people whose hobby is swimming_. Doing this type of query in SQL is awkward, unnecessarily complicated, and easy to mess up.

A `Relationship` can also have a `Property` associated to it, for instance if James knows Jill since 2005, then the relationship would have the `Property:` ('since'=2005) and can be visually represented as `Relationship: James -> KNOWS (since:2005) -> Jill`

## Lua implementation ##
AlchemyDB implements a GraphDB using Lua for logic and SQL for indexing. GraphDB functionality is available both at the SQL level and at the Lua function level. GraphDB nodes are embedded into AlchemyDB's [LuaTable](http://code.google.com/p/alchemydatabase/wiki/LuaTableColumnType) column type, which is how they achieve schemaless storage.

The core code for the GraphDB can be found [here](https://github.com/JakSprats/Alchemy-Database/blob/master/redis_unstable/src/core/graph.lua).  Examples, showing depth-first traversals, breadth-first traversals, and shortest-path algorithms can be found [here](https://github.com/JakSprats/Alchemy-Database/blob/master/redis_unstable/src/core/example_user_cities.lua).

## Examples ##
Here is an example, that contains some cities and some fictional distances between them and finds the shortest route from Washington D.C. to San Francisco.

_NOTE: in this particular use-case it made sense to have all APIs in Lua & the Lua routines call SQL. The [LUATRIGGER](http://code.google.com/p/alchemydatabase/wiki/CommandReference#LUATRIGGER) is used to create a `City-Name to PK lookup table` contained (and used) completely in Lua_

Definitions for the functions `addSqlCityRowAndNode(), addCityDistance(), shortestPathByCityName() + add_city(), del_city() ` can be found [here.](https://github.com/JakSprats/Alchemy-Database/blob/master/redis_unstable/src/core/example_user_cities.lua#L64-87). They are all fairly simple routines that make the API simpler for the programmer, and are good examples of how server-side scripting can not only encapsulate funcitonality, but also avoid unnecessary App-Server to DB-Server calls.
```
  INTERPRET LUAFILE "core/graph.lua";
  INTERPRET LUAFILE "core/example_user_cities.lua";
  CREATE TABLE cities (pk INT, lo LUAOBJ, name TEXT)
  CREATE LUATRIGGER lt_cities ON cities INSERT add_city(name, pk)
  CREATE LUATRIGGER lt_cities ON cities DELETE del_city(name)
  LUAFUNC addSqlCityRowAndNode 10 'Washington D.C.' 'DC'
  LUAFUNC addSqlCityRowAndNode 20 'New York City' 'NYC'
  LUAFUNC addSqlCityRowAndNode 30 'San Francisco' 'SF'
  LUAFUNC addSqlCityRowAndNode 40 'Chicago' 'CHI'
  LUAFUNC addSqlCityRowAndNode 50 'Cheyenne' 'CHE'
  LUAFUNC addSqlCityRowAndNode 60 'Atlanta' 'ATL'
  LUAFUNC addSqlCityRowAndNode 70 'Dallas' 'DAL'
  LUAFUNC addSqlCityRowAndNode 80 'Albuquerque' 'ALB'
  LUAFUNC addSqlCityRowAndNode 90 'Lexington' 'LEX'
  LUAFUNC addSqlCityRowAndNode 100 'St. Louis' 'STL'
  LUAFUNC addSqlCityRowAndNode 110 'Kansas City' 'KC'
  LUAFUNC addSqlCityRowAndNode 120 'Denver' 'DEN'
  LUAFUNC addSqlCityRowAndNode 130 'Salt Lake City' 'SLC'

  LUAFUNC addCityDistance 'Washington D.C.' 'Chicago'        200
  LUAFUNC addCityDistance 'Chicago'         'San Francisco'  500 # = 700
  LUAFUNC addCityDistance 'Chicago'         'Cheyenne'       200
  LUAFUNC addCityDistance 'Cheyenne'        'San Francisco'  200 # = 600

  LUAFUNC addCityDistance 'Washington D.C.' 'Atlanta'        100
  LUAFUNC addCityDistance 'Atlanta'         'Dallas'         100
  LUAFUNC addCityDistance 'Dallas'          'San Francisco'  300 # = 500
  LUAFUNC addCityDistance 'Dallas'          'Albuquerque'    100
  LUAFUNC addCityDistance 'Albuquerque'     'San Francisco'  100 # = 400

  LUAFUNC addCityDistance 'Washington D.C.' 'Lexington'       50
  LUAFUNC addCityDistance 'Lexington'       'St. Louis'       50
  LUAFUNC addCityDistance 'St. Louis'       'Kansas City'     50
  LUAFUNC addCityDistance 'Kansas City'     'Denver'          50
  LUAFUNC addCityDistance 'Denver'          'Salt Lake City'  50
  LUAFUNC addCityDistance 'Salt Lake City'  'San Francisco'   50 # = 300

  LUAFUNC shortestPathByCityName 'cities' 'Washington D.C.' 'San Francisco' 'RELATIONSHIP_COST.WEIGHT'             
```


## USERS EXAMPLE ##
This example has _"USERS"_ who _"KNOW"_ each other and who _"VIEWED\_PIC"_ each other.
These _"USERS"_ can also _"HAS\_VISITED"_ different _"CITIES"_.

Once the relationships have been defined, the following two graph traversals are done:
  1. Friends-Of-Friends who have seen my picture (w/ Breadth-First-Search & Depth-First-Search)
  1. Friends-Of-Friends who have seen each other's picture AND seen my picture

Traversal are guided by various functions that the user supplies to the traversal to tell it how to proceed. The traversals below demonstrate the GraphDB's API, which can be grouped into:
  * Breadth & Depth First Search Members:
    1. EDGE\_EVAL: function that decides if the graph traversal should continue by passing back different values [Evaluation.INCLUDE\_AND\_CONTINUE, Evaluation.INCLUDE\_AND\_PRUNE,Evaluation.EXCLUDE\_AND\_CONTINUE, Evaluation.EXCLUDE\_AND\_PRUNE]
    1. EXPANDER: function that decides in which direction the graph traversal should go in: [Direction.OUTGOING,Direction.INCOMING,Direction.BOTH]
    1. ALL\_RELATIONSHIP\_EXPANDER: function usage is similar to EXPANDER, but evaluate across ALL the relationships a node has, whereas EXPANDER evaluates a single relationship.
    1. UNIQUENESS: function that decides whether a node should be traversed more than once, options:[Uniqueness.NODE\_GLOBAL,Uniqueness.NONE,Uniqueness.PATH\_GLOBAL]
    1. MIN\_DEPTH & MAX\_DEPTH: self explanatory, often the depth of a relationship is a determining factor.
  * Shortest Path Members:
    1. NODE\_DELTA: function that accesses two nodes and performs a delta on them which equates to a cost
    1. RELATIONSHIP\_COST: function that uses a Relationship property to represent a cost of traversing this relationship

The functionality of these API arguments follows the principles outlined in this Neo4j [chapter](http://docs.neo4j.org/chunked/stable/tutorial-traversal-concepts.html). They explain it better than I can, I **recommend** reading it. Here is an excerpt of the 3 most important concepts:
```
  Expanders — define what to traverse, typically in terms of relationships direction and type.
  Uniqueness — visit nodes (relationships, paths) only once.
  Evaluator — decide what to return and whether to stop or continue traversal beyond the current position. 
```

At the end of the example, the query for _Users who have visited Washington D.C_ is made. The Indexing of Graph Relationships is accomplished via AlchemyDB's Pure Lua Function Index, which leaves the Index add/delete/updating to be done in Lua (in user defined routines), but allows the Index to be queries in SQL.

```
  CREATE TABLE users (pk INT, hometown INT, lo LUAOBJ)
  CREATE INDEX lf_users ON users (relindx()) LONG constructUserGraphHooks destructUserGraphHooks

  LUAFUNC addSqlUserRowAndNode 1 10 'A'
  LUAFUNC addSqlUserRowAndNode 2 10 'B'
  LUAFUNC addSqlUserRowAndNode 3 20 'C'
  LUAFUNC addSqlUserRowAndNode 4 20 'D'
  LUAFUNC addSqlUserRowAndNode 5 30 'E'
  LUAFUNC addSqlUserRowAndNode 6 30 'F'
  LUAFUNC addSqlUserRowAndNode 7 30 'G'

  LUAFUNC addNodeRelationShipByPK 'users' 1 "KNOWS" 'users' 2
  LUAFUNC addNodeRelationShipByPK 'users' 2 "KNOWS" 'users' 4
  LUAFUNC addNodeRelationShipByPK 'users' 4 "KNOWS" 'users' 7
  LUAFUNC addNodeRelationShipByPK 'users' 1 "KNOWS" 'users' 3
  LUAFUNC addNodeRelationShipByPK 'users' 3 "KNOWS" 'users' 5
  LUAFUNC addNodeRelationShipByPK 'users' 3 "KNOWS" 'users' 6

  LUAFUNC addNodeRelationShipByPK 'users' 1 "VIEWED_PIC" 'users' 2
  LUAFUNC addNodeRelationShipByPK 'users' 2 "VIEWED_PIC" 'users' 4
  LUAFUNC addNodeRelationShipByPK 'users' 4 "VIEWED_PIC" 'users' 1

  LUAFUNC addNodeRelationShipByPK 'users' 6 "VIEWED_PIC" 'users' 1
  LUAFUNC addNodeRelationShipByPK 'users' 5 "VIEWED_PIC" 'users' 7

  # 'BreadthFirstSearch: FOF who have seen my picture'
  LUAFUNC traverseByPK BFS "users" 1 "REPLY.NODENAME_AND_PATH"
                                     "EXPANDER.FOF"
                                     "UNIQUENESS.PATH_GLOBAL"
                                     "EDGE_EVAL.FOF"

  # 'DepthFirstSearch: FOF who have seen my picture'
  LUAFUNC traverseByPK DFS "users" 1 "REPLY.NODENAME_AND_PATH" 
                                     "EXPANDER.FOF"            
                                     "UNIQUENESS.PATH_GLOBAL"  
                                     "EDGE_EVAL.FOF"

  # 'BreadthFirstSearch: FriendsANDPictureSeen-OfFriends  who have seen my picture'
  LUAFUNC traverseByPK BFS "users" 1 "REPLY.NODENAME_AND_PATH"
                                     "ALL_RELATIONSHIP_EXPANDER.FOF"
                                     "UNIQUENESS.PATH_GLOBAL"
                                     "EDGE_EVAL.FOF"

  # WASHINGTON
  LUAFUNC addNodeRelationShipByPK 'users' 1 "HAS_VISITED" 'cities' 10
  LUAFUNC addNodeRelationShipByPK 'users' 4 "HAS_VISITED" 'cities' 10
  LUAFUNC addNodeRelationShipByPK 'users' 7 "HAS_VISITED" 'cities' 10

  # NYC
  LUAFUNC addNodeRelationShipByPK 'users' 1 "HAS_VISITED" 'cities' 20
  LUAFUNC addNodeRelationShipByPK 'users' 2 "HAS_VISITED" 'cities' 20
  LUAFUNC addNodeRelationShipByPK 'users' 3 "HAS_VISITED" 'cities' 20
  # SAN FRAN
  LUAFUNC addNodeRelationShipByPK 'users' 1 "HAS_VISITED" 'cities' 30
  LUAFUNC addNodeRelationShipByPK 'users' 2 "HAS_VISITED" 'cities' 30
  LUAFUNC addNodeRelationShipByPK 'users' 4 "HAS_VISITED" 'cities' 30

  # Users who have VISITed DC (using Pure Lua Function)
  SELECT lo.node.__name FROM users WHERE relindx() = 10

  # Unvisit DC for one user
  LUAFUNC deleteNodeRelationShipByPK 'users' 7 "HAS_VISITED" 'cities' 10

  # Users who have VISITed DC
  SELECT lo.node.__name FROM users WHERE relindx() = 10

  SELECT hometown, get_fof(lo) FROM users WHERE relindx() = 30
```

## Usage of Pure Lua Function Indexes ##
AlchemyDB has a feature called [Pure Lua Function Indexes](http://code.google.com/p/alchemydatabase/wiki/CommandReference#Pure_Lua_Function_Index).

In the above example, the "Pure Lua Function Index" is used to index the very nested relationship of a `USER -> HAS_VISITED -> CITY`, which exists in the GraphDB, and must be made to exist in the relational database.

Once indexed, looking up users who have visited NYC is an efficient index lookup.

Using the example of indexing the relationship: `users who have visited NYC` the developer should add new primary keys to this index every time a user creates a relationship `VISITS->NYC`. Deleting this relationship doesnt really make much sense, but if a user could somehow UN-visit NYC, then his primary key should be deleted from this index. This is an oversimplification: every time a user visits a city (assuming users are in the SQL table: `users` and cities are in the SQL table: `cities`) the call: `alchemySetIndexByName('visits_index', cities.pk, user.pk)` should be made. This allows later calls to get all users.PKs who have visited city w/ cities.PK=X possible (e.g. all users who have visited NYC).

The intended use case of the "Pure Lua Function Index" is to index data that is not relational, but is better indexed by a function.

# MORE TO COME #
This is all brand new, so it is a work in progress, stay tuned :)