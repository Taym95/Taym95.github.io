---
layout: post
title: "Enabling SharedWorker in Servo"
description: "How a series of PRs brought SharedWorker support to Servo, fixed reuse and stability issues, and moved more WPT coverage to passing."
date: 2026-06-21 00:00:00 +0200
series: Servo generational run
categories:
  - servo
  - servo-generational-run
tags:
  - servo
  - sharedworker
  - workers
  - streams
  - wpt
pr_url: https://github.com/servo/servo/pull/45786
---

This is the fourth post in my Servo generational run series. This one is about finishing SharedWorker support in Servo.

The work did not land as one big patch. It came together through a set of smaller PRs, starting with worker refactoring and ending with SharedWorker being enabled and the WPT metadata being updated.

The final PR was [Servo PR #45786](https://github.com/servo/servo/pull/45786).

That PR enabled SharedWorker support, fixed the last intermittent failures I was seeing, and updated the tests that now pass with SharedWorker available.

## Why SharedWorker is different

Servo already had dedicated workers. A dedicated worker belongs to one page. The page creates it, talks to it, and owns that worker connection.

SharedWorker has a different shape.

A SharedWorker can be reused by multiple same-origin contexts. If two pages create a worker with matching options, they should connect to the same worker instead of getting two separate worker globals.

Each connecting page gets its own `MessagePort`. Inside the worker, the connection is delivered as a `connect` event:

```js
onconnect = event => {
  const port = event.ports[0];
};
```

That means the constructor is not just "start a worker and return it". It has to create a port for the page, create or reuse a `SharedWorkerGlobalScope`, create the inside port for the worker, entangle both ports, and then deliver the `connect` event.

The hard part is the "or reuse" part.

## The PR series

The work landed in these pieces:

- [#44244](https://github.com/servo/servo/pull/44244): generalized common worker code so it was not tied only to dedicated worker globals.
- [#44375](https://github.com/servo/servo/pull/44375): added the `SharedWorker` and `SharedWorkerGlobalScope` WebIDL interfaces.
- [#44440](https://github.com/servo/servo/pull/44440): added the worker and global plumbing needed by SharedWorker.
- [#44761](https://github.com/servo/servo/pull/44761): implemented the constructor, port wiring, worker startup, and `connect` event delivery.
- [#45088](https://github.com/servo/servo/pull/45088): implemented SharedWorker reuse and fixed the duplicate-creation race.
- [#45741](https://github.com/servo/servo/pull/45741): fixed partitioned reuse.
- [#45786](https://github.com/servo/servo/pull/45786): enabled SharedWorker and updated WPT coverage.

That split made review easier. It also kept the early patches boring. The first PRs mostly made room for a non-dedicated worker global without changing existing dedicated worker behavior.

## First working path

The first useful milestone was getting a new `SharedWorker(...)` constructor to create a worker and deliver a connection.

In simple terms, the constructor flow became:

```text
new SharedWorker(url, options)
-> parse the script URL
-> create the outside MessagePort for the page
-> create or reuse a SharedWorkerGlobalScope
-> create the inside MessagePort for the worker
-> entangle the two ports
-> fire connect inside the worker with the inside port
-> expose the outside port as worker.port
```

At that point a page could do:

```js
const worker = new SharedWorker("worker.js");
worker.port.postMessage("hello");
```

and the worker could receive the connection through `event.ports[0]`.

That was enough to make a chunk of the SharedWorker WPTs pass, but it was still not the complete feature. A SharedWorker that always creates a new worker is not really shared.

## Getting connect right

The `connect` event is the center of the feature.

The page never gets direct access to the worker global. It gets `worker.port`. The worker gets the other side of that channel through the event.

So the implementation had to make sure the event was fired on the `SharedWorkerGlobalScope`, and that the inside port appeared in `event.ports[0]`.

There was also a normal DOM detail here: `onconnect` has to behave like an `EventHandler` attribute. Tests do not only use `addEventListener("connect", ...)`; they also assign `onconnect = ...`. That path had to be wired the same way other event handler attributes are wired.

Once that worked, the first end-to-end path was in place:

```text
page creates SharedWorker
-> page gets outside port
-> worker gets connect event
-> worker reads event.ports[0]
-> both sides can exchange messages
```

## Reuse was the hard part

The next step was making two constructors find the same worker when they are supposed to.

For reuse, the matching key has to include the things that make one SharedWorker distinct from another. In Servo's temporary registry, that meant matching the storage key, constructor URL, name, type, credentials, and secure-context state.

The storage key matters because SharedWorker reuse is not just per URL and name. Partitioning can make two same-origin-looking contexts separate for storage purposes. Those contexts should not accidentally share a worker.

The constructor origin also matters, especially for `data:` URLs. A `data:` worker URL does not behave like a normal same-origin script URL. The constructor origin has to be part of the reuse decision so two constructors do not reuse a worker when the spec says they should not.

The useful mental model is:

```text
same storage key
same constructor URL
same name
same worker type
same credentials mode
same secure-context state
compatible constructor origin
-> reuse the existing SharedWorker
```

If one of those does not match, create a different worker.

## The race

Reuse introduced a race.

Two pages can call the constructor at almost the same time. Both can look in the registry, both can see no existing worker, and both can try to create one.

That gives the wrong result. There should be one shared worker and two connections, not two workers.

The fix was to add explicit registry state:

```text
Creating
Created
```

When the first constructor claims a key, it marks it as `Creating`. If another constructor reaches the same key while creation is still in progress, it waits on a `Condvar`. Once the worker is ready, the state moves to `Created`, the waiting constructor wakes up, and it connects to the worker that was just created.

This is intentionally not the final architecture.

The current registry is still script-side. It fixed the duplicate-creation race and made the tests stable, but it is not where I would want the final SharedWorker manager to live. A proper manager should probably sit higher than the script thread and own the lifetime and lookup story more directly.

For now, the tradeoff was acceptable: keep the temporary registry simple, block only during the rare duplicate creation case, and avoid adding a more complicated pending-connection system that would likely be replaced later anyway.

## Blob and data URLs

Blob and data URLs exposed a few edge cases after the main constructor path was working.

For `data:` URLs, reuse depends on the constructor origin. That was part of the matching work. Without it, Servo could make the wrong reuse decision for workers created from `data:` URLs.

Blob URLs had a different set of failures. Some tests covered relative behavior from blob-backed worker scripts. Others covered dynamic module imports from blob URLs, including the case where the blob URL is revoked soon after import starts.

The final enable PR fixed a blob URL lifetime race in module script fetching. That made the sharedworker dynamic-import blob URL tests pass instead of failing with a module fetch error.

These were not the largest patches, but they were good reminders that worker support is tied into URL parsing, script fetching, module loading, and origin handling. The constructor can be correct and still fail because one of those pieces treats a worker script URL slightly differently than the tests expect.

## Partitioned reuse

Partitioning was another place where "same URL and same name" was not enough.

SharedWorker reuse has to respect the storage key. If two contexts are in different partitioned storage buckets, they should not share a worker just because the script URL and name match.

That matters for privacy, and it also matters for observable behavior. A worker can read and write state through APIs that depend on the storage partition. If Servo reuses the worker across partitioned contexts, the test can observe the wrong state through the shared connection.

The partitioned reuse fix made the shared worker partitioning test pass. It also cleaned up the matching model:

```text
same origin is not enough
same URL and name are not enough
the storage key is part of the worker identity
```

The later cookie-related failures were in the same area. Once the reuse key respected partitioning properly, the partitioned cookie behavior lined up with the tests.

## The last failures

The last PR was not only flipping the preference and deleting stale metadata. It also fixed the remaining SharedWorker failures that showed up once the wider test set was enabled.

A few of them were small Web-facing behavior mismatches.

One test checked the exception type for recursive construction. The implementation was failing, but not with the exception shape the test expected.

Another checked `onconnect` as an event handler attribute. That needed to behave like the rest of the platform event handler attributes, not like a special-case callback.

The partitioned worker reuse and partitioned cookie tests checked that storage keys were actually part of reuse, not just computed somewhere and then ignored.

The blob URL tests checked relative behavior and module import lifetime. Those caught cases where the worker machinery was working, but the script fetching path was still not quite right for SharedWorker.

After those fixes, SharedWorker stopped being intermittent in the test runs I was using. That was the point where enabling the feature made sense.

## The result

SharedWorker is now enabled in Servo.

The final PR updates the WPT metadata for the tests that now pass with SharedWorker available. SharedWorker is also no longer intermittent in the relevant test runs. The failures that were intermittent during the work are fixed rather than papered over in metadata.

One visible result is in the streams area on the WPT dashboard. Some streams tests have SharedWorker variants, so enabling SharedWorker made more of that coverage runnable and passing. On the dashboard, the streams subtest pass rate is now close to 90%.

<figure class="post-figure">
  <img src="{{ '/assets/wpt_dashboard_sharedworker_streams_after.png' | relative_url }}" alt="WPT dashboard showing Servo streams subtests close to 90% passing after SharedWorker work">
  <figcaption>After the SharedWorker work, the streams subtest pass rate is close to 90%.</figcaption>
</figure>

The broader WPT numbers need a bit of care. `servo.org/wpt` and `wpt.fyi` do not count exactly the same set of tests. On `servo.org/wpt`, Servo now passes over 2 million subtests. On `wpt.fyi`, some wasm tests were removed, and encoding changes also skew the raw comparison. If encoding is ignored, this work still moved 4,963 more subtests to passing.

The important result is not one exact dashboard number. It is that the basic SharedWorker model now works:

```text
constructors create ports
matching constructors reuse the worker
new connections fire connect events
partitioned contexts stay separate
the WPT coverage is enabled
```

That is a much better base to build on than keeping the feature disabled.

## Follow-up work

There is still follow-up work to do.

The main architectural cleanup is the registry. The current registry lives on the script side. It works for the current tests, but SharedWorker lookup and lifetime should eventually move into a proper shared worker manager.

That manager should probably live at a higher level than the script thread. SharedWorkers are shared across contexts, so their ownership model should not be too closely tied to one script thread's local bookkeeping.

The owner-set and lifetime behavior can also be improved. A SharedWorker should stay alive while it has relevant owners and go away when it no longer does. The current work gets Servo to a working and tested baseline, but the lifetime model is still an area worth tightening.

For this run, I kept the scope focused: make the constructor work, deliver `connect`, implement reuse, fix the races and partitioning issues, and enable the WPT coverage once the failures were real passes.

That is now done.
