# CleanupSome Method for WeakRefs

# Introduction

The `CleanupSome` method was originally part of the TC39 Proposal for [`WeakRefs`](https://github.com/tc39/proposal-weakrefs). This method provided a way for library authors to express "back pressure" to the engine, and allow them to clean up memory in performance sensitive applications. Due to a few questions around this API, it is proposed to move this into it's own proposal.

[Historical document - Support for long wasm jobs](https://github.com/tc39/proposal-weakrefs/wiki/Support-for-long-wasm-jobs)
