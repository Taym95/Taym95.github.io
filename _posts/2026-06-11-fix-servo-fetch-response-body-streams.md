---

layout: post
title: "Fixing Servo Fetch BYOB response streams"
description: "How following the Fetch spec for BodyInit byte streams made constructed Response bodies readable with BYOB readers in Servo."
date: 2026-06-11 00:00:00 +0200
series: Servo generational run
categories:
  - servo
  - servo-generational-run
tags:
  - servo
  - fetch
  - response
  - streams
  - byob
pr_url: https://github.com/servo/servo/pull/45589
---

This is the second post in my Servo generational run series. This one is about a small Fetch API fix in Servo.

The patch itself is not big, but the failure was interesting because it was not in the BYOB reader code directly. It was in the way Fetch was creating response body streams.

The fix landed in [Servo PR #45589](https://github.com/servo/servo/pull/45589).

A bit of context: I had already implemented BYOB stream support in Servo before this. So the problem was not that Servo had no BYOB reader support at all.

The BYOB reader path was there, and it was doing the right check. BYOB readers only work with readable byte streams. The issue was earlier than that: some Fetch body extraction paths were creating default streams instead of byte streams.

The failing test was:

```text
Read text response's body as readableStream with mode=byob
```

It came from `fetch/api/response/response-consume-stream.any.js`. The same pattern also failed for `URLSearchParams`, `ArrayBuffer`, and `FormData` response bodies.

<figure class="post-figure">
  <img src="{{ '/assets/wpt_dashbord_byob_before_fix.png' | relative_url }}" alt="WPT dashboard showing Servo failing the BYOB response stream test while Firefox, Chromium, and Safari pass">
  <figcaption>Before the fix, the WPT dashboard showed Servo failing the BYOB response stream test while Firefox, Chromium, and Safari passed.</figcaption>
</figure>

The important part is the reader mode:

```js
response.body.getReader({ mode: "byob" })
```

BYOB means "bring your own buffer". Instead of the stream allocating a new buffer for every read, the caller gives it a buffer to fill.

That only works when the stream is a readable byte stream.

## Before the fix

Servo already had two helpers for creating a `ReadableStream` from bytes:

```rust
ReadableStream::new_from_bytes(...)
ReadableStream::new_from_bytes_with_byte_reading_support(...)
```

The first helper creates a normal default stream. It can still enqueue byte chunks, but the stream does not have a byte stream controller.

The second helper creates a stream with byte reading support. That is the kind of stream that can be read with a BYOB reader.

Before the fix, the text response path looked like this:

```text
new Response("This is response's body")
-> BodyInit::String extraction
-> string bytes are created
-> Servo creates the body stream with new_from_bytes
-> response.body is a default stream
-> getReader({ mode: "byob" }) asks for a BYOB reader
-> Streams rejects it
```

Streams was right to reject this.

`ReadableStreamBYOBReader` should only be created for readable byte streams. If the stream has a default controller, rejecting a BYOB reader is correct.

So the bug was not in the BYOB reader validation. The bug was that Fetch body extraction was creating the wrong kind of stream.

## The Fetch spec

Fetch's BodyInit extraction algorithm says that for these byte-backed body sources, the stream should be set up with byte reading support.

That matters for:

```text
USVString
URLSearchParams
BufferSource
FormData
byte sequence
```

Those are exactly the body kinds covered by the failing tests.

Blob was not part of this bug. Servo's Blob stream already goes through the Blob get-stream path, and that path already uses byte reading support.

The missing part was the BodyInit extraction path for the non-Blob byte-backed inputs.

## The fix

The patch added a small helper:

```rust
fn stream_from_body_init_bytes(
    cx: &mut js::context::JSContext,
    global: &GlobalScope,
    bytes: Vec<u8>,
) -> Fallible<DomRoot<ReadableStream>> {
    ReadableStream::new_from_bytes_with_byte_reading_support(cx, global, bytes)
}
```

Then the affected extraction paths were changed to use it:

```text
BodyInit::ArrayBuffer
BodyInit::ArrayBufferView
Vec<u8>
DOMString
FormData
URLSearchParams
```

Before the fix, those paths created default streams:

```rust
ReadableStream::new_from_bytes(cx, global, bytes)
```

After the fix, they create byte streams:

```rust
stream_from_body_init_bytes(cx, global, bytes)
```

That changes the test path to:

```text
new Response("This is response's body")
-> BodyInit::String extraction
-> string bytes are created
-> Servo creates the body stream with byte reading support
-> response.body is a readable byte stream
-> getReader({ mode: "byob" }) succeeds
-> the caller's buffer is filled
```

The important part is that this does not weaken Streams.

Streams still rejects BYOB readers for non-byte streams. Fetch now gives Streams the right kind of stream for these response bodies.

## The test metadata

Once the implementation matched Fetch, these expected-fail entries were stale:

```text
Read text response's body as readableStream with mode=byob
Read URLSearchParams response's body as readableStream with mode=byob
Read array buffer response's body as readableStream with mode=byob
Read form data response's body as readableStream with mode=byob
Reading with offset from Response stream
```

They were removed for both window and worker runs in:

```text
tests/wpt/meta/fetch/api/response/response-consume-stream.any.js.ini
```

## The result

The focused WPT run for `response-consume-stream.any.js` now has the BYOB cases passing in window and worker.

The shape of the fix is simple:

```text
BodyInit bytes should produce a readable byte stream.
Readable byte streams can have BYOB readers.
Default streams still cannot.
```

The patch is small because Servo already had most of the stream machinery. The main work was finding the place where Fetch was using the default-stream helper instead of the byte-stream helper.
