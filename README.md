# The `FinalizationRegistry.prototype.cleanupSome` Method

Champion group: Daniel Ehrenberg, Yulia Startsev

Stage 2

# Introduction

The `FinalizationRegistry.prototype.cleanupSome` method was originally part of the TC39 Proposal for [`WeakRefs`](https://github.com/tc39/proposal-weakrefs). This method provided a way for library authors to synchronously allow them to clean up without yielding to the event loop. We propose to move it into a separate proposal, to allow the core of WeakRefs to be used in the wild a bit before adding this additional functionality.

## Motivation of cleanupSome

WeakRef and FinalizationRegistry allow JavaScript programs to observe garbage collection. This observability presents significant interoperability risks, which are contained somewhat by giving the host environment control over the granularity in time of when JS can see that an object has been garbage-collected.

In HTML, the granularity is based on tasks and microtasks: WeakRefs are only allowed to observably go from "filled with an object" to undefined at the end of a microtask checkpoint (i.e., after all the Promise jobs run), and FinalizationRegistry cleanup callbacks happen in a queued task, meaning also only when all the Promise jobs have run. This all adds up to: you need to yield to the event loop regularly to make WeakRef and FinalizationRegistry work.

With Workers, SharedArrayBuffer and/or WebAssembly threads/shared memory, it's possible to create a "long job" which does a bunch of computation and communicates with other agents through shared memory and atomics, without yielding to the event loop very frequently. WebAssembly facilitates this use case. If there is also use of JavaScript garbage-collected objects, there may be demand to get FinalizationRegistry cleanup callbacks *without* yielding to the event loop.

This use case is met by `FinalizationRegistry.prototype.cleanupSome`. It accepts a function as a parameter (or can fall back to the `FinalizationRegistry`'s callback), and may call that callback with the `heldValue` of any registered, garbage-collected value, to synchronously perform cleanup actions.

## Hesitation: When should `cleanupSome` be exposed on the Web?

The "long job" case doesn't quite make sense on the "window", the Web's main thread. It makes more sense for background Worker/Worklet threads. So, in browsers, it may make sense to exclude it from the window. There are several other Web Platform APIs which are only exposed in the context of certain global objects and not others, so this would follow a typical idiom.

Further, Apple has raised concerns about `cleanupSome` being exposed on the Web at all, due to concerns about whether we want to encourage the "long job" programming style in a context where objects are also being used.

Discussion about where and whether `cleanupSome` should be exposed in web browsers is ongoing in [a WHATWG HTML issue](https://github.com/whatwg/html/issues/5446).

Splitting out `cleanupSome` into a separate proposal would give everyone time to consider its presence in browsers at its own pace.

## Design changes over time

Earlier drafts of `cleanupSome` had certain differences from the latest version:
- Initially, `cleanupSome` returned a boolean to indicate whether anything is cleaned up. Now, `cleanupSome` always returns `undefined`.
- Initially, `cleanupSome` called its callback with an iterator of held values. The JavaScript code had the option of not exhausting the iterator, leaving it to the engine to requeue the parts which were not used. Now, `cleanupSome` calls the callback  with one value at a time, repeatedly.

These changes were made without the involvement of the people who made the previous design decisions. Splitting out `cleanupSome` into a separate proposal would give the committee time to reconsider these design decisions.

## Ecosystem impact of a proposal split vs normative optional

Declaring `FinalizationRegistry.prototype.cleanupSome` has been a bit confusing from an ecosystem perspective. For example,
- The [TypeScript typing](https://github.com/microsoft/TypeScript/pull/38232) includes `cleanupSome` but marked as optional
- The [MDN documentation](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/FinalizationRegistry/cleanupSome) notes that the method is "optional", but it's not really clear how this should be interpreted by readers.

Splitting `cleanupSome` into a separate proposal would make it clear to the JavaScript community that the main parts of `WeakRef` are standard, and this part is still under discussion.

## Committee status

`FinalizationRegistry.prototype.cleanupSome` has been off from the WeakRefs proposal and is at Stage 2.

## History

This API was originally proposed in a form which would be WebAssembly-specific. See the [historical document - Support for long wasm jobs](https://github.com/tc39/proposal-weakrefs/wiki/Support-for-long-wasm-jobs) for more details.
