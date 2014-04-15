---
layout: page
title: "ZiNC API structure"
---
For ease of use, ZiNC is provided with language bindings
for popular languages.  Java and JavaScript are provided
based on the atmosphere runtime websocket library.

For obvious reasons, JavaScript in the browser only provides
the "Requestor" portion of the library; however, the responder
portion is available for node.js applications.

This section describes the overall design of the API in
approximately language-neutral terms.  The individual language
bindings should reflect the spirit of this API but use
the most appropriate features in the local language.

## The Zinc class

Applications wishing to use ZiNC should ideally create a
single instance of the central ZiNC class.  This object
is responsible for creating the Requestor and Responder
objects.

Because the communication between machines is essentially
peer-to-peer, a Zinc instance can pool connections, even
connections that were created in the "opposite" direction.

In languages which support threading, all ZiNC API classes
are inherently thread-safe, in order to support the maximum
amount of pooling of connections.

```
newRequestor :: URL -> Requestor
```

In order to start a connection that can send requests, the
`newRequestor` method should be called with the URL of the
server intended to handle the requests.

If there is already a connection between this machine and the
requested machine, it will be reused, but the requestor objects
will be separate.

The URL is a URL, with scheme, host, port and path portions (although
query options should not be used).  If the scheme and port are not
specified the defaults of http and 80 will be used.

```
handleResource :: String(resource) -> ResourceHandler -> Void
```

In order for an individual request to be processed on the server,
there must be a handler for the requested resource.

This method associates a `ResourceHandler` with a resource prefix.
The `ResourceHandler` with the most precise prefix will be selected.
If the resource prefix is null, this handler will handle all
resources that are otherwise not handled.

Calling this multiple times with the same resource prefix will
update the handler for that prefix.

```
getMulticastResponse :: String(forResource) -> MulticastResponse
```

Obtain a specified multicast response, creating it if necessary.
See the discussion under `MulticastResponse`.

```
addConnection :: OutgoingConnection -> IncomingConnection
```

The `addConnection` method is part of the server-side implementation.
See the discussion under `OutgoingConnection`.

## ZincExceptionHandler

A connection can break at any time.  Instances of the
`ZincExceptionHandler` class are used to receive information
about the connections as they break

brokenConnection :: Requestor|Responder -> Void

## Requestor

A requestor is created in order to initiate communication
with a server which is accepting connections.  Once created,
the communication channel is bidirectional.

If there is already a channel between the two endpoints which
has been established (in either direction) it is reused.

If requests are to be sent in both directions, each end of the
channel needs to create a requestor specifying the URL of the
other end, except when a requestor is created specifically from
a `HandleRequest` object.

For any of the operations which can receive a response,
a `ResponseHandler` is required in the method call.  However,
if the user is not interested in the reponse(s), specifying
a null or undefined response handler will prevent any
responses being sent.

Operations include:

```
subscribe :: String(resource) -> ResponseHandler -> MakeRequest
```

Create a request to start subscribing to a resource.  The
specified resource does not need to exist.  Once processed,
any responses to this request will be sent to the `ResponseHandler`.

```
unsubscribe :: ResponseHandler -> Void
```

Cancel an existing subscription request (from any source).  In theory,
no more responses for this request will be delivered to the response
handler.  In practice, race conditions may mean that one or more messages
are delivered, although library implementations will generally attempt
to mitigate this issue.

```
get :: String(resource) -> ResponseHandler -> MakeRequest
```

Request a resource if it currently exists without subscribing to updates.
While this is a very common operation in single-shot HTTP protocols, its
use in ZiNC is discouraged, since there is the real question of why you
would ever want stale data.

Pragmatically, this consists of a subscribe followed by an unsubscribe.

```
put :: String(resource) -> MakeRequest
```

Create a request object that, when activated will cause a payload 
consisting of one or more objects to be sent for storage.

```
update :: String(resource) -> MakeRequest
```

Update functions identically to put, except the semantics expect that
the objects being put simply reflect _deltas_ to the existing objects.
Only the fields specified will be changed, and special semantics exist
for updating relationships.

```
create :: String(resource) -> ResponseHandler -> MakeRequest
```

Request that a resource be created.  It should be an error for the
requested resource to already exist.  

The resource here is not the resource of the object to _be_ created,
but the object which should be identified as capable of doing the creating.

Likewise, there is no inherent requirement to pass any payload
data or options; it is up to the creating resource to determine what
is required.

Once the resource has been created, the client library will automatically
subscribe to it.  This first means that the new object will be returned
once created, but also that any subsequent changes will be sent to this
server.

```
delete :: String(resource) -> MakeRequest
```

Request that a specific resource be deleted.  Not all servers will respect
delete requests.

```
invoke :: String(resource) -> ResponseHandler -> MakeRequest
```

Request that a method be invoked on the resource.  The options and payload
may contain information about how the operation should be carried out.

## MakeRequest

