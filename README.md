node-proxywrap
==============

This module wraps node's various `Server` interfaces so that they are compatible with the [PROXY protocol](http://haproxy.1wt.eu/download/1.5/doc/proxy-protocol.txt).  It automatically parses the PROXY headers and resets `socket.remoteAddress` and `socket.remotePort` so that they have the correct values.

    npm install proxywrap

This module is especially useful if you need to get the client IP address when you're behind an AWS ELB in TCP mode.

In HTTP or HTTPS mode (aka SSL termination at ELB), the ELB inserts `X-Forwarded-For` headers for you.  However, in TCP mode, the ELB can't understand the underlying protocol, so you lose the client's IP address.  With the PROXY protocol and this module, you're able to retain the client IP address with any protocol.

In order for this module to work, you must [enable the PROXY protocol on your ELB](http://docs.aws.amazon.com/ElasticLoadBalancing/latest/DeveloperGuide/enable-proxy-protocol.html) (or whatever proxy your app is behind).

Usage
-----

proxywrap is a drop-in replacement.  Here's a simple Express app:

    var http = require('http')
        , proxiedHttp = require('proxywrap').proxy(http)
        , express = require('express')
        , app = express()
        , srv = proxiedHttp.createServer(app); // instead of http.createServer(app)

    app.get('/', function(req, res) {
        res.send('IP = ' + req.connection.remoteAddress + ':' + req.connection.remotePort);
    });

    srv.listen(80);

The magic happens in the `proxywrap.proxy()` call.  It wraps the module's `Server` constructor and handles a bunch of messy details for you.

You can do the same with `net` (raw TCP streams), `https`, and `spdy`.  It will probably work with other modules that follow the same pattern, but none have been tested.

*Note*: If you're wrapping [node-spdy](https://github.com/indutny/node-spdy), its exports are a little strange:

    var proxiedSpdy = require('proxywrap').proxy(require('spdy').server);

**Warning:** By default, *all* traffic to your proxied server MUST use the PROXY protocol.  If the first five bytes received aren't `PROXY`, the connection will be dropped.  Obviously, the node server accepting PROXY connections should not be exposed directly to the internet; only the proxy (whether ELB, HAProxy, or something else) should be able to connect to node.

API
---

### `proxy(Server[, options])`

Wraps something that inherits from the `net` module, exposing a `Server` and `createServer`.  Returns the same module patched to support the PROXY protocol.

Options:

- `strict` (default `true`): Incoming connections MUST use the PROXY protocol.  If the first five bytes received aren't `PROXY`, the connection will be dropped.  Disabling this option will allow connections that don't use the PROXY protocol (so long as the first bytes sent aren't `PROXY`).  Disabling this option poses a security risk; it should be enabled in production.
