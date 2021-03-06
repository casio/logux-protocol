# Logux Protocol

Logux protocol is used to synchronize events between [Logux logs].

This protocol is based on simple JS types: boolean, number, string, array
and key-value object.

You can use any encoding and any low-level protocol: JSON encoding
over WebSockets, XML over AJAX and HTTP “keep-alive”.
Low-level protocol must guarantee messages order and content.
Main way is JSON over WebSocket Secure.

[Logux logs]: https://github.com/logux/logux-core

## Versions

Protocol uses two major and minor numbers for version.

```ts
[number major, number minor]
```

If other client uses bigger `major`, you should send `wrong-protocol` error
and close connection.

## Messages

Communication is based on messages. Every message is a array with string
in the beginning and any types next:

```ts
[
  string type,
  …
]
```

First string in message array is a message type. Possible types:

* [`error`]
* [`connect`]
* [`connected`]
* [`ping`]
* [`pong`]
* [`sync`]
* [`synced`]

If client received unknown type, it should send `wrong-format` error
and continue communication.

Protocol design has no client and server roles. But in most real cases
client will send `connect` and `ping`. Server will send `connected` and `pong`.
Both will send `error`, `sync` and `synced`.

[`connected`]: #connected
[`connect`]:   #connect
[`synced`]:    #synced
[`error`]:     #error
[`ping`]:      #ping
[`pong`]:      #pong
[`sync`]:      #sync

## `error`

Error message contains error description and error type.

```ts
[
  "error",
  string errorType,
  (any options)?
]
```

Right now there are 7 possible errors:

- `wrong-protocol`: client Logux protocol version is not supported by server.
  Error options object will contain `supported` key with array
  with supported major versions and `used` with used version.
* `wrong-format`: message is not correct JSON, is not a array or have no `type`.
  Error options will contain bad message string.
* `unknown-message`: message’s type is not supported. Error options will contain
  bad message type.
- `wrong-credentials`: sent credentials doesn’t pass authentication.
* `missed-auth`: not `connect`, `connected` or `error` messages was sent
  before authentication. Error options will contain bad message string.
* `timeout`: a timeout was reached. Errors options will contain timeout duration
  in milliseconds.
* `wrong-subprotocol`: client application subprotocol version is not supported
  by server. Error options object will contain `supported` key with array
  with supported major versions and `used` with used version.

## `connect`

After connection was started some client should send `connect` message to other.

```ts
[
  "connect",
  number[] protocol,
  string nodeId,
  number synced,
  (object options)?
]
```

Receiver should check [protocol version] in second position in message array.
If major version is different from receiver protocol,
it should send `wrong-protocol` error and close connection.

Third position contains unique node name. Same node name is used in default
log timer, so sender must be sure that name is unique.
Client should use UUID if it can’t guarantee name uniqueness with other way.

Fourth position contains last `added` time used by receiver
in previous connection (`0` on first connection).
message with all new events since `synced` (all events on first connection).

Fifth position is optional and contains extra client option in object.
Right now protocol supports only `subprotocol` and `credentials` keys there.

Subprotocol version is a string in [SemVer] format. It describes a application
subprotocol, which developer will create on top of Logux protocol.
If other node doesn’t support this subprotocol version,
it could send `wrong-subprotocol` error.

Credentials could be in any type. Receiver may check credentials data.
On wrong credentials data receiver may send `wrong-credentials` error
and close connection.

In most cases client will initiate connection, so client will send `connect`.

[protocol version]: #versions
[SemVer]: http://semver.org/

## `connected`

This message is answer to received [`connect`] message.

```ts
[
  "connected",
  number[] protocol,
  string nodeId,
  [number start, number end],
  (object options)?
]
```

`protocol`, `nodeId` and `options` are same with [`connect`] message.

Fourth position contains [`connect`] receiving time and `connected` sending time.
Time should be a milliseconds elapsed since 1 January 1970 00:00:00 UTC.
Receiver may use this information to calculate difference between sender
and receiver time. It could prevents problems if somebody has wrong time
or wrong time zone. Calculated time fix may be used to correct
events `created` time in [`sync`] messages.

Right after this message receiver should send [`sync`] message with all new events
since last connection (all events on first connection).

In most cases client will initiate connection, so server will answer `connected`.

## `ping`

Client could send `ping` message to check connection.

```ts
[
  "ping",
  number synced
]
```

Message array contains also sender last `added`. So receiver could update it
to use in next [`connect`] message.

Receiver should send [`pong`] message as soon as possible.

In most cases client will send `ping`.

## `pong`

`pong` message is a answer to [`ping`] message.

```ts
[
  "pong",
  number synced
]
```

Message array contains sender last `added` too.

In most cases server will send `pong`.

## `sync`

This message contains new events for synchronization.

```ts
[
  "sync",
  number synced
  (object event, array created)+
]
```

Second position contain biggest `added` time from events in message.
Receiver should send it back in [`synced`] message.

This message array length is dynamic. For each event sender should add
2 position: for event object and event’s `created` time.

Event object could contains any key and values, but it must contains at least
`type` key with string value.

`created` time is a array with numbers and strings. It actual format depends
on used timer. For more details read [Logux Core docs].
For example, standard timer generated:

```ts
[number milliseconds, string nodeId, number orderInMs]
```

Sender and receiver should use same timer type to have same time format.

Every event should have unique `created` time. If receiver’s log already
contains event with same `created` time, receiver must silently ignore
new event from `sync`.

Received event’s `created` time may be different with sender’s log,
because sender could correct event’s time based on data from [`connected`]
message. This correction could fix problems when some client have wrong
time or time zone.

[Logux Core docs]: https://github.com/logux/logux-core#created-time

## `synced`

`synced` message is a answer to [`sync`] message.

```ts
[
  "synced",
  number synced
]
```

Receiver should mark all events with lower `added` time as synchronized.