Once one of the methods on the `Requestor` interface has been called,
the user will be returned a `MakeRequest` object.  This can then be
populated and executed.

```
setOption :: String(name) -> Any(value) -> Void
```

Set an option field in the request to a value.  This option is
transmitted with the request to the server.  Only one value can
be associated with a given name, but it may be any valid JSON value.

```
setPayload :: JSONObject -> Void
```

The payload may be set to any valid JSONAPI value.

```
send :: Void
```

Send the request to the server.

## ResponseHandler

Objects meeting this contract are created by
the user and can then be associated with certain
kinds of request.  When a matching response
is received by the library, the response method
will be invoked.

```
response :: MakeRequest -> JSONObject -> Void
```

The request is the request object that caused
this subscription to be set up so that the response
arrived.

The object is the payload data in the response.

## ResourceHandler

On the server side, the ZiNC library unpacks the request and
identifies the resource to be acted upon.  From this and the registrations
in the main Zinc object it identifies a `ResourceHandler` (if none can
be identified, the request is rejected).

Objects conforming to `ResourceHandler` are created by the user on the server and
passed to the Zinc object's handleResouce method.

When a `ResourceHandler` has been identified, its handle method is invoked,
except when the method is unsubscribe, which is handled internally.

```
handle :: HandleRequest -> Response -> Void
```

The user method is invoked with a request and a response object, much
as with standard server code.

However, the response object may be null (indicating no response is
desired) or else will be long-lived (beyond the lifetime of the request,
until cancelled).

## HandleRequest

This object is the server-side version of the `MakeRequest` object, which 
can be used to discover the nature of the request.

```
method :: String
```

Obtain the method being used (subscribe, put, create, invoke, etc).

```
options :: Map(String,Object)
```

Obtain all the options associated with the request.

```
payload :: JSONObject
```

Recover the JSON payload associated with the request (if any).

```
obtainRequestor :: Requestor
```

In certain circumstances, a bi-directional connection may be
desirable in response to a given request.  In this case, the
server code can call this method to obtain a `Requestor` object
attached to the "server" at the far end.

Note that in this instance the so-called "server" can in fact
be located in a browser without difficulty, since it may have
initiated the connection.

## Response

A response is created on the server side for every request
that indicates it desires a response.

The response lasts until such time as it is specifically
cancelled (using the `unsubscribe` method).  The resource handler
may choose to send a response during the invocation, immediately
after, or at some later point or points.

Once cancelled, attempting to send further payloads will cause
an exception to be thrown.

```
sendPayload :: JSONObject -> Void
```

Send the payload to the user.

## MulticastResponse

Many applications may have the desire to send the same payload
to multiple clients simultaneously.  To avoid this becoming an
application responsibility, the Zinc object allows for the
mapping of names to `MulticastResponse` objects.  Once the
`MulticastResponse` is obtained, an individual client `Response`
can be attached to it.

The `MulticastResponse` "inherits" the `Response` methods, and
sending a payload to a `MulticastResponse` sends it to all the
attached `Response` objects.

The `MulticastResponse` automatically cleans up after any broken
connections and only throws an exception if ALL the attached
responses have died.

```
attachResponse :: Response -> Void
```

## JSONObject

All payload information is encoded as `JSONObject` instances.  Where
applicable, it may be useful for a binding to have a higher-level
construct to create these payload objects.

The definition of a `JSONObject` depends on the language binding.  In
JavaScript, a simple hash suffices.

Note that most uses of `JSONObject` in this API are for payloads which
should conform to the [JSONAPI](http://jsonapi.org) specification.

## OutgoingConnection

One of the complexities of Web software is not just
the number of languages and formats involved
but also the range of web servers and technologies that
can be used to handle web requests in any given language.
Rather than attempt to encompass all such
technologies in a single library, ZiNC assumes that additional
code will be provided by third party projects to link
to different technologies.

In each case, this third party code is responsible for
establishing a listening socket, accepting connections
and doing the appropriate WebSocket negotiation.  Ultimately
it is responsible for creating an object conforming to
`OutgoingConnection` and passing this to Zinc's 
addConnection method; in response it receives an
`IncomingConnection` object.

The `OutgoingConnection` object is retained by the ZiNC
system and used every time a message should be sent to
the far end of the socket.

```
sendTextMessage :: String -> Void
```

When this method is invoked, the string should be sent
as a single message to the remote end.

## IncomingConnection

The `IncomingConnection` object is created by the `Zinc`
class in response to the web server calling `addConnection`
(see `OutcomingConnection` for more details).

When new messages arrive over the physical socket
connection, the `receiveTextMessage` method on this
object should be invoked by the server.

```
receiveTextMessage :: String -> Void
```

The message is a single, complete, text message from
the client which should contain an appropriately
formatted ZiNC JSON message.

There is also a method by which the server can notify
the ZiNC system that the connection has died.

```
connectionBroken :: Void
```

When the server code notices that the connection is
broken, it should notify the ZiNC code so that it can
clean up any necessary resources.
