---
 title: 'How we made JSON.stringify more than twice as fast'
 author: 'Patrick Thier'
 avatars:
  - 'patrick-thier'
 date: 2025-08-04
 tags:
   - internals
 description: "This post explains our recent effort to improve JSON.stringify performance"
---

`JSON.stringify` is a core JavaScript function for serializing data. Its performance directly affects common operations across the web, from serializing data for a network request to saving data to `localStorage`. A faster `JSON.stringify` translates to quicker page interactions and more responsive applications. That’s why we’re excited to share that a recent engineering effort has made `JSON.stringify` in V8 **more than twice as fast**. This post breaks down the technical optimizations that made this improvement possible.

# A Side-Effect-Free Fast Path

The foundation of this optimization is a new fast path built on a simple premise: if we can guarantee that serializing an object will not trigger any side effects, we can use a much faster, specialized implementation. A "side effect" in this context is anything that breaks the simple, streamlined traversal of an object.  
This includes not only the obvious cases like executing user-defined code during serialization, but also more subtle internal operations that might trigger a garbage collection cycle. For more details on what exactly can cause side effects and how you can avoid them, see [Limitations](#limitations).

As long as V8 can determine that serialization will be free from these effects, it can stay on this highly-optimized path. This allows it to bypass many expensive checks and defensive logic required by the general-purpose serializer, resulting in a significant speedup for the most common types of JavaScript objects that represent plain data.

Furthermore, the new fast path is iterative, in contrast to the recursive general-purpose serializer. This architectural choice not only eliminates the need for stack overflow checks and allows us to quickly resume after [encoding changes](#handling-different-string-representations), but also allows developers to serialize significantly deeper nested object graphs than was previously possible.

# Handling different String Representations

Strings in V8 can be represented with either one-byte or two-byte characters. If a string contains only ASCII characters, they are stored as a one-byte string in V8 that uses 1 byte per character. However if a string contains just a single character outside of the ASCII range, all characters of the string use a 2 byte representation, essentially doubling the memory utilization.

To avoid the constant branching and type checks of a unified implementation, the entire stringifier is now templatized on the character type. This means we compile two distinct, specialized versions of the serializer: one completely optimized for one-byte strings and another for two-byte strings. This has an impact on binary size, but we think the increased performance is definitely worth it.

The implementation handles mixed encodings efficiently. During serialization, we must already inspect each string's instance type to detect representations we can’t handle on the fast path (like [`ConsString`](https://crsrc.org/c/v8/src/objects/string.h;drc=9768251f3e8f598d82420259a940d2057ed56b42;l=1013), which might trigger a GC during flattening) that require a fallback to the slow path. This necessary check also reveals whether a string uses one-byte or two-byte encoding.

Because of this, the decision to switch from our optimistic one-byte stringifier to the two-byte version is essentially free. When this existing check reveals a two-byte string, a new two-byte stringifier is created, inheriting the current state. At the end, the final result is constructed by simply concatenating the output from the initial one-byte stringifier with the output from the two-byte one. This strategy ensures we stay on a highly-optimized path for the common case, while the transition to handling two-byte characters is lightweight and efficient.

# Optimizing String Serialization with SIMD

Any string in JavaScript can contain characters that require escaping when serializing to JSON (e.g. `"` or `\`). A traditional character-by-character loop to find them is slow.

To accelerate this, we employ a two-level strategy based on the string's length:

- For longer strings, we switch to dedicated hardware SIMD instructions (e.g., ARM64 Neon). This allows us to load a much larger chunk of the string into a wide SIMD register and check multiple bytes for any escapable characters at once in just a few instructions. ([source](https://crsrc.org/c/v8/src/json/json-stringifier.cc;drc=1645281bbd1b183a252835d376166bd210135bbe;l=3369))  
- For shorter strings, where the setup cost of hardware instructions would be too high, we use a technique called SWAR (SIMD Within A Register). This approach uses clever bitwise logic on standard general-purpose registers to process multiple characters at once with very low overhead. ([source](https://crsrc.org/c/v8/src/json/json-stringifier.cc;drc=1645281bbd1b183a252835d376166bd210135bbe;l=3353))

Regardless of the method, the process is highly efficient: we rapidly scan through the string chunk by chunk. If no chunk contains any special characters (the common case), we can simply copy the whole string.

# The Express Lane on the Fast Path

Even within the main fast path, we found an opportunity for another, even faster 'express lane'. By default, the fast path must still iterate over an object's properties and, for each key, perform a series of checks: confirm the key is not a `Symbol`, ensure it's enumerable, and finally, scan the string for characters that require escaping (e.g. `"` or `\`).

To eliminate this, we introduce a flag on an object's [hidden class](https://v8.dev/docs/hidden-classes). Once we have serialized all properties of an object, we mark its hidden class as fast-json-iterable if no property key is a `Symbol`, all properties are enumerable, and no property key contains characters that require escaping.

When we serialize an object that has the same hidden class as an object we serialized before (which is quite common, e.g. an array of objects which all have the same shape) and it is fast-json-iterable, we can simply copy all the keys to the string buffer without any further checks.

We also added this optimization to `JSON.parse`, where we can utilize it for fast key comparisons while parsing an array, assuming that objects in the array often have the same hidden classes.

# A faster double-to-string algorithm

Converting numbers to their string representation is a surprisingly complex and performance-critical task. As part of our work on `JSON.stringify`, we identified an opportunity to significantly speed up this process by upgrading our core `DoubleToString` algorithm. We have now replaced the long-serving Grisu3 algorithm with [Dragonbox](https://github.com/jk-jeon/dragonbox) for shortest length number to string conversions.

While this optimization was driven by our `JSON.stringify` profiling, the new Dragonbox implementation benefits **all** calls to `Number.prototype.toString()` throughout V8. This means any code that converts numbers to strings, not just JSON serialization, will see this performance boost for free.

# Optimizing the underlying temporary buffer

A significant source of overhead in any string-building operation is how memory is managed. Previously, our stringifier built the output in a single, contiguous buffer on the C++ heap. While simple, this approach has a significant drawback: whenever the buffer ran out of space, we had to allocate a larger one and copy the entire existing content over. For large JSON objects, this cycle of re-allocation and copying created major performance overhead.

The crucial insight was that forcing this temporary buffer to be contiguous offered no real benefit, as the final result is assembled into a single string only at the very end.

With this in mind, we replaced the old system with a segmented buffer. Instead of one large, growing block of memory, we now use a list of smaller buffers (or "segments"), allocated in V8's Zone memory. When a segment is full, we simply allocate a new one and continue writing there, completely eliminating the expensive copy operations.

# Limitations

The new fast path achieves its speed by specializing for common, simple cases. If the data being serialized doesn't meet these criteria, V8 falls back to the general-purpose serializer to ensure correctness. To get the full performance benefit, the `JSON.stringify` call must adhere to the following conditions.

- No `replacer` or `space` arguments: Providing a `replacer` function or a `space`/`gap` argument for pretty-printing are features handled exclusively by the general-purpose path. The fast path is designed for compact, non-transformed serialization.  
- Plain data objects and arrays: The objects being serialized should be simple data containers. This means they, and their prototypes, must not have a custom `.toJSON()` method. The fast path assumes standard prototypes (like `Object.prototype` or `Array.prototype`) that don't have custom serialization logic.  
- No indexed properties on objects: The fast path is optimized for objects with regular, string-based keys. If an object contains array-like indexed properties (e.g., `'0', '1', ...`), it will be handled by the slower, more general serializer.  
- Simple string types: Some internal V8 string representations (like `ConsString`) can require memory allocation to be flattened before they can be serialized. The fast path avoids any operation that might trigger such allocations and works best with simple, sequential strings. This is something that’s hard to influence as a web developer. But don’t worry, it should just work in most cases.

For the vast majority of use cases, such as serializing data for API responses or caching configuration objects, these conditions are naturally met, allowing developers to benefit from the performance improvements automatically.

# Conclusion

By rethinking `JSON.stringify` from the ground up, from its high-level logic down to its core memory and character-handling operations, we've delivered a more than 2x performance improvement measured on the JetStream2 json-stringify-inspector benchmark. See the figure below for results on different platforms. These optimizations are available in V8 starting with version 13.8 (Chrome 138).  
![JetStream2 Results](/_img/json-stringify/results-jetstream2.svg)
