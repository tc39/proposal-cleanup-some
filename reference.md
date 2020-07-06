# Introduction

### `FinalizationRegistry.prototype.cleanupSome`

This **optional** method triggers callbacks for an implementation-chosen number of objects in the registry that have been reclaimed but whose callbacks have not yet been called:

```js
registry.cleanupSome?.(heldValue => {
    // ...
});
```

This method is optional, any given implementation may not have it. It's expected not to be present on web browsers, at least not initially; see [HTML issue #5446](https://github.com/whatwg/html/issues/5446) for details. Since the method is optional, you need to check that it's there before calling it. One way to do that is to use [optional chaining](https://github.com/tc39/proposal-optional-chaining) (`?.`) as in the example above.

Normally, you don't call this function. Leave it to the JavaScript engine's garbage collector to do the callbacks as appropriate. This function primarily exists to support long-running code which doesn't yield to the event loop, which is more likely to come up in WebAssembly than ordinary JavaScript code. Also note that the callback may not be called (for instance, if there are no registry entries whose targets have been reclaimed).

**Parameters**

* `callback` - (Optional) If given, the registry uses the given callback instead of the one it was created with. It must be callable.

**Returns**

* *(none)*

**Notes**

* The number of entries for reclaimed objects that are cleaned up from the registry (calling the cleanup callbacks) is implementation-defined. An implementation might remove just one eligible entry, or all eligible entries, or somewhere in between.


