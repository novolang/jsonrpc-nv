# Changelog

All notable changes to jsonrpc-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## [0.1.0] — 2026-09-28

The first implementation of the interface published as 0.0.1: the
error object and the faults, the ids, the messages, the codec and the
Content-Length framing.

### Behaviour the interface left open

- `error_for_fault` answers -32700 for text that is not JSON and for a
  framing fault, -32602 Invalid params for `params` of the wrong shape,
  and -32600 Invalid Request for everything else that parsed.
- A refusal carries the request's id when the text is an object whose
  `id` can be read, and the null id otherwise (section 5).
- `method_not_found`, `invalid_params` and `internal_error` carry their
  argument as `data`.
- An id or an error code is read only from a number written as an
  integer inside the signed 64-bit range; `1.0`, `1e3` and
  `9223372036854775808` are refused.
- A header name must be RFC 9110 token characters; `Content-Length`
  must be at most 18 decimal digits; a `Content-Type` charset of
  `utf-8` or `utf8`, in any case and quoted or not, is accepted, and no
  charset is accepted.
- `id_text` renders a string id as JSON text, with its quotes.

### Changes to the interface

- `JrpcRead` gains `JrpcReadMixed(messages, refusals)`: a batch some of
  whose members are not valid answers the valid members and a refusal
  for each of the others, as section 6's examples require.
- `jrpcerr.integral` is the rule ids and error codes are read by.

### Toolchain

- The toolchain floor is 0.14.0.  The bodies target novo 0.14.0 and
  carry no workaround for a compiler defect.

### Tests

- 57 tests in five suites, with 100% line coverage over `src/`
  measured by `tests/coverage.sh`.  `spec_tests.nv` holds every example
  of the specification's section 7 and this project's MCP server
  traffic.

## [0.0.1] — 2026-09-17

**The interface, published before anyone implements it.** Every public
type and function carries its full signature, its effect row and its
doc comment; every body is `todo()`; the release is recorded
`implemented = false`.

### Added

- `jrpcmsg` — the load-bearing interface. `JrpcRequest.id` is `?JrpcId`
  and `None` IS the notification, which is section 4.1's own definition
  rather than a flag beside the id; so a request that is a notification
  and has an id cannot be written, and a server that answered one would
  have had to invent the id. `JrpcOutcome` is an enum with a result arm
  and an error arm, so section 5's "exactly one" holds by construction.
  `batch` refuses an empty list, so section 6's empty-array special
  case lives at the one place a batch is made. `replies_needed` is
  section 6's sentence implementations skip: a batch of nothing but
  notifications answers zero, and zero means send no bytes rather than
  `[]`.
- `jrpcid` — the identifier as the three shapes section 4 permits, so
  the id `1` and the id `"1"` stay different through a round trip.
  `id_equals` is false for two nulls, because a null id means the
  request's id could not be determined and two undetermined ids are not
  the same id. A fractional number is refused at the decode rather than
  accepted as an id that cannot be returned unchanged.
- `jrpcerr` — two types, because they have two audiences. `JrpcError`
  is section 5.1's error object, written by the method and read by the
  peer. `JrpcFault` is what this package could not do, read by the
  server that received the bytes. `error_for_fault` is the mapping
  between them, written once: only non-JSON text is -32700, and
  everything that parsed and is not a valid request object is -32600.
  `is_framing_fault` separates a fault that leaves the byte stream in
  step from one that does not, which is the difference between
  answering an error and closing the connection.
- `jrpcframe` — the base protocol's `Content-Length` framing as a
  feed-and-drain reader, so this package reads no socket. `feed` takes
  bytes and answers at most one message; `take` drains the rest;
  `finish` is how a caller reports the close that turns a half-read
  frame into `JrpcTruncated`. `content_length` counts UTF-8 bytes, not
  characters, which is the framing bug that works on a developer's
  machine and truncates in production.
- `jrpccodec` — decoding answers a `Result` because the text came from
  a peer; encoding answers a `Str` and no `Result`, because every value
  of `JrpcEnvelope` is a message the specification permits.
  `decode_or_error` hands a server either the envelope or the response
  it must send, so a read loop makes no protocol decisions of its own.
  It answers a `JrpcRead` and **not** a `Result`: both arms are values
  the caller uses, and a refusal is a message to SEND rather than a
  failure to propagate. The refusal carries the fault beside the
  response, so `jrpcerr.is_framing_fault` can still say whether to
  answer or close — a question the earlier shape, which handed back
  only the response, left a caller unable to ask.
- `jrpcerr.JrpcError` is an `Error`. Section 5.1's error object is what
  a failed method answers with, so it is what a `Result`'s error
  position holds in a server built on this package (lsp-nv's parameter
  readers are the first), and SPEC § 3.4 requires the impl for that to
  be legal. Its `message` is section 5.1's `message` member.

### Known

- `novo test` is red, and that is the release's expected state: every
  assertion in the API suite reaches `not implemented:
  jsonrpc-nv.<module>.<fn>`.
- **`params`, `result` and `data` are `std.json`'s `JsonValueH`.** A
  second JSON value type in the assembly would mean every caller
  converting a document before it could send it, and every consumer of
  this package converting it back. What that costs is that `std.json`
  does not build at `@tier(embedded)`, so this package does not either;
  it makes no device claim and ships no probe.
- **A compiler defect is filed against a qualified type name.**
  `bugs/type-system-meta/qualified-type-name-with-unknown-module-is-silently-accepted.md`:
  a type written `<module>.<Name>` is accepted whatever the module is,
  so `json.JsonValueH` builds green and is a different type from
  `JsonValueH`. This package writes stdlib types unqualified, which is
  what every published package already does, so no workaround is in the
  source.
