# Ledge

A [Lua](http://www.lua.org) module for [OpenResty](http://openresty.org), providing scriptable HTTP cache (edge) functionality for Nginx.

It utilises [Redis](http://redis.io) as a storage backend, and depends on the [lua-resty-redis](https://github.com/agentzh/lua-resty-redis) module bundled with OpenResty as well as [lua-resty-rack](https://github.com/pintsized/lua-resty-rack), maintained separately.

## Status

This library is considered experimental and under active development, functionality may change without notice.

Please feel free to raise issues at [https://github.com/pintsized/ledge/issues](https://github.com/pintsized/ledge/issues).

## Installation

Download and install:

* [Redis](http://redis.io/download) >= 2.4.14
* [OpenResty](http://openresty.org/) >= 1.2.1.9

Review the [lua-nginx-module](http://wiki.nginx.org/HttpLuaModule) documentation on how to run Lua code in Nginx.

Clone this repo and [lua-resty-rack](https://github.com/pintsized/lua-resty-rack) into a path defined by [lua_package_path](http://wiki.nginx.org/HttpLuaModule#lua_package_path) in `nginx.conf`.

### Basic usage

Ledge can be used to cache any `location` blocks in Nginx, the most typical case being one which uses the [proxy module](http://wiki.nginx.org/HttpProxyModule), allowing you to cache upstream resources.

```nginx
server {
	listen 80;
	server_name example.com;
	
	location /__ledge_origin {
		internal;
		rewrite ^/__ledge_origin(.*)$ $1 break;
		proxy_set_header X-Real-IP  $remote_addr;
		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
		proxy_set_header Host $host;
		proxy_read_timeout 30s;
		
		# Keep the origin Date header for more accurate Age calculation.
		proxy_pass_header Date;
		
		# http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html#sec14.38
		# If the response is being forwarded through a proxy, the proxy application MUST NOT
		# modify the Server response-header.
		proxy_pass_header Server;
		
		proxy_pass $scheme://YOUR.UPSTREAM.IP.ADDRESS:80;
	}
}
```

To place Ledge caching in front of everything on this server, first initialise `resty.rack` and `ledge.ledge` during `init_by_lua`, and then start Ledge with the `content_by_lua` directive.

```lua
http {
    init_by_lua '
        rack = require "resty.rack"
        ledge = require "ledge.ledge"
    ';

    server {
        listen 80;
        server_name example.com;

        location / {
            content_by_lua '
                rack.use(ledge)
                rack.run()
            ';
        }

        location /__ledge_origin {
            ...
        }
    }
}
```

## Functions

You can configure Ledge behaviours and extend the functionality by calling API functions **before** running `rack.run()`.

### ledge.set(param, value)

**Syntax:** `ledge.set(param, value)`

Sets a configuration option.

```lua
ledge.set("origin_location", "/__my_origin")
```

### ledge.get(param)

**Syntax:** `local value = ledge.get(param)`

Gets a configuration option.


### ledge.bind(event_name, callback)

**Syntax:** `ledge.bind(event, function(req, res) end)`

Binds a user defined function to an event. See below for details of event types.

The `req` and `res` parameters are documented in [lua-resty-rack](https://github.com/pintsized/lua-resty-rack). Ledge adds some additional convenience methods.

* `req.accepts_cache()`
* `res.cacheable()`
* `res.ttl()`

## Events

Ledge provides a set of events which are broadcast at the various stages of cacheing / proxying. The req/res environment is passed through functions bound to these events, providing the opportunity to manipulate the request or response as needed. For example:

```lua
ledge.bind("response_ready", function(req, res)
	res.header['X-Homer'] = "Doh!"
end)
```

The events currently available are:

#### cache_accessed

Broadcast when an item was found in cache and loaded into `res`.

#### origin_required

Broadcast when Ledge is about to proxy to the origin.

#### origin_fetched

Broadcast when the response was successfully fetched from the origin, but before it was saved to cache (and before __before_save__!). This is useful when the response must be modified to alter its cacheability. For example:

```lua
ledge.bind("origin_fetched", function(req, res)
	local ttl = 3600
	res.header["Cache-Control"] = "max-age="..ttl..", public"
	res.header["Pragma"] = nil
	res.header["Expires"] = ngx.http_time(ngx.time() + ttl)
end)
```

This blindly decides that a non-cacheable response can be cached. Probably only useful when origin servers aren't cooperating.

#### before_save

Broadcast when about to save a cacheable response.

#### response_ready

Ledge is finished and about to return. Last chance to jump in before rack sends the response.

## Configuration options

### origin_location

*Default:* `/__ledge_origin`

### origin_mode

*Default:* `ORIGIN_MODE_NORMAL`

One of:

* `ORIGIN_MODE_NORMAL`
* `ORIGIN_MODE_AVOID`
* `ORIGIN_MODE_BYPASS`

`ORIGIN_MODE_NORMAL` proxies to the origin as expected. `ORIGIN_MODE_AVOID` will disregard cache headers and expiry to try and use the cache items wherever possible, avoiding the origin. This is similar to "offline_mode" in Squid. `ORIGIN_MODE_BYPASS` assumes the origin is down (for maintenance or otherwise), using cache where possible and exiting with `503 Service Unavailable` otherwise.

### redis_host

*Default:* `127.0.0.1`

### redis_port

*Default:* `6379`

### redis_socket

*Default:* `nil`

`connect()` will use TCP by default, unless `redis_socket` is defined.

### redis_database

*Default:* `0`

### redis_timeout

*Default:* `nil`

ngx_lua defaults to *60s*, overridable per worker process by using the `lua_socket_read_timeout` directive. Only set this if you want fine grained control over Redis timeouts (rather than all cosocket connections).

### redis_keepalive_timeout

*Default:* `nil`

ngx_lua defaults to *60s*, overridable per worker process by using the `lua_socket_keepalive_timeout` directive.

### redis_keepalive_pool_size

*Default:* `nil`

ngx_lua defaults to *30*, overridable per worker process by using the `lua_socket_pool_size` directive.

### cache_key_spec

Overrides the cache key spec. This allows you to abstract certain items for great hit rates (at the expense of collisons), for example.

The default spec is:

```lua
{
    ngx.var.request_method,
    ngx.var.scheme,
    ngx.var.host,
    ngx.var.uri,
    ngx.var.args
}
```

Which will generate cache keys in Redis such as:

```
ledge:cache_obj:HEAD:http:example.com:/about
ledge:cache_obj:HEAD:http:example.com:/about:p=2&q=foo
```

If you're doing SSL termination at Nginx and your origin pages look the same for HTTPS and HTTP traffic, you could simply provide a cache key spec omitting `ngx.car.scheme`, to avoid splitting the cache.

Another case might be to use a hash algorithm for the args, if you're worried about cache keys getting too long (not a problem for Redis, but potentially for network and storage).

```lua
ledge.set("cache_key_spec", {
    ngx.var.request_method,
    --ngx.var.scheme,
    ngx.var.host,
    ngx.var.uri,
    ngx.md5(ngx.var.args)
})
```

### keep_cache_for

*Default:* `30 days`

Specifies how long cache items are retained regardless of their TTL. You can use the [volatile-lru](http://antirez.com/post/redis-as-LRU-cache.html) Redis configuration to evict the least recently used cache items when under memory pressure. Therefore this setting is really about serving stale content with `ORIGIN_MODE_AVOID` or `ORIGIN_MODE_BYPASS` set.

## Logging / Debugging

For cacheable responses, Ledge will add headers indicating the cache status.

### X-Cache

This header follows the convention set by other HTTP cache servers. It indicates simply `HIT` or `MISS` and the host name in question, preserving upstream values when more than one cache server is in play. For example:

* `X-Cache: HIT from ledge.tld` A cache hit, with no (known) cache layer upstream.
* `X-Cache: HIT from ledge.tld, HIT from proxy.upstream.tld` A cache hit, also hit upstream.
* `X-Cache: MISS from ledge.tld, HIT from proxy.upstream.tld` A cache miss, but hit upstream.
* `X-Cache: MISS from ledge.tld, MISS from proxy.upstream.tld` Regenerated at the origin.

### X-Ledge-Cache

This custom header provides a little more information about the cache status.

* `X-Cache-Ledge: REVALIDATED from ledge.tld` The cache was revalidated. The item is fresh (and may have been fetched).
* `X-Cache-Ledge: IGNORED from ledge.tld` The cache was ignored (no-cache / max-age=0 / must-revalidate). The item is fresh (and was fetched).
* `X-Cache-Ledge: HOT from ledge.tld` The ttl is greater than 0.
* `X-Cache-Ledge: WARM from ledge:tld` ttl + max_stale is greater than 0.
* `X-Cache-Ledge: COLD from ledge:tld` ttl + max_stale is less than 0, but we have an old cache item.
* `X-Cache-Ledge: SUBZERO from ledge:tld` We have nothing for this key.
* `X-Cache-Ledge: PRIVATE from ledge:tld` The response is not cacheable. You will never see this in the headers, but it will be available in the log (see below).

### Log variables

Variables for Nginx to include in the access log must be instantiated in the config file first. If the following variables are defined, Ledge will update them with useful values.

#### `$ledge_version`

In the format `ledge/REV`.

#### `$ledge_cache`

As the X-Cache header above, but also present (MISS) for non cacheable responses.

#### `$ledge_cache_state`

As the `X-Cache-Ledge` header above, with the addition of `PRIVATE` for non-cacheable responses.

#### `$ledge_origin_action`

* `NONE` We served from cache.
* `FETCHED` We went to the origin.
* `COLLAPSED` We received a shared (newly cached) response after waiting for another connection to fetch. __(not yet implemented)__

#### Example

```nginx
http {
    log_format ledge '$remote_addr - $remote_user [$time_local]  '
        '"$request" $status $bytes_sent '"$http_referer" "$http_user_agent" '
        '"$ledge_version" "$ledge_cache" "$ledge_cache_state" "$ledge_origin_action"';

    access_log logs/access.log ledge;

    server {
        set $ledge_version 0;
        set $ledge_cache 0;
        set $ledge_cache_state 0;
        set $ledge_origin_action 0;

        location / {
            ...
        }
    }
}
```

## Known limitations

The following major items are currently not implemented, but on the short term TODO list.

* No support for validation (If-Modified-Since, If-None-Match etc).
* No support for logic around the Vary header.


## Planned features

Once the core functionality is more stable, there are plans for:

* A plugin mechanism for modules to hook into events.
* Some bundled plugins to solve common problems:
 * ESI parser.
 * CSS Combining.
 * Stats gathering and reporting.
 * ...
* Stale while revalidate.
* Collapse forwarding.
* ...
 
## Author

James Hurst <james@pintsized.co.uk>

## Licence

This module is licensed under the 2-clause BSD license.

Copyright (c) 2012, James Hurst <james@pintsized.co.uk>

All rights reserved.

Redistribution and use in source and binary forms, with or without modification, are permitted provided that the following conditions are met:

* Redistributions of source code must retain the above copyright notice, this list of conditions and the following disclaimer.

* Redistributions in binary form must reproduce the above copyright notice, this list of conditions and the following disclaimer in the documentation and/or other materials provided with the distribution.

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
