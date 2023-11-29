# Jitsitop

A very simple bit of python using the available API (if configured) from jitsi
to display some live statistics on the commandline. It is VERY beta..

## Installation

The program is written in python and requires some libraries to be available.

Required packages:

``apt install python3 python3-requests python3-tzlocal python3-psutil``

## API's

### Jitsi Video Bridge

On the jitsi server, at least the Jitsi Video Bridge API needs to be enabled. To
do this, do the following:

```
cd /etc/jitsi/videobridge
```

```
vi jvb.conf
Make sure the following section is set within videobridge:

    apis {
        rest {
            enabled = true
        }
    }
```

```
vi config
Make sure the following options are enabled on JAVA_SYS_PROPS:

-Dorg.jitsi.videobridge.rest.private.jetty.host=0.0.0.0 -Dorg.jitsi.videobridge.rest.private.jetty.port=8080 -Dorg.jitsi.videobridge.ENABLE_STATISTICS=true -Dorg.jitsi.videobridge.STATISTICS_TRANSPORT=muc,pubsub,callstats.io,colibri"
-Dorg.jitsi.videobridge.rest.private.jetty.host=localhost -Dorg.jitsi
.videobridge.rest.private.jetty.port=8080 -Dorg.jitsi.videobridge.ENABLE_STATISTICS=t
rue -Dorg.jitsi.videobridge.STATISTICS_INTERVAL=1000
```

Restart jitsi video bridge

``systemctl restart jitsi-videobridge2``

Check information is now available

```
curl -s http://127.0.0.1:8080/about/version | jq .
curl -s http://127.0.0.1:8080/colibri/stats | jq .
```

### Prosody

To enable statistics in prosody, it seems the best option is to add the
mod_muc_status module to prosody. This is a modification of the mod_muc_size.
Information on the latest version of this module can be found in a jitsi thread on:

https://community.jitsi.org/t/jitsi-monitoring-mod-room-list-occupant-list/31191

Place the module in the following location:

``/usr/share/jitsi-meet/prosody-plugins/mod_muc_status.lua``

To make this module work, there are a number of extra lua modules required.
This seems to be not very trivial. The following steps seem to work. It does
require to install some development packages on the jitsi server. It is also
possible to build this as a Debian package. There is a recipe available on:

https://github.com/mverboom/build-recipes/tree/master/luajwtjitsi

To do this on a jitsi server, the following steps should work:

```
apt install git lua5.4 luarocks cmake libssl-dev liblua5.4-dev
cd /tmp
git clone https://github.com/evanlabs/luacrypto
cd luacrypto
sed -i '/Lua51 REQUIRED/d' dist.cmake
sed -i -r 's#(lua.h|lauxlib.h)#/usr/include/lua5.4/\1#' src/lcrypto.c
sed -i 's/SHLIB_VERSION_NUMBER/SSLEAY_VERSION_NUMBER/' src/lcrypto.c
luarocks --lua-version 5.4 make
mv /usr/local/lib/lua/5.4/lua/crypto.so /usr/local/lib/lua/5.4/crypto.so
rmdir /usr/local/lib/lua/5.4/lua
luarocks install inspect
luarocks install net-url
luarocks install basexx
luarocks install lua-cjson 2.1.0-1
luarocks install luajwtjitsi
```

After this is installed, the module can be enabled in prosody.

```
vi /etc/prosody/conf.d/localhost.cfg.lua
  -- Section for localhost

  -- This allows clients to connect to localhost. No harm in it.
  VirtualHost "localhost"
          app_id=""
          app_secret=""
          c2s_require_encryption = false

          modules_enabled = {
              "muc_status";
          }
```

Restart prosody

``systemctl restart prosody``

Test if there is output from the rest API, use the domain under which the Jitsi
server is known in the argument.

``curl -s "http://localhost:5280/status?domain=<jitsi domain"``

