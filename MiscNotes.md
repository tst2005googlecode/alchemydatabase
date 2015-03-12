<br />
## Additional features: ##
  1. Bindings available in all supported client side languages to import/export tables to/from Mysql (import for speed, export for data-mining)
  1. Datastore-side-scripting is possible via embedded Lua, including a Lua function (written in C) called "client" which calls the Alchemy Database server internally from Lua scripts. Complex data structures and server side logic and functions can be implemented quickly and easily.
  1. trivial to setup master-slave replication
  1. Persistence: from time to time data is saved to disk asynchronously (semi persistent mode) or alternatively every change is written into an append only file (fully persistent mode). Alchemy Database is able to rebuild the append only file in the background when it gets too big.
  1. An experimental [web\_server](http://code.google.com/p/alchemydatabase/wiki/AppStack) mode speaks HTTP and calls whitelisted Lua functions to server web pages
<br />