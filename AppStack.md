# The Alchemy AppStack #

**WARNING: Alchemy's AppStack is experimental, it has been tested, but it needs more testing**

Alchemy's AppStack is a combined `DynamicWebserver+Datastore` stack. Dynamic HTML webpages are create via [LUA](http://www.lua.org/about.html) functions that access data via an embedded Alchemy Database.

Turning on `webservermode` in [Alchemy's conf file](https://github.com/JakSprats/Alchemy-Database/blob/master/redis_unstable/redis.conf), will activate HTTP serving on Alchemy's listening port. A layer to protect database access from the outside world is provided by only allowing 2 paths to access data:
  * whitelisted-IPs: App-servers whose IP addresses have been whitelisted in `redis.conf` can directly access Alchemy Database
  * whitelisted lua functions: other IP addresses (i.e. external/not-whitelisted) can only call whitelisted lua functions (which then access the DB), ALL other calls (e.g. GET,SET,INSERT,SELECT) are prohibited.

Alchemy's AppStack supports both HTTP and redis' protocol, the latter can be accessed in Javascript via a [Flash object](https://github.com/claus/as3redis) and has improved performance (up to 50% faster for tiny requests).

The following is a diagram of how Alchemy can be used in a AppStack architecture:

![http://allinram.info/alsosql/AlchemyAppStack.png](http://allinram.info/alsosql/AlchemyAppStack.png)

Simple requests (via HTTP, Javascript, or Flash) can be processed by  whitelisted lua functions and will be served w/ minimal latency at very high concurrency.

To configure this setup, the following settings must be activated in the file [redis.conf](https://github.com/JakSprats/Alchemy-Database/blob/master/redis_unstable/redis.conf):
```
  webservermode yes
  include_lua functions.lua
  webserver_index_function index_page
  webserver_whitelist_address 192.168.1.1
  webserver_whitelist_netmask 255.255.255.0
```

The variable `webservermode` turns the AppStack on.

The variable `include_lua` points to a file that contains whitelisted lua modules (which MUST start w/ "WL`_`" and are callable directly from the frontend) and normal non-whitelisted (helper) functions (can NOT start w/ "WL`_`").

Example functions.lua
```
function htmlify(t)
  return '<html>' .. h .. '</html>';
end
function WL_index_page() 
  return htmlify('HI I AM THE INDEX PAGE');
end
function WL_welcome(user) 
  return htmlify'<h2>HI: ' + user + '</h2><img src ="/STATIC/logo.png"/>');
end
```

The HTTP call `GET /welcome/user333 HTTP/1.0` would be translated to the whitelisted lua function `WL_welcome('user333');` and the HTML `'<html><h2>HI: user333</h2><img src ="/STATIC/logo.png"/></html>` would be returned as a HTTP 200.

The function `htmlify()` is a helper function and **NOT** callable from the frontend. The HTTP call `GET /htmlify/HIYA HTTP/1.0` would be translated to `WL_htmlify('HIYA');` and would result in a 404 as the function is not defined.

The variable `webserver_index_function` is the function that is called, when "GET / HTTP/1.0" is called, or it is the function that generates the index.html page (in the above whitelist.lua, `WL_index_page()` would be called (NOTE: the "WL`_`" gets preprended).

The variables `webserver_whitelist_address` and `webserver_whitelist_netmask` define the range of ip-addresses that will be allowed DIRECT access to Alchemy's Database (i.e. the whitelisted ips). These whitelisted ips are for App-Servers.

Together whitelisted lua functions and whitelisted IPs, provide a very crude security framework to protect data access from the outside world.

A blog explaining the why, when, and how of the AppStack can be found
[here](http://jaksprats.wordpress.com/2011/07/28/the-alchemy-app-stack/)

The Actionscript and Jsquery libraries can be found here: _TODO: LINK_

### STATIC FILES ###
A backdoor for static files has been built in. Any key starting w/ "STATIC/" can be served directly from Alchemy.

So the HTTP request `GET /STATIC/a.gif HTTP/1.0\r\n\r\n` is allowed from the outside world and maps to the Alchemy command `GET STATIC/a.gif` and can be populated by calling `SET STATIC/a.gif [BLOB]`.

### HTTP Interface ###
HTTP Request Headers are made available in Lua via the associative array `HTTP_HEADER[]`. If you want the value of the HTTP Request Header 'Host', it is accessible in Lua as `HTTP_HEADER['Host']`

HTTP Request Cookies are made available in Lua via the associative array `COOKIE[]`. If you want the value of the HTTP Request Cookie 'mycookie', it is accessible in Lua as `COOKIE['mycookie']`

Setting HTTP Response Headers can be done in Lua via the Lua function `SetHttpResponseHeader`. Setting the HTTP Response Header 'Expires', would be done by calling `SetHttpResponseHeader('Expires', 'Wed, 09 Jun 2021 10:18:14 GMT');`. Setting the HTTP Response Cookie 'mycookie' can be done in Lua by calling `SetHttpResponseHeader('Set-Cookie', 'mycookie=value_for_mycookie; Expires=Wed, 09 Jun 2021 10:18:14 GMT; path=/;');`

HTTP Redirects can be called from Lua via the function `SetHttpRedirect(redirect_url);`. To redirect to the page '/index\_page', call: `SetHttpRedirect('/index_page'); return;` (NOTE: return immediately w/ no return value).

Etags are supported via the `SetHttp304()` call

### DEMO ###
A port of the demo app [retwis](http://retwis.antirez.com/) (written in php) to Alchemy's AppStack Lua can be found [here](https://github.com/JakSprats/Alchemy-Database/blob/master/redis_unstable/src/docroot/retwis_whitelist.lua)