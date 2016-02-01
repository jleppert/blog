---
layout: post
title: "Reverse Proxy Web Sockets with Nginx and Socket.IO"
description: "If you’re using Socket.io and want to reverse proxy your web socket connections, you’ll quickly find it’s somewhat difficult. Since web sockets are done over HTTP 1.1 connections (where the handshake and upgrade are completed), your backend needs to support HTTP 1.1, and from what I have researched, they break the HTTP 1.0 spec"
tags: [web-sockets, proxy, nginx, socket-io]
oldurl: /reverse-proxy-web-sockets
---

If you’re using Socket.io and want to reverse proxy your web socket connections, you’ll quickly find it’s somewhat difficult. Since web sockets are done over HTTP 1.1 connections (where the handshake and upgrade are completed), your backend needs to support HTTP 1.1, and from what I have researched, they break the HTTP 1.0 spec (see this discussion at stackoverflow).

Some people have been able to get HAProxy to work for effective proxying of websocket connections, but I couldn’t get this to work reliably when I tried the latest version and TCP mode.

If you’re using nginx, you won’t be able to proxy web socket connections using the standard nginx proxy_pass directives. Fortunately, Weibin Yao has developed a tcp proxy module for nginx that allows you to reverse proxy general tcp connections, especially well suited for websockets.

Let’s get a simple web socket proxy up and running. Download nginx sources and tcp_proxy module:

## Compile Nginx with tcp_proxy module

{% highlight bash %}
export NGINX_VERSION=1.0.4
curl -O http://nginx.org/download/nginx-$NGINX_VERSION.tar.gz
git clone https://github.com/yaoweibin/nginx_tcp_proxy_module.git
tar -xvzf nginx-$NGINX_VERSION.tar.gz
cd nginx-$NGINX_VERSION
patch -p1 < ../nginx_tcp_proxy_module/tcp.patch
./configure --add-module=../nginx_tcp_proxy_module/
sudo make && make install
{% endhighlight %}

## Proxy Configuration

Create a simple vhost like the following (note tcp must be defined at the server directive level):

{% highlight js %}
tream websockets {
  ## node processes
  server 127.0.0.1:8001;
  server 127.0.0.1:8002;
  server 127.0.0.1:8003;
  server 127.0.0.1:8004;

  check interval=3000 rise=2 fall=5 timeout=1000;
}

  server {
    listen 127.0.0.1:80;
    server_name _;

    tcp_nodelay on;
    proxy_pass websockets;
  }
}

http {
  ## status check page for websockets
  server {
    listen 9000;

    location /websocket_status {
      check_status;
    }
  }
}
{% endhighlight %}

You’ll notice there is an additional http section, with a check_status directive. The tcp_proxy module provides a simple and convenient status check page which you can use to see if your backend node processes are up:

<figure>
	<img src="/images/down.png"/>
</figure>

## Create a Simple Websocket Server

{% highlight js %}
var http = require('http'), io = require('socket.io');

var server = http.createServer(function(req, res){
  res.writeHead(200, {'Content-Type': 'text/html'});
  res.end('Hello world');
});
server.listen(process.argv[2]);

// socket.io
var socket = io.listen(server);
socket.on('connection', function(client){
  // new client is here!
  console.log('client has connected');
  client.on('message', function(){  })
});
{% endhighlight %}

Start four node processes, each listening on different ports:

{% highlight bash %}
node ./websocket-server.js 8001 &
node ./websocket-server.js 8002 &
node ./websocket-server.js 8003 &
node ./websocket-server.js 8004 &
{% endhighlight %}

You should now check your status page to verify all backends are up and running:

<figure>
	<img src="/images/up.png"/>
</figure>

You can also go to our proxy via standard http (web browser) and correctly see “Hello World” to verify we’re hitting one of our node backends.

## Test Websocket Proxying

Now lets create a simple web socket client to test if we can actually create a websocket connection over the proxy:

{% highlight html %}
<html>
  <head>
  <title>Websockets Proxy Test</title>
  <script src="sio.js"></script>
  <script>
  var socket = new io.Socket('ws://localhost');
    socket.connect();
    socket.on('connect', function() {
    console.log('connected!');
  });
  </script>
</head>
  <body>
    <h1>Websockets Proxy Test</h1>
  </body>
</html>
{% endhighlight %}

For simplicity, I used a node static page server to serve this test page from the same node process and attached my Socket.IO instance to it:

{% highlight js %}
var io = require('./Socket.IO-node');

var http = require("http"),
    url = require("url"),
    path = require("path"),
    fs = require("fs"),
    port = process.argv[2] || 8888;

var server = http.createServer(function(request, response) {
  var uri = url.parse(request.url).pathname, filename = path.join(process.cwd(), uri);

  path.exists(filename, function(exists) {
    if(!exists) {
      response.writeHead(404, {"Content-Type": "text/plain"});
      response.write("404 Not Found\n");
      response.end();
      return;
    }

  if (fs.statSync(filename).isDirectory()) filename += '/index.html';
  fs.readFile(filename, "binary", function(err, file) {
    if(err) {
      response.writeHead(500, {"Content-Type": "text/plain"});
      response.write(err + "\n");
      response.end();
      return;
    }

    response.writeHead(200);
    response.write(file, "binary");
    response.end();
    });
 });

 server.listen(parseInt(port, 10));

 // socket.io
 var socket = io.listen(server);
{% endhighlight %}

Now, when we run our node instances:

{% highlight bash %}
node ./websocket-server.js 8001
Static file server running at
  => http://localhost:8001/
  CTRL + C to shutdown
     info  - socket.io started
{% endhighlight %}

And when we go to http://localhost, we can see successful websocket handshakes (which means we’re going over the nginx proxy):

{% highlight bash %}
 debug - client authorized
  info  - handshake authorized
  info  - handshaken 5ec456d53d8dc0b43f61b5f3acdf8b8e
{% endhighlight %}

Using this method you can successfully balance a cluster of web socket nodes, with some simple failure provided by the tcp_proxy module. Depending on the need and placement of nginx within your specific application stack, the ability to use this instead of something like HAProxy or some other balancer strategy could allow scaling many more connections per server without introducing significant complexity.

One thing that should be noted is you can no longer guarantee that a client will always connect to the same node process, like in the event of disconnection, so your application will need to understand this and implement session management across the cluster (such as using a redis or memcache session store, for example).

Happy hacking!
