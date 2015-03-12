**DRAFT** This is very much a work in progress

# AlchemyDB's Lua Integration #

[Lua](http://www.lua.org/about.html) has been integrated into AlchemyDB in basically every way that made sense for an important use case. The following is a list of how and where and maybe even why it was integrated

## INTERPET LUAFILE ##
> [->](http://code.google.com/p/alchemydatabase/wiki/CommandReference?ts=1330364119&updated=CommandReference#Adding_Lua_functions_at_run-time) Load a file full of lua into AlchemyDB's lua universe, and it will be interpreted to byte-code (or if LuaJIT is installed, parts of it _may_ be turned into machine-code).

## INTERPRET LUA ##
> [->](http://code.google.com/p/alchemydatabase/wiki/CommandReference?ts=1330364119&updated=CommandReference#Adding_Lua_functions_at_run-time) Load a line of Lua into AlchemyDB's lua universe, and it will be interpreted to byte-code (or if LuaJIT is installed, parts of it _may_ be turned into machine-code).

## LUAFUNC functionname args ##
> [->](http://code.google.com/p/alchemydatabase/wiki/CommandReference?ts=1330364119&updated=CommandReference#calling_Lua_functions_via_LUAFUNC) Once you have interpreted some Lua, you need to be able to call it from your client and have its return value returned to your client

## LUA CRON ##
> AlchemyDB has a config file setting to run a lua function every 100ms. Usage can be handy, but is discouraged, because the function BLOCKS.

## LUATRIGGER ##
> [->](http://code.google.com/p/alchemydatabase/wiki/CommandReference?ts=1330364119&updated=CommandReference#LUATRIGGER) Similar to a RDBMS trigger, Alchemy will call a lua function when you INSERT,DELETE,UPDATE a SQL row and pass this function the contents of the row.

## Pure Lua Function Index ##
> [->](http://code.google.com/p/alchemydatabase/wiki/CommandReference?ts=1330364119&updated=CommandReference#Pure_Lua_Function_Index) A SQL index can be declared, that is accessible in the where-clause of a SQL command, but who is populated solely via user defined lua functions.

## LUATABLE column type ##
> [->](http://code.google.com/p/alchemydatabase/wiki/LuaObject) A column type capable of storing anything a Lua-Table can store. This opens up unstructured data and nested documents on the per-row level

## SQL INSERT ##
> Inserting a row into a table w/ a `LUATABLE` column type can insert the column via [JSON, a lua function, a lua eval()]

## SQL SELECT ##
> Lua functions are callable in a SELECT call, in the following sections [column-list, where-clause-predicates, order-by-columns]. The last two are also possible for SQL DELETEs and UPDATEs

## SQL UPDATE ##
> a SQL update can call lua functions in the SET clause

## AppStack ##
> [->](http://code.google.com/p/alchemydatabase/wiki/AppStack) AlchemyDB has an integrated HTTP server, that can call lua functions that return HTML pages

## Lua Override Response Functions ##
> [->](http://code.google.com/p/alchemydatabase/wiki/RestAPI?ts=1330441177&updated=RestAPI#Lua_Override_Response_Functions) Lua functions can be defined to reformat the responses of SQL SELECT calls, enabling a developer to come up w/ a customer line protocol (via REST).


### NOTES ###
The only place interpretation of Lua is possible is via the 2 "INTERPRET" commands. On INSERTs, if the `LUATABLE` value is JSON it is decoded into a Lua table. On SELECTs, if the where-clause-predicate lua function is a complex function call (i.e. nested) then the entire function call will be passed to lua and eval()ed {which is interpretation}, but the function will be saved, and will later be callable w/o interpretation by pattern matching subsequent calls.