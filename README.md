PIVX Nginx Proxy
====================

This repo allows exposing a Stash JSON-RPC server to remote hosts using nginx as a reverse proxy.
For security purpose, a Stash client exposes the JSON-RPC service only to localhost interface. In some cases, it can be interesting to expose JSON-RPC to remote hosts but with limited access.

This solution is to use a LUA script, used with nginx-lua-module to control access to the upstream stash client. You can blacklist or whitelist certain JSON-RPC methods (for example, authorize only JSON-RPC requests to `sendrawtransaction` to allow remote hosts to send transactions to the client, without allowing them to read states in the chain).

Why nginx?
----------

nginx has some interesting characteristics as a reverse proxy on top of Stash JSON-RPC service:

* Can expose the JSON-RPC service through a TLS channel
* Can load balance between multiple JSON-RPC backends
* Can plug standard authentication on top of the JSON-RPC interface (basic authentication, TLS mutual authentication, ...)


How does it work
----------------

The script `stash-jsonrpc-access.lua` must be put in a `location` of your nginx, to be used from `access_by_lua_file` directive.
In order to have a fully functional nginx proxy, you need:

* Stash [core wallet](https://github.com/stashpayio/stash/releases)
* [nginx](https://nginx.org/en/linux_packages.html#stable) equipped with:
  * lua-nginx-module, see the setup [here](https://github.com/openresty/lua-nginx-module#installation).
  * cjson lua package installed, see [here](https://www.kyne.com.au/~mark/software/lua-cjson.php)
* [Alternatively] [openresty](https://openresty.org/en/installation.html) default nginx binaries already come with the required plugins (see below "Easy Install")

You have to set one of `jsonrpc_whitelist` or `jsonrpc_blacklist` nginx variable to restrict the list of JSON-RPC methods available for this location, separated by commas:

```
location / {
  set $jsonrpc_whitelist 'decodemasternodebroadcast,relaymasternodebroadcast,sendrawtransaction';
  access_by_lua_file 'stash-jsonrpc-access.lua';
  proxy_pass http://localhost:9998;
}
```

With the configuration above, only `decodemasternodebroadcast`, `relaymasternodebroadcast`, `sendrawtransaction` calls will be fowarded to JSON-RPC interface at `http://localhost:9998`, other requests will be rejected with HTTP 403 status.

You can find a full `nginx.conf` example file in the repo.

<br>

Easy Install
------------
The following guide is intended for Ubuntu 16.04 servers and can be easily adapted for other linux versions.

<br>

#### Requirements
Make sure to have a Stash full node running with the following `stash.conf` configuration (replace `myRpcUser` and `myRpcPass` with random credentials):
```
txindex=1
server=1
rpcuser=myRpcUser
rpcpassword=myRpcPass
daemon=1
logtimestamps=1
maxconnections=256
```

Make sure to have webdomain (for the purpose of this guide `myexampledomain.com` will be used) and an A RECORD pointing to the IP of your VPS.

<br>

#### Packages
Install the following package (for the `add-apt-repository` command):
```
sudo apt-get update && sudo apt-get -y install software-properties-common
```

Add **openresty** GPG key:
```
wget -qO - https://openresty.org/package/pubkey.gpg | sudo apt-key add -
```

Add the official openresty APT repository and install the package (then disable it for now)
```
sudo add-apt-repository -y "deb http://openresty.org/package/ubuntu $(lsb_release -sc) main"
```
```
sudo apt-get update && sudo apt-get -y install openresty
sudo systemctl disable openresty
```

Add the official APT **certbot** repository and install the package
```
sudo add-apt-repository ppa:certbot/certbot
```
```
sudo apt-get update && sudo apt-get -y install certbot
```

<br>

#### Configuration
- [1. Clone this repository](#1)
- [2. Create the required folders and copy the files](#2)
- [3. Obtain SSL certificates for your domain and copy them](#3)
- [4. Create and save the credentials to query this server](#4)
- [5. Encode base64 your RPC credentials](#5)
- [6. Edit nginx.conf](#6)

<a name="1"></a>1. Clone this repository
```
git clone https://github.com/stashpayio/stash-nginx-proxy.git
```
<br>

<a name="2"></a>2. Create the required folders and copy the files (nginx configuration file and lua script)
```
mkdir nginx-proxy && cd nginx-proxy && mkdir conf && mkdir logs
```
```
cp ../stash-nginx-proxy/nginx.conf conf/
cp ../stash-nginx-proxy/stash-jsonrpc-access.lua conf/
```
<br>

<a name="3"></a>3. Allow port 80 on `ufw` (if enabled) and obtain a certificate for your domain `myexampledomain.com` with certbot (after that, port 80 can be disabled again)
```
sudo certbot certonly --standalone -d myexampledomain.com
```

Copy the certificates (replacing `myexampledomain.com` with your actual domain)
```
sudo cp /etc/letsencrypt/live/myexampledomain.com/fullchain.pem conf/fullchain.pem
sudo cp /etc/letsencrypt/live/myexampledomain.com/privkey.pem conf/privkey.pem
```
<br>

<a name="4"></a>4. Create a `.htpasswd` to store the credentials that will be used to query this server (in this example the username is `myProxyUser`, replace it with one of your choice and type a password when prompted):
```
sudo htpasswd -c conf/.htpasswd myProxyUser
```
<br>

<a name="5"></a>5. Encode Base64 the rpc credentials that you set in your `stash.conf`.<br>
Run this command (replacing `myRpcUser` and `myRpcPass`) and save the output string which will be copied inside `nginx.conf` file:
```
echo -n "myRpcUser:myRpcPass" | base64
```
<br>

<a name="6"></a>6. Open Nginx configuration file. <br>
```
sudo nano conf/nginx.conf
```

- Replace `server_name  _;` with your domain, e.g.
```
server_name  myexampledomain.com;
```

- Replace `xxxxxxxxx==` with the base64 encoded string you obtained at point (5) here
```
proxy_set_header Authorization "Basic xxxxxxxxx==";
```

Close and give permissions to the user which will run the server (e.g. `www-data`)
```
sudo chown www-data: -R .
```

<br>

#### Usage
This setup uses port 8080. Make sure to allow such port on `ufw` or change it inside `nginx.conf`.<br>
From the `nginx-proxy` folder, with the linux-user allowed (e.g. `www-data`) start the server with:
```
openresty -p `pwd`/ -c conf/nginx.conf
```

- To follow the logs:
```
tail -f logs/access.log
tail -f logs/error.log
```

- To reload the server (after editing `nginx.conf`):
```
openresty -p `pwd`/ -c conf/nginx.conf -s reload
```

- To stop the server:
```
openresty -p `pwd`/ -c conf/nginx.conf -s stop
```

<br>

#### Testing
You can test your server, for example, asking for the sync status:
```
curl --user myProxyUser --data-binary '{"jsonrpc": "1.0", "id": "nginxtest", "method": "mnsync", "params": ["status"] }' -H 'content-type: text/plain;' https://myexampledomain.com:8080/
```

and receiving an output like this:
```
{
  "result":{
    "IsBlockchainSynced":true,
    "lastMasternodeList":1539912115,
    "lastMasternodeWinner":1539914694,
    "lastBudgetItem":1539910982,
    "lastFailure":0,
    "nCountFailures":0,
    "sumMasternodeList":0,
    "sumMasternodeWinner":25779,
    "sumBudgetItemProp":19707,
    "sumBudgetItemFin":11759,
    "countMasternodeList":0,
    "countMasternodeWinner":4,
    "countBudgetItemProp":5,
    "countBudgetItemFin":5,
    "RequestedMasternodeAssets":999,
    "RequestedMasternodeAttempt":0
  },
  "error":null,
  "id":"nginxtest"
}
```
