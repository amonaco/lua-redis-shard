# lua-redis-shard

A simple library based on crc32 for supporting hasring-distribution on Redis.

This ensures that given string "key" which could be hostname or customer account-code, the resulting destination will always be the same one. This allows some basic tenant distribution in multi-tenant systems.

This code was created pre- redis cluster times. But it can be used also with memcached databases pretty much anything that has an IP address and TCP port.

This library can be used in standalone Lua or OpenResty. For the latter, see the current sample `nginx.conf`:


```
worker_processes 1;
error_log logs/error.log;

events {
    worker_connections 1024;
}

http {
    server {
        location / {
            content_by_lua_block {

		-- say we want to do hashring on the following hosts
                redis_shards = {
                  "127.0.0.1:6375",
                  "127.0.0.1:6378",
                  "127.0.0.1:6377",
                  "127.0.0.1:6379",
                  "127.0.0.1:6376"
                }

                local redis = require "resty.redis"
                local hash_ring = require "hash_ring"
                local hr = hash_ring:create(redis_shards)

                -- sting "foo" will be used as a distribution key
                local shard = hr:get_node("foo")

		-- now with resulting host:port combination ...
                local redis_client = redis:new()
                local host, port = shard:match("([^:]+):([^:]+)")

		-- ... we connect to the specified redis instance
                local ok, err = redis_client:connect(host, port)
                if not ok then
                  ngx.log(ngx.STDERR, "failed to connect to redis because: ", err)
                  return
                end

		-- we can still do other redis stuff
                red:set_keepalive(2000, 50)
                redis_client:set("foo", "bar")
            }
        }
    }
}
```

Standalone usage:

```
shards = {
  "127.0.0.1:6375",
  "127.0.0.1:6378",
  "127.0.0.1:6377",
  "127.0.0.1:6379",
  "127.0.0.1:6376"
}

local hash_ring = require "hash_ring"
local hr = hash_ring:create(shards)

print(hr:get_node("foo"))
```
