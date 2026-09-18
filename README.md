# jsonrpc-nv

JSON-RPC 2.0 is a stateless, light-weight remote procedure call
protocol, specified at
[jsonrpc.org](https://www.jsonrpc.org/specification). A call and its
answer are JSON objects. This package brings the protocol's objects to
novo-lang as typed values, together with the header framing that the
Language Server Protocol and the Debug Adapter Protocol put in front of
them.

[lsp-nv](https://novo-lang.org/packages/lsp-nv) and
[dap-nv](https://novo-lang.org/packages/dap-nv) are built on this
package.

**Status: NOT IMPLEMENTED — interface only.** Every function is
declared with its full signature, but every body is a `todo()` that
panics when called. The package is published so its design can be
reviewed and depended on before it is implemented. Version 0.1.0 will
be the first working release.

## What JSON-RPC is

A **request** is a JSON object with four members. `jsonrpc` is the
string `"2.0"` and is mandatory. `method` is the name of the procedure
to call. `params` holds the arguments and may be absent. `id` is an
identifier the client chose, and may be absent.

A request with no `id` is a **notification**. The server must not
answer it. That is the whole difference: there is no other member and
no flag.

A **response** is a JSON object with three members. `jsonrpc` is again
`"2.0"`. `id` is the identifier of the request being answered. The
third member is either `result`, holding whatever the method returned,
or `error`, holding an **error object**. Exactly one of the two is
present.

An error object has a `code`, which is an integer, a `message`, which
is one short sentence, and an optional `data` of any shape the method
likes. The specification reserves the codes -32768 to -32000 and names
five of them.

| Code | Meaning |
| --- | --- |
| -32700 | Parse error — the text received was not valid JSON |
| -32600 | Invalid Request — the JSON sent is not a valid request object |
| -32601 | Method not found |
| -32602 | Invalid params |
| -32603 | Internal error |
| -32099 to -32000 | Reserved for implementation-defined server errors |

A **batch** is a JSON array of request objects. The server answers it
with an array holding one response per non-notification member. A batch
of nothing but notifications is answered with no message at all.

JSON-RPC says nothing about where one message ends and the next begins.
The **base protocol** of the Language Server Protocol answers that, and
the Debug Adapter Protocol uses the same answer: a header section that
looks like an HTTP message, then a blank line, then the content.

```
Content-Length: 41\r\n
\r\n
{"jsonrpc":"2.0","method":"initialized"}
```

`Content-Length` is the one mandatory header, and its value is the
number of **bytes** of the UTF-8 encoded content. The only other header
either protocol defines is `Content-Type`, whose charset must be
`utf-8`.

Every function in this package performs no input or output. It reads no
socket and opens no pipe: the program using it reads bytes however it
likes and hands them over.

## Install

```
novo pkg add jsonrpc-nv
```

## Example

```novo
use std.bytes
use jrpcid
use jrpcmsg
use jrpccodec
use jrpcframe

fn main() [io]
    // Ask a peer for something, and frame the request for sending.
    let call = jrpcmsg.call(jrpcid.id_int(1), "initialize", None)
    let out = jrpcframe.frame(jrpccodec.encode_request(call))
    println("${bytes.len(bytes.from_byte_list(out))} bytes to write")

    // A reader holds the bytes that have arrived but are not yet a
    // whole message.
    let reader = jrpcframe.reader(jrpcframe.default_limits())

    // Bytes the transport read. `feed` answers at most one message.
    match jrpcframe.feed(reader, out)
        Err(f)   => println("framing: ${f.message()}")
        Ok(step) =>
            match step.message
                None    => println("a partial message; read more")
                Some(text) =>
                    // The content, decoded. A text this server cannot
                    // read becomes the response it must send back.
                    match jrpccodec.decode_or_error(text)
                        JrpcReadRefusal(reply, _) => println("refused: ${jrpccodec.encode_response(reply)}")
                        JrpcReadEnvelope(env)     => println("${jrpcmsg.replies_needed(env)} reply needed")
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: jsonrpc-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `jrpcerr` | The error object the protocol defines, the reserved codes, and the faults a decode can produce. |
| `jrpcid` | The request identifier in its three permitted shapes, and the rule that decides whether a response answers a request. |
| `jrpcmsg` | The request, the response, the batch, and the constructors that refuse the shapes the specification forbids. |
| `jrpccodec` | Text to typed values and back. |
| `jrpcframe` | The `Content-Length` framing, as a reader fed bytes and drained of messages. |

## How to choose an entry point

**`jrpccodec.decode` reads one message.** It takes the content of one
frame, not a stream, and answers the envelope or the fault.

**`jrpccodec.decode_or_error` reads one message and prepares the
refusal.** It answers a `JrpcRead`: either the envelope, or the
response the server must send because it could not read the text,
together with the fault that produced it. It is not a `Result` —
neither arm is a failure to propagate, and the refusal is a message to
send rather than an error to raise. A server's read loop written
against this one makes no protocol decisions of its own; it asks
`jrpcerr.is_framing_fault` of the refusal's fault to decide between
answering and closing.

**`jrpcframe.feed` takes bytes and answers at most one message.**
`jrpcframe.take` answers the next message already buffered, adding
nothing. A read that delivered three messages gives one from `feed`,
so a caller loops on `take` until it answers `None`.

**`jrpcframe.frame` produces the bytes to write.** `header_text` and
`content_length` are the two halves, for a caller writing its own
buffer.

## The rules a user needs

1. **A notification is a request with no `id`, and the server must not
   answer it** (section 4.1). `JrpcRequest.id` is optional, and `None`
   is what makes the request a notification.
2. **A batch whose members are all notifications is answered with
   nothing at all** (section 6). Not an empty array.
   `jrpcmsg.replies_needed` answers zero, and zero means send no bytes.
   A client reading `[]` has been told its batch produced results.
3. **A batch may not be empty** (section 6). `jrpcmsg.batch` refuses
   one, and an empty array on the wire is an Invalid Request answered
   with a single response rather than an array.
4. **A response carries a `result` or an `error`, never both and never
   neither** (section 5). `JrpcOutcome` has one arm for each, so the
   other two shapes cannot be written.
5. **An `id` is a string, a number or null, and the three are
   different** (section 4). The id `1` and the id `"1"` do not match.
6. **Two null ids never match.** A null id means the request's id could
   not be determined (section 5), and two undetermined ids are not the
   same id. A client that matched them would deliver a parse-error
   response to whichever request was first in its table.
7. **A number used as an id must have no fractional part** (section 4).
   This package refuses one at the decode rather than accepting an id
   it cannot return unchanged.
8. **`params` is an array or an object, and nothing else** (section
   4.2).
9. **Method names beginning `rpc.` are reserved** (section 4).
   `jrpcmsg.is_reserved_method` is the check to run at start-up.
10. **Only text that is not JSON is a Parse error** (section 5.1).
    Everything that parsed and is not a valid request object is an
    Invalid Request. `jrpcerr.error_for_fault` applies the mapping.
11. **A server error code must be inside -32099 to -32000** (section
    5.1). `jrpcerr.server_error` refuses one outside that range rather
    than clamping it. An application code outside the whole reserved
    block needs no permission: `jrpcerr.error` takes any code.
12. **`Content-Length` counts bytes, not characters.** A message
    holding one character outside ASCII is longer in bytes than in
    characters, and a framer that wrote the character count truncates
    it. `jrpcframe.content_length` is the count to write.
13. **A framing fault means close the connection.** A message fault
    leaves the byte stream in step, so the server answers an error and
    reads the next frame. A framing fault does not, so a server that
    answers and keeps reading has resynchronised on whatever bytes
    happened to follow. `jrpcerr.is_framing_fault` tells the two apart.
14. **Tell the reader when the peer closed.** `jrpcframe.finish` is
    what turns a close with a half-read frame into `JrpcTruncated`. A
    close with nothing buffered is an ordinary end of connection and no
    fault.
15. **The two limits are the caller's.** A header section with no blank
    line and a `Content-Length` of nine quintillion are both denial of
    service costing the attacker one connection. The content limit is
    checked when the header is read, before any buffer is grown.

## What is not included

- **A transport.** This package reads no socket, opens no pipe and
  starts no process. A program using it reads bytes itself and hands
  them to `jrpcframe.feed`.
- **A dispatcher.** Routing a method name to a handler is the server's
  own table, and a server that wanted one from a library would be
  taking its concurrency model from a library too.
- **A JSON value type of this package's own.** `params`, `result` and
  an error's `data` are the standard library's `JsonValueH`, so a
  caller uses the JSON parser it already has and nothing converts on
  the way in or out.
- **JSON-RPC 1.0.** A peer speaking it sends no `jsonrpc` member, which
  this package refuses as `JrpcBadVersion`.
- **HTTP as a transport.** JSON-RPC over HTTP puts the message in a
  request body, which is a different framing entirely; take
  [http-codec-nv](https://novo-lang.org/packages/http-codec-nv) for
  that.
- **Newline-delimited JSON framing.** Some peers separate messages with
  a newline instead of a header. That is a third framing, and this
  package implements the one LSP and DAP define.

## Related packages

- [lsp-nv](https://novo-lang.org/packages/lsp-nv) is the Language
  Server Protocol's own types, carried in these messages.
- [dap-nv](https://novo-lang.org/packages/dap-nv) is the Debug Adapter
  Protocol's, carried the same way.
- [http-codec-nv](https://novo-lang.org/packages/http-codec-nv) reads
  and writes HTTP/1.1 messages. Its reader has the same feed-and-drain
  shape as this package's.
- [websocket-codec-nv](https://novo-lang.org/packages/websocket-codec-nv)
  is a third framing over a byte stream, for a peer that wants one.

## Tests

```bash
novo test tests/jrpcmsg_tests.nv    # ids, notifications, batches, outcomes
novo test tests/jrpcframe_tests.nv  # the codec, the framing and the limits
```

The normative source is the JSON-RPC 2.0 specification, and each
assertion names the section it comes from. The framing assertions come
from the Language Server Protocol's base protocol document. The
reference implementations are the Python package `jsonrpcserver`, for
the batch rules, and the Rust crate `jsonrpc-core`, for the id type.

The suite asserts that a notification has no id, that a batch of
notifications requires no reply at all, that an empty batch is refused,
that the id `1` does not match the id `"1"`, that two null ids do not
match each other, that a fractional id is refused, that only
non-JSON text is a Parse error, that a frame fed in pieces produces one
message, and that `Content-Length` counts bytes.

The tests compile today and fail at run, each on the
`not implemented: jsonrpc-nv.<module>.<fn>` panic that is its body.
That is the expected state of an interface release. They turn green one
at a time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `jrpcerr` — the error object, the codes, the faults | the types are declared; every body is a `todo()` |
| `jrpcid` — the identifier and its matching rule | the types are declared; every body is a `todo()` |
| `jrpcmsg` — the request, the response, the batch | the types are declared; every body is a `todo()` |
| `jrpccodec` — text to values and back | every body is a `todo()` |
| `jrpcframe` — the `Content-Length` framing | the types are declared; every body is a `todo()` |

## Licence

Apache-2.0. See [LICENSE](LICENSE).
