---
title: "No Ephemerons in Ruby's WeakMap"
date: 2025-02-09T21:30:00-05:00
---

Let's say you want to associate a property with an object, but you don't want
to write to the object directly, maybe for design reasons or logistical reasons
(what if the object is frozen and immutable?) You can use `WeakMap` for this,
and the garbage collector is free to collect keys of the map:

```ruby
require "objspace"

object = []
map = ObjectSpace::WeakMap.new
map[object] = [object]

p map.member?(object) # => true
```

There is an issue with this demo, however. Currently, Ruby's `WeakMap`
doesn't keep the values of the map alive, so the map evicts the pair in the
demo after a garbage collection run:

```ruby
require "objspace"

object = []
map = ObjectSpace::WeakMap.new
map[object] = [object]

p map.member?(object) # => true
GC.start
p map.member?(object) # => false, as of Ruby 3.4
```

So, you can have the property, but it may go away when the collector decides to
run. A bit unruly, but alright for some situations.

ECMAScript also offers `WeakMap`, and they come with ephemerons. I like them
because they sound like mystical creatures. Practically, they deliver a
reliable way to achieve my contrived stated purpose. Nothing in the word "weak"
and "map" indicate that ephemerons are involved, but the spec links to the 1997
paper by Barry Hayes that introduced the mechanism.

Here's a program you can run in your browser to get a taste of their semantics:

```js
// Make an object, and log when it becomes garbage
const registry = new FinalizationRegistry((value) => {
  console.log(value, "is garbage")
})
let object = []
registry.register(object, "object")

// Put the object into a WeakMap and have the value reference itself
let map = new WeakMap
map.set(object, object)

// Stop referring to the object through the local.
// FireFox collects after a few seconds.
object = undefined
```

From my admittedly imperfect understanding, they offer a sort of one-way
conditional liveness, where in a pair the value is live if the key is. Further,
values kept alive solely due to this mechanism cannot "upgrade" their liveness
to stay alive forever by way of reference cycles. Implementing ephemerons in
practice seems to require [discovering them to a fixed-point][1] after the
usual object tracing -- their special kind of liveness seems to call for
tracing under a different mode.

[1]: https://blog.mozilla.org/sfink/2022/06/09/ephemeron-tables-aka-javascript-weakmaps/
