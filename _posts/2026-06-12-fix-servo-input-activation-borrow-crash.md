---

layout: post
title: "Fixing a Servo input activation borrow crash"
description: "How a Facebook crash led to a RefCell borrow bug in checkbox activation, and why the fix was to stop holding input_type across script-visible events."
date: 2026-06-12 00:00:00 +0200
series: Servo generational run
categories:
  - servo
  - servo-generational-run
tags:
  - servo
  - html
  - input
  - activation
  - refcell
  - facebook
issue_url: https://github.com/servo/servo/issues/45585
pr_url: https://github.com/servo/servo/pull/45619
---

This is the third post in my Servo generational run series. This one is about a small crash fix in Servo's HTML input activation code.

The report was [Servo issue #45585](https://github.com/servo/servo/issues/45585): Facebook crashes Servo.

The fix landed in [Servo PR #45619](https://github.com/servo/servo/pull/45619).

The issue itself did not have a reduced testcase. The useful part was a screenshot with a Servo panic in the terminal. The panic was:

```text
RefCell already borrowed
```

and it pointed into `HTMLInputElement::attribute_mutated`, in the code that handles a change to the `type` attribute:

```rust
*self.input_type.borrow_mut() = InputType::new_from_atom(attr.value().as_atom());
```

That line needs a mutable borrow of `self.input_type`. If it panics with `RefCell already borrowed`, then something else still has an immutable borrow of the same `input_type` alive.

So the question became: what code was holding `self.input_type()` while Facebook changed `input.type`?

## Reproducing the crash

The browser page was Facebook, but the bug did not need Facebook to reproduce. The important pattern was much smaller:

```js
const input = document.createElement("input");
input.type = "checkbox";
document.body.append(input);

input.addEventListener("input", () => {
  input.type = "text";
});

input.click();
```

Clicking the checkbox runs the input element activation behavior. For checkbox inputs, the HTML spec says activation fires an `input` event and then a `change` event.

That matters because firing an event means author script can run. In this testcase, the `input` event listener changes the input type from `checkbox` to `text`.

That re-enters Servo's type mutation code while activation is still running.

## Before the fix

The activation dispatch in Servo looked like this:

```rust
self.input_type()
    .as_specific()
    .activation_behavior(cx, self, event, target);
```

At first glance this looks harmless. It borrows the current `InputType`, gets the type-specific behavior, and calls it.

The problem is the lifetime of the borrow.

`self.input_type()` returns a `Ref<InputType>` from a `DomRefCell`. Because the call is chained, that `Ref` stays alive until the whole statement finishes.

For a checkbox, the call path is:

```text
HTMLInputElement::activation_behavior
-> self.input_type() borrow is alive
-> CheckboxInputType::activation_behavior
-> fire input event
-> page JS runs
-> JS sets input.type = "text"
-> HTMLInputElement::attribute_mutated wants borrow_mut
-> RefCell panic
```

The bug was not that changing `input.type` from an event listener is invalid. Script is allowed to do that.

The bug was that Servo kept an internal borrow alive while running code that can call back into script.

## The fix

My first fix was to clone the current type name while the borrow was short-lived, then build a temporary `InputType` from that name and dispatch activation on the temporary value.

That did fix the crash, but it had a bad smell.

The temporary `InputType` was not the real `self.input_type()`. It was a fresh enum value with default internal fields. Today that happened to be okay because the activation methods did not use those fields, but it would be very confusing if a later activation method started using `self` and silently read default state instead of the state from the element.

There was also a Crown wrinkle. Crown did not catch one of the temporary-expression shapes here, so just making Crown quiet was not enough proof that the shape was good.

The version I ended up with splits input activation dispatch from the real `InputType` storage.

There is still the normal `InputType` enum for the actual stateful input type:

```rust
#[must_root]
pub(crate) enum InputType {
    Button(ButtonInputType),
    Checkbox(CheckboxInputType),
    // ...
}
```

But activation now uses a separate enum made only of empty marker types:

```rust
#[derive(Clone, Copy)]
pub(crate) enum InputActivationType {
    Button(ButtonInputActivation),
    Checkbox(CheckboxInputActivation),
    Color(ColorInputActivation),
    File(FileInputActivation),
    Image(ImageInputActivation),
    Radio(RadioInputActivation),
    Reset(ResetInputActivation),
    Submit(SubmitInputActivation),
}
```

Those marker types implement a separate trait:

```rust
pub(crate) trait SpecificInputActivationType {
    fn activation_behavior(
        &self,
        _cx: &mut js::context::JSContext,
        _input: &HTMLInputElement,
        _event: &Event,
        _target: &EventTarget,
    ) {
    }
}
```

For checkbox and radio, the activation bodies still live in the checkbox and radio files. The important change is that the receiver is now `CheckboxInputActivation` or `RadioInputActivation`, not the stateful `CheckboxInputType` or `RadioInputType`.

Then `HTMLInputElement::activation_behavior` only borrows the real `input_type` long enough to choose the marker:

```rust
let input_activation_type = {
    let input_type = self.input_type();
    InputActivationType::new_from_input_type(&input_type)
};

if let Some(input_activation_type) = input_activation_type {
    input_activation_type
        .as_specific()
        .activation_behavior(cx, self, event, target);
}
```

After the block ends, the `Ref<InputType>` is gone. The activation behavior can fire `input` and `change` events, and author script can change `input.type`, without colliding with the old immutable borrow.

This also avoids the "fake `InputType`" problem. Activation methods no longer receive a made-up checkbox or radio object with default fields. They receive an empty marker, and the actual element is passed separately as `input`.

So the useful invariant is clearer:

```text
real InputType: owns state and must be rooted
InputActivationType: copyable marker used only to choose activation behavior
```

It is a little more code than the first patch, but the split is honest. If an activation method needs element state, it has to get it from `input`, not from a pretend copy of the stored input type.

## The regression test

The regression test lives in:

```text
tests/wpt/tests/html/semantics/forms/the-input-element/input-type-change-during-activation-crash.html
```

It creates a checkbox, changes its type from an `input` event listener, and clicks it.

Without the fix, Servo crashes with the `RefCell already borrowed` panic. With the fix, the page loads and the click finishes.

This is a crashtest, so it does not use `testharness.js`. The pass condition is simply that Servo does not crash while loading the page.

One small WPT detail: adding a new test file is not enough by itself. Servo's WPT runner discovers tests through the WPT manifest, so the manifest has to be updated too:

```text
./mach update-manifest
```

After that, the focused test can be run with:

```text
./mach test-wpt tests/wpt/tests/html/semantics/forms/the-input-element/input-type-change-during-activation-crash.html
```

## The result

The focused WPT now passes:

```text
TEST_END: PASS
Unexpected results: 0
```

The final lesson is simple, but easy to miss in DOM code:

```text
Do not hold RefCell borrows across event dispatch.
```

Event dispatch can run author script. Author script can mutate the DOM. If the mutation path needs the same `RefCell`, a borrow that looked local can become a crash.

In this case, the behavior was already correct. Servo just needed a dispatch shape that did not hold the internal `input_type` borrow while running the spec-required activation events.
