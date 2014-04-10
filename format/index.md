---
layout: page
title: "ZiNC Format"
---
The basic format of ZiNC messages is a three-part JSON structure:

```json
{
  "subscription": id,
  "request": { ... },
  "payload": { ... }
}
```

However, individual messages will generally not have all three parts,
but only those parts that are necessary to their function.

Unlike HTTP, streaming communications are fundamentally bi-directional;
messages can flow in both directions and can be initiated by either end.
Note however that ZiNC only specifies the _format_ of the communication;
the semantics are determined by the operations understood at each end of
the communication channel.  It is important that the two ends understand
what the other is capable of before beginning communication.

## Requests

Either end of the communication channel can initiate a _request_ on the
other end.

The simplest type of request just asks the other end to perform an
operation and does not send any additional data or want a response.
For example, a heartbeat operation might look like this:

```json
{
  "request": {
    "method": "invoke",
    "operation": "heartbeat"
  }
}
```

A simple request for a resource requires both a request object and a
subcription handle.  The handle will be sent back with a response object.

```json
{
  "subscription": 14,
  "request": {
    "method": "get",
    "resource": "/path/to/resource"
  }
}
```

Unlike HTTP, which is strictly one-request, one-response, streaming
protocols can have any number of responses to a request.  The previous
two examples have had 0 and 1 responses respectively, but it is also
possible to "subscribe" to a resource, that is, to request it and all
future versions of it.

```json
{
  "subscription": 14,
  "request": {
    "method": "subscribe",
    "resource": "/path/to/resource"
  }
}
```
