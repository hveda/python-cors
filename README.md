[![Build Status](https://travis-ci.org/synacorinc/python-cors.svg)](https://travis-ci.org/synacorinc/python-cors)


# CORS

> A Python package for dealing with HTTP requests and same-origin policies.


## Overview

This package was developed to improve automated HTTP API tests by being able to
automatically test that any requests we make can also be made from browsers with
scripts on other origins.

The code in this package strives to be mostly agnostic to the library you are
using to actually make the HTTP requests and thus mostly deals in an internally
defined `Request` class which simply contains the url, headers, and method as
properties. From here you can convert to whatever you need to send the request.

Of course, Python programmers tend to love the hell out of
[_requests_](https://github.com/kennethreitz/requests) (and, I mean,
who doesn't?), so there's also an included function you can use to send a
request including preflight and CORS header checks.


## Usage

### Client

#### Low-level

For maximum flexibility you can use the package's low-level functionality to
generate a preflight request and callable check methods to validate your
request.

```python

import requests
import cors.preflight
import cors.utils

# create my own request
request = requests.Request(
    "POST", "http://example.com",
    headers={"Content-Type": "application/json"},
    body="{}").prepare()

# generate any required preflight and validation checks
preflight, checks = cors.preflight.prepare_preflight(request)

# if a preflight was needed, send it and check its response
if preflight:
    response = requests.request(
        preflight.method,
        preflight.url,
        preflight.headers)

    # any failed check will raise cors.errors.AccessControlError
    for check in checks:
        check(response, request)

# must be OK.
response = requests.Session().send(request.prepare())

# check again that the actual response agrees with the preflight
for check in checks:
    check(response, request)

# We can also enforce access to the response headers.
# Now whenever we try to use a head which was not explicitly exposed by the CORS
# response headers, or is not a "simple response header" an AccessControlError
# is raised.
response.headers = cors.utils.ProtectedHTTPHeaders(
    response.headers.get("Access-Control-Allow-Headers", ""),
    response.headers)

```

The intention here is for you to write a suitable wrapper which accepts requests
in whatever form works best for your HTTP client library.

But for many people that library is `requests`, so...


#### High-level wrapper for python-requests

```python

import requests
from cors.clients.requests import send

my_request = requests.Request(
    "POST", "http://example.com",
    headers={"Content-Type": "application/json"},
    body="{}").prepare()

response = send(my_request)

```

Done. The `cors.clients.requests.send` function will inspect your prepared
request object and:

1. generate and send a preflight request if necessary
2. pick and run any necessary validation checks
3. wrap the response headers in a `ProtectedHTTPHeaders` instance.

When calling `send` you may also include a custom requests.Session instance, and
a flag to specify that CORS checks on the actual request should be if the server
comes back with a `5XX` error.


#### High-level wrapper for tornado async http client

```python

from tornado.httpclient import AsyncHTTPClient
import cors.clients.tornado

@coroutine
def my_cors_coroutine():
    client = AsyncHTTPClient()
    yield cors.clients.tornado.cors_enforced_fetch(client, "http://example.com")

@coroutine
def my_alternate_cors_coroutine():
    client = cors.clients.tornado.WrappedClient(AsyncHTTPClient())
    yield client.fetch("http://example.com")

```

~~The `enforce_cors_on_client` function replaces the AsyncHTTPClient object's~~
The `WrappedClient` class provides you an object with an interface identical to
`tornado.httpclient.AsyncHTTPClient` but whose `fetch` method with will perform
CORS preflight request and followup checks as appropriate and wrap the
response's headers. Wrapping a client is preferable to replacing its attributes
because AsyncHTTPClient uses a custom `__new__` method to attempt to share class
instances.

If you wish to explicitly perform a cors request and don't want to deal with a
wrapper object, you may directl use `cors_enforced_fetch` which can be called
with an unmodified client as its first argument.


### Server

#### No-fuss enabling of a cross-origin request

`generate_acceptable_preflight_response_headers` accepts a key-value mapping of
headers from a CORS preflight request and returns a key-value mapping of headers
which should be included in the response in order for the User-Agent to send its
followup request.

`generate_acceptable_actual_response_headers` accepts a key-value mapping of the
response headers generated by your request handler and returns a key-value
mapping of response headers including those necessary to let the response be
shared with the client script.

Below is an example implementation for a Tornado request handler.

```python

import tornado.web
from cors.preflight import (
    generate_acceptable_preflight_response_headers,
    generate_acceptable_actual_response_headers,
)

class MyHandler(tornado.web.RequestHandler):
    def options(self):
        headers = self.request.headers
        headers = generate_acceptable_preflight_response_headers(headers)
        map(self.set_header, *zip(*headers.iteritems()))

    def post(self):
        # at end of request
        headers = self._headers
        headers = generate_acceptable_actual_response_headers(headers)
        map(self.set_header, *zip(*headers.iteritems()))

```
