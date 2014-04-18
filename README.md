## flashlight

Lightweight chained web proxy written in go.

flashlight runs in one of two modes:

client - meant to run locally to wherever the browser is running, forwards requests to the server

server - handles requests from an upstream flashlight client proxy and actually proxies them to the final destination

Using CloudFlare, flashlight has the ability to masquerade as running on a different domain than it is.  The client simply
specified the "masquerade" flag with a value like "thehackernews.com".  flashlight will then use that masquerade host for
the DNS lookup and will also specify it as the ServerName for SNI (though this is not actually necessary on CloudFlare).
The Host header of the HTTP request will actually contain the correct host (e.g. getiantem.org), which causes CloudFlare
to route the request to the correct host.

Flashlight works for HTTPS requests by man-in-the-middling them.  Each
flashlight client uses its own generated private key and a corresponding self-
signed CA certificate.  It then generates certificates for different domains on
the fly, signing these with its CA certificate.  When flashlight first generates
its CA certificate, it installs it into the current user's system trust store as
a trusted root CA so that the dynamically generated domain-specific certs are
trusted by the browser and other HTTPS clients. 

### Usage

```bash
Usage of flashlight:
  -addr="": ip:port on which to listen for requests.  When running as a client proxy, we'll listen with http, when running as a server proxy we'll listen with https
  -help=false: Get usage help
  -masquerade="": masquerade host: if specified, flashlight will actually make a request to this host's IP but with a host header corresponding to the 'server' parameter
  -server="": hostname at which to connect to a server flashlight (always using https).  When specified, this flashlight will run as a client proxy, otherwise it runs as a server
  -serverPort=443: the port on which to connect to the server
```

**IMPORTANT** - when running a test locally, you must run the client first so that
it generates and installs cacert.pem into the user's trust store.  If cacert.pem
already exists (i.e. you ran the server first) you have to manually install
cacert.pem into your system trust store, otherwise the client will not trust the
server's certificate.  In production, this is not an issue because the TLS
connection is terminated at CloudFlare, which presents a real certificate signed
by an already trusted CA.

Example Client:

```bash
./flashlight -addr localhost:10080 -server getiantem.org -masquerade thehackernews.com
```

Example Server:

```bash
./flashlight -addr 0.0.0.0:443
```

Example Curl Test:

```bash
curl -x localhost:10080 http://www.google.com/humans.txt
Google is built by a large team of engineers, designers, researchers, robots, and others in many different sites across the globe. It is updated continuously, and built with more tools and technologies than we can shake a stick at. If you'd like to help us out, see google.com/careers.
```

On the client, you should see something like this for every request:

```bash
2014/04/18 11:32:28 Using localhost:10081 to handle request
```

On the server, you should see something like this for every request that's received:
```bash
2014/04/18 11:30:59 Handling request for: https://www.google.com/humans.txt
```