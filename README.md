# CleanupSome Method for WeakRefs

# Introduction

The `CleanupSome` method was originally part of the TC39 Proposal for [`WeakRefs`](https://github.com/tc39/proposal-weakrefs). This method provided a way for library authors to express "back pressure" to the engine, and allow them to clean up memory in performance sensitive applications. Due to a few questions around this API, it is proposed to move this into it's own proposal.

# Motivation

In [Issue 16](https://github.com/tc39/proposal-weakrefs/issues/16) of the weakref proposal, Lars describes a use case in wasm for weak references: objects in JavaScript should act as "remote references" for elements implemented and managed in wasm. This reference handling shares a lot with the RemoteConnection scenario, where the remote service happens to be in a local wasm module.

A [separate document](https://github.com/tc39/proposal-weakrefs/wiki/WeakRefGroups) proposes some enhancements for the API to support manual finalization. There is an important issue not addressed there: when the target is pulled from a WeakRef (using `deref()`), the WeakRef is strong until the end of the JavaScript turn. This precludes a class of subtle bugs and substantially reduces observable non-determinism in GC activity. (See the "Strong Deref Example" below for a detailed example.)

However, in some wasm scenarios, execution is driven from wasm and not from the JavaScript event loop, so the turn doesn't change. Those scenarios require that the finalization operations be able to be driven from wasm, without requiring the JavaScript.

This proposal addresses the "turn counter" issue and allows wasm-driven manual finalization to work correctly, while preserving the turn-based correctness features.

# Approach

The "strong deref" semantics are to ensure that JavaScript code that makes a decision based on the value in a WeakRef can rely on that decision later in the turn. That property can be trivially achieved for wasm as long as there are no JavaScript stack frames on the stack. This requires two operations:

* `AdvanceJobReference()` - Increment the turn counter if and only if there are no JavaScript frames on the stack. This is invoked by the JavaScript runtime between turns, or by a wasm program.
* `ScheduleWasmJob(...)` - This will schedule a job on the JavaScript job queue to enter wasm at an appropriate entry point directly from the base JavaScript runtime. As a result, the wasm stack will have no JavaScript stack frames on it, and so can invoke `AdvanceJobReference()` whenever it needs to.

Within the wasm job, WeakRefs can be dereferenced. Each dereferenced WeakRef will act as a strong reference until `AdvanceJobReference()` is called, at which point it will be weak again. GC may happen at any time. The additional primitive for iterating finalizabe WeakRefs in [WeakRefGroups](https://github.com/tc39/proposal-weakrefs/wiki/WeakRefGroups) can be invoked at any time.

# Strong Deref Example

This just has a pseudo-code example of one of the less subtle bugs that could arise in finalization systems without the strong-deref semantics.  The `sendRef` operation is sending a mention of a particular remote ID on a connection. It creates a remote reference if we don't already have it, then sends it, then increments the count of how many times it has been sent.
```js
sendRef(remoteId) {
  if (!hasRemoteRef(remoteId)) {
    this.makeRemoteRef(remoteId);
  }
  ... send ...
  // What if a GC happened here?
  incrementUsageCount(remoteId);
}

// Return a remote reference for a remoteID, if we have it already
// Note that we might have the entry but have already
// garbage collected the weakly pointed-at remote reference
fetchRemoteRef(remoteId) {
  const weakRef = this.remotes[remoteId];
  return weakRef && weakRef.deref();
}
// Return true if we have a particular remote ref
hasRemoteRef(remoteId) {
  return this.fetchRemoteRef(remoteId) !== undefined;
}
// increment the remote usage count in a remote reference
incrementUsageCount(remoteId) {
  // Without strong deref(), this could error
  this.fetchRemoteRef(remoteId).incrementUsage();
}
```

The above code is pretty reasonable, except that if the `hasRemoteRef` test succeeds, but then a GC happens and the remote reference gets collected, the `incrementUsage` call could fail because `fetchRemoteRef` returns `undefined`.  In a more subtle version, the relevant remote reference itself might have been on the stack (e.g., because someone sent it a message), but clever compiler optimization determined that it would never be used again, so it drops the last reference. Then, even though the remote reference appears to be in teh midst of being used, it actually gets garbage collected, and looking it up with `fetchRemoteRef` fails. The strong deref approach eliminates all these bugs, allowing simpler prorgamming (as above) and permitting optimizing compilers to aggresively release no-longer-used variables.

