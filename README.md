PIVX Nginx Proxy
====================

This repo allows exposing a PIVX JSON-RPC server to remote hosts using nginx as a reverse proxy.
For security purpose, a PIVX client exposes the JSON-RPC service only to localhost interface. In some cases, it can be interesting to expose JSON-RPC to remote hosts but with limited access.

This solution is to use a LUA script, used with nginx-lua-module to control access to the upstream pivx client. You can blacklist or whitelist certain JSON-RPC methods (for example, authorize only JSON-RPC requests to `sendrawtransaction` to allow remote hosts to send transactions to the client, without allowing them to read states in the chain).

Why nginx?
----------

nginx has some interesting characteristics as a reverse proxy on top of PIVX JSON-RPC service:

* Can expose the JSON-RPC service through a TLS channel
* Can load balance between multiple JSON-RPC backends
* Can plug standard authentication on top of the JSON-RPC interface (basic authentication, TLS mutual authentication, ...)


Installation
------------

The script `pivx-jsonrpc-access.lua` must be put in your nginx, to be used from `access_by_lua_file` directive. 
In order to have a fully functional nginx proxy, you have to install:

* lua-nginx-module, see the setup [here](https://github.com/openresty/lua-nginx-module#installation).
* cjson lua package installed, see [here](https://www.kyne.com.au/~mark/software/lua-cjson.php)


Usage
-----

The lua script must be used in a `location` nginx using the `access_by_lua_file` directive.

You have to set one of `jsonrpc_whitelist` or `jsonrpc_blacklist` nginx variable to restrict the list of JSON-RPC methods available for this location, separated by commas:


```
location / {
  set $jsonrpc_whitelist 'decodemasternodebroadcast,relaymasternodebroadcast,sendrawtransaction';
  access_by_lua_file 'pivx-jsonrpc-access.lua';
  proxy_pass http://localhost:51473;
}
```

With the configuration above, only `decodemasternodebroadcast`, `relaymasternodebroadcast`, `sendrawtransaction` calls will be fowarded to JSON-RPC interface at `http://localhost:51473`, other requests will be rejected with HTTP 403 status.

You can find a full `nginx.conf` example file in the repo.


Easy Setup
----------

...