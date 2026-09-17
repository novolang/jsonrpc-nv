# Changelog

All notable changes to jsonrpc-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

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
