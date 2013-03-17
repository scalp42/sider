# sider
=====

A Redis to HTTP protocol proxy.

## Concepts

You have this simple Ruby code:

```ruby
require 'redis'

redis = Redis.new(:port => 9000)
redis.set("key", "value")
```

Sider listens on 9000 and is specified an HTTP endpoint, connected to a remote Redis instance:

	./sider.rb -l 9000 -e "http://proxy.askcerebro.com/sider"

It also speaks Redis unified protocol:


	*3
	$3
	set
	$3
	key
	$5
	value


From there, our initial request is converted to HTTP protocol and sent to our endpoint:

	"GET http://proxy.askcerebro.com/sider/set/key/value HTTP/1.1\n\n"

	URI expected (sider location omitted):

	/set/tata/titi
	
A `HTTP/1.1 200 OK` is returned with the actual response from our remote Redis instance (A first byte "+" to return Redis status):

	HTTP/1.1 200 OK
	Server: Cerebro
	Date: Sun, 17 Mar 2013 22:27:50 GMT
	Content-Type: application/octet-stream
	Content-Length: 3
	Connection: close
	
	+OK

The idea is to support every Redis reply depending on `sider` request:

	In a Status Reply the first byte of the reply is "+"
	In an Error Reply the first byte of the reply is "-"
	In an Integer Reply the first byte of the reply is ":"
	In a Bulk Reply the first byte of the reply is "$"
	In a Multi Bulk Reply the first byte of the reply s "*"
	
We could for example have Nginx take care of the `/sider` location and talk to upstream Redis instances using some lua:

	location /sider {
		default_type text/plain;
		content_by_lua '
		local redis, err = require "resty.redis"
	
		local red = redis:new()
		red:set_timeout(500)
		local ok, err = red:connect("127.0.0.1", 6379)
	
		local res, err = red:set(ngx.var.siderkey, ngx.var.siderva)
	
		if res == ngx.null then
			ngx.status = "-ERR"
			ngx.exit(ngx.status)
		end
	
		ngx.say(res)
	
		local ok, err = red:set_keepalive(5, 10)
		if not ok then
			ngx.say("failed to set keepalive: ", err)
			return
		end';
	}
	
Coming back to our Ruby sample code, we should get a Redis protocol reply:

```ruby
=> #<Redis client v3.0.3 for redis://proxy.askcerebro.com/sider:6379/0>
[14] pry(main)> redis.set("key", "value")
+OK
=> "OK"
```

The idea would allow routing, load balancing, logging and extra X-headers based authorization.