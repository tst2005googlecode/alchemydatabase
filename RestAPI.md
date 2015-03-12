**DRAFT**: written in great haste

# REST API #

With release 0.2 AlchemyDB has switched to using a REST API.

The mechanism to turn a AlchemyDB/redis request into a HTTP request is pretty simple. The request "GET XYZ" is written in redis' protocol as `*2\r\n$3\r\nGET\r\n$1\r\nXYZ\r\n`. In the Rest API it is simply `http://server:ip/GET/XYZ`

By default responses are returned in redis' protocol, (e.g. the response of "I AM XYZ" would be `$8\r\nI AM XYZ\r\n`). SQL SELECT responses can be written in custom protocols using AlchemyDB's [Lua Override Response Functions](http://code.google.com/p/alchemydatabase/wiki/RestAPI?ts=1330441177&updated=RestAPI#Lua_Override_Response_Functions).

To activate the REST API, use the following command
```
  ./alchemy-cli CONFIG SET rest_api_mode yes
```

Here are some example REST API Queries, for queries uploading data (e.g. INSERT) the post-body can be used as the final argument to the command
```
  curl -D - 127.0.0.1:6379"/DROP/TABLE/rest"
  curl -D - -d "(pk INT, fk INT, col TEXT)" 127.0.0.1:6379"/CREATE/TABLE/rest/"
  curl -D - 127.0.0.1:6379"/CREATE/INDEX/i_rest/ON/rest/(fk)"
  curl -D - -d "(,2,'TOPDAWG')/RETURN SIZE" "127.0.0.1:6379/INSERT/INTO/rest/VALUES/"
  curl -D - -d "(,2,'TEST')/RETURN SIZE" "127.0.0.1:6379/INSERT/INTO/rest/VALUES/"
  curl -D - 127.0.0.1:6379"/SCAN/*/FROM/rest"
```
_NOTE: the usage of double quotes above is to avoid bash expansion_

Here is a list of AlchemyDB's SQL commands as REST API requests
  * `http://server:port/CREATE/TABLE/users/(pk INT, fk INT, name TEXT, age INT)`
  * `http://server:port/CREATE/INDEX/indexname/ON/tbl/(column)`
  * `http://server:port/INSERT/INTO/users/VALUES/(1,1,'bill',22)`
  * `http://server:port/INSERT/INTO/users/VALUES/(2,1,'jane',20)`
  * `http://server:port/SELECT/*/FROM/users/WHERE/pk =2`
  * `http://server:port/UPDATE/users/SET/age=age + 1/WHERE/pk=700`
  * `http://server:port/DELETE/FROM/users/WHERE/fk IN (3,4,5)`

Alchemy's protocol is described [here](http://code.google.com/p/alchemydatabase/wiki/Protocol), the mapping of arguments in this protocol to number of '/' in the REST API is 1-to-1

## Lua Override Response Functions ##
SQL SELECT response's default to being returned in redis' protocol but can be overridden by user defined functions.

To activate and define AlchemyDB's Lua Override Response Functions, use the following 3 commands
```
  ./alchemy-cli CONFIG SET lua_output_start output_start_http;
  ./alchemy-cli CONFIG SET lua_output_cnames output_cnames_http;
  ./alchemy-cli CONFIG SET lua_output_row output_row_http;
  ./alchemy-cli CONFIG SET OUTPUTMODE LUA
```
The function defined by `lua_output_start` is called before the response row's are output, w/ a single argument representing the number of rows in the response.

The function defines by `lua_output_cnames` handles the outputting of the column names, each argument is a column-name, and is called after `lua_output_start` and before the first call to `lua_output_row`

The function defined by `lua_output_row` is called for each ROW returned by the SELECT statement and each argument to the function is the value of a column (that was specified in the SELECT column list)

**NOTE**: Both `lua_output_cnames` and `lua_output_row` will later recieve a single argument, which is a lua-table w/ all the column-names and columns-values in it. This is a more efficient means of passing data between C and Lua.

Example Lua Override Response Functions can be found [here](https://github.com/JakSprats/Alchemy-Database/blob/master/redis_unstable/src/core/internal.lua#L241-261) and look like this:
```
RowCounter = 0;
function output_start_http(card)
    RowCounter = 0;
    return "nrows=" .. card .. ';\r\n';
end
local function output_delim_http(...)
   local printResult = '';
   for i,v in ipairs(arg) do
     if (string.len(printResult) > 0) then printResult = printResult .. ","; end
     printResult = printResult .. tostring(v);
   end
   return printResult .. ';\r\n';
end
function output_cnames_http(...)
   return 'ColumnNames=' .. output_delim_http(...);
end
function output_row_http(...)
  RowCounter = RowCounter + 1;
  return 'row[' .. RowCounter .. ']=' .. output_delim_http(...);
end
```

The Lua Override Response Functions allow you to write your own response protocols for SQL SELECT responses.