uwsgi-sse-offload
=================

uWSGI offload bridge between redis pubsub and server sent events (sse)

This is a fork of the uwsgi-realtime (https://github.com/unbit/uwsgi-realtime) project, exposing only sse features.

The bridge waits on a redis pubsub channel and whenever it receives a message it forwards it to the connected sse clients.

It is an offload engine so you can manage thousand of concurrent requests without bothering your workers/threads/async-cores

How it works
============

A client (read: a webbrowser) open an SSE connection to the webserver/proxy forwarding the request to uWSGI.

uWSGI (or your app) recognize (via internal routing or via special response headers) it is an sse session and forward it to the offload engine. The offload engine subscribe to a redis pubsub channel and starts waiting for messages.

Whenever a message is enqueued, the offload engine collects it and forward to the connected client.

Remember: the offload engine is fully non-blocking so you can manage thousand of clients concurrently while your blocking main app continues its job. The maximum number of clients is defined by about half of the file descriptors limit

Installation
============

The plugin is 2.0 friendly (it requires uWSGI >=2.0.8):

```sh
uwsgi --build-plugin https://github.com/unbit/uwsgi-sse-offload
```

will generate sse_offload_plugin.so in the current directory

you can eventually build a monolithic binary with sse-offload plugin in one-shot:

```sh
curl http://uwsgi.it/install | UWSGI_EMBED_PLUGINS="sse_offload=https://github.com/unbit/uwsgi-sse-offload" bash -s psgi /tmp/uwsgi
```

this will result in a binary in /tmp/uwsgi with psgi and sse-offload support

in the same way (for a python setup):

```sh
UWSGI_EMBED_PLUGINS="sse_offload=https://github.com/unbit/uwsgi-sse-offload" pip install uwsgi
```

Usage (via internal routing)
============================

Let's start with a simple perl clock. A perl script will publish in the 'clock' redis channel the current unix time (seconds since the epoch):

```perl
use Redis;

my $redis = Redis->new;

while(1) {
        sleep(1);
        $redis->publish('clock', time);
}
```

save it as clock.pl

now we want an html page that "subscribe" to the /whattimeisit url via sse (server sent events) and set the content of the 'clock' div to the received data (yes, they are the timestamps sent to redis)

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8" />
</head>
<body>
  <script>
    var source = new EventSource('/whattimeisit');
    source.onmessage = function(e) {
      document.getElementById('clock').innerHTML = e.data;
    };
  </script>

<div id="clock" />
</body>
</html>
```

(save it as clock.html)

finally we start uWSGI with a mule running clock.pl and a rule for mapping requests to /whattimeisit to the sse engine:

```ini
[uwsgi]
; eventually use absolue path for the plugin if it is not in the current directory
plugin = sse_offload
; bind on http port 9090
http-socket = :9090
; run clock.pl as a mule
mule = clock.pl
; map requests to / to the html file
static-map = /=clock.html
; route requests to ^/whattimeisit to the sse engine subscribed to the 'clock' redis channel
route = ^/whattimeisit sse:clock
; enable 1 offload thread
offload-threads = 1
```

Usage (app-governed)
====================

If you want to hold control over the sse url (for example for managing authentication and authorization) you can pass control of the sse url to your app and (after having done your checks) pass back the control to the offload engine.

There are various ways to accomplish this, the easiest is using uWSGI request vars (this time we use python):

```python
import uwsgi
def application(environ, start_response):
    if environ['PATH_INFO'] == '/whattimeisit':
        uwsgi.add_var('X-SSE-OFFLOAD', 'clock')
        return []
    else:
        ...
```
