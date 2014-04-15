---
layout: page
title: ZiNC Streaming Communications Protocol
---

## Description

The use of HTTP, URL and REST communications is well understood
for request/response interactions.  The use of continuously
connected, streaming interactions over the web is less well
understood.

The ZiNC protocol is designed to eliminate much of the confusion
and uncertainty surrounding streaming interactions.

The ZiNC protocol is a simple, well-defined protocol that builds
on top of other standard protocols, but leaves the definition of
operational semantics up to the applications.

Building on top of WebSockets, the [ZiNC protocol](/format) is compatible
with most modern web servers and browsers.

ZiNC uses JSON as its transport because it is easily and natively
understood by all browsers; but this does not require servers or
applications to use JavaScript.  It is simply an encoding format.

ZiNC uses JSON to encode requests, but stays in the spirit of
HTTP requests and URIs.

ZiNC builds on top of [JSONAPI](http://jsonapi.org/) to define its object payloads.

## Implementations

Simple [APIs](/api) to use to communicate using the ZiNC protocol are available for:

* JavaScript
* Java
