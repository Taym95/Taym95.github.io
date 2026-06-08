---

layout: post
title: "Fixing a Servo canvas memory leak"
description: "How reporting CanvasRenderingContext2D native memory to SpiderMonkey helped Servo collect canvas objects soon enough to release their resources."
date: 2026-06-07 00:00:00 +0200
series: Servo generational run
categories:
  - servo
  - servo-generational-run
tags:
  - servo
  - canvas
  - memory-leak
  - spidermonkey
  - webrender
issue_url: https://github.com/servo/servo/issues/45369
pr_url: https://github.com/servo/servo/pull/45455
---

This is the first post in my Servo generational run series. I want to use this series to write down small debugging stories from Servo work, mostly the kind of issues where the final patch is small, but understanding the problem takes some digging.

This one is about [Servo issue #45369](https://github.com/servo/servo/issues/45369), a 2D canvas memory leak report fixed in [Servo PR #45455](https://github.com/servo/servo/pull/45455).

The report used the `candle-repeat-var.html` testcase from [issue #45199](https://github.com/servo/servo/issues/45199). In Firefox and Chromium, memory eventually stabilized around 900 MB. In Servo nightly, the same testcase kept growing and reached around 5 GB after eight minutes.

<figure class="post-figure">
  <img src="{{ '/assets/before.png' | relative_url }}" alt="Memory usage before the Servo canvas memory leak fix">
  <figcaption>Before the fix, memory kept growing while the canvas testcase ran.</figcaption>
</figure>

<figure class="post-figure">
  <img src="{{ '/assets/after.png' | relative_url }}" alt="Memory usage after the Servo canvas memory leak fix">
  <figcaption>After the fix, SpiderMonkey collected old canvas contexts sooner and memory stabilized.</figcaption>
</figure>

At first, this looked like a canvas resource not being released. But the real issue was slightly different: the release path existed, it just was not being reached soon enough.

The important ownership chain looked like this:

```text
JS object / SpiderMonkey reflector
-> CanvasRenderingContext2D Rust object
-> CanvasState field
-> canvas paint-thread resource
-> WebRender ImageKey
```

`CanvasState` is a Rust field inside `CanvasRenderingContext2D`. So `CanvasState::drop` only runs when the whole `CanvasRenderingContext2D` object is destroyed.

And that depends on SpiderMonkey finalizing the JS DOM object that owns it.

## Before the fix

Before the fix, SpiderMonkey mostly saw this:

```text
CanvasRenderingContext2D JS object: small object
```

But keeping that small JS object alive also kept native canvas resources alive:

```text
small JS object
+ 10-20 MB native canvas memory
```

SpiderMonkey did not know about that native memory. From the GC's point of view, these objects were cheap, so there was not enough pressure to collect them quickly.

The result was:

```text
old canvas becomes unreachable
-> GC does not run soon enough
-> CanvasRenderingContext2D is not finalized
-> CanvasState::drop is not called
-> CanvasMsg::Close is not sent
-> CanvasData and ImageKey stay alive
-> memory keeps growing
```

So the problem was not that `Drop` was missing. The problem was that Servo relied on GC finalization to reach the Rust `Drop` path, and the GC did not know these objects were expensive.

## The fix

The fix was to report the native memory held by the canvas context:

```rust
self.reflector_.update_memory_size(self, calculated_size);
```

That tells SpiderMonkey something closer to the real cost of the object:

```text
This JS object is small,
but keeping it alive also keeps about 22 MB of native memory alive.
```

Now, when many old canvas contexts accumulate, SpiderMonkey sees the extra memory pressure and runs GC sooner.

Once GC collects the unreachable `CanvasRenderingContext2D`, Servo's normal cleanup path can run:

```text
GC sees old CanvasRenderingContext2D is unreachable
-> GC finalizes its JS reflector
-> Servo destroys the Rust CanvasRenderingContext2D
-> Rust drops its fields
-> CanvasState::drop runs
-> CanvasMsg::Close is sent
-> CanvasPaintThread removes CanvasData
-> CanvasData::drop deletes ImageKey
-> native memory is released
```

The important detail is that `update_memory_size` does not free anything by itself. It only tells SpiderMonkey how much native memory is tied to the JS object.

The actual cleanup still happens through Servo's normal Rust `Drop` path.

So the fix was not:

```text
manually free canvas memory
```

It was:

```text
make the GC aware of the native memory,
so it collects the owner object soon enough
```

Before the fix, the GC treated the canvas context like a small JS object. After the fix, it also accounted for the native canvas resources behind it. That was enough to make collection happen earlier and stop the memory growth.
