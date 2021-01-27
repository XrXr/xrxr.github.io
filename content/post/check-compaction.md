---
title: "Checking Ruby C extensions for object movement crashes"
date: 2021-01-26T19:58:24-05:00
draft: true
---

This guide intends to help Ruby native extension maintainers upgrade libraries to be compatible with `GC.compact`. Application developers can also use this guide to check applications for compaction compatibility. At the time of writing, the latest Ruby release is version 3.0.0.

### Using automated tests to surface crashes

If your test suite runs under Ruby 2.7.0 or newer, it is possible to check for compaction crashes with a small addition to your test suite.

Add the following line such that it runs after all the code in the test suite finishes loading, but before any tests run. For libraries, this usually means inserting after `require "library_name"` during test setup.

```
GC.verify_compaction_references(double_heap: true, toward: :empty)
```

Look for crash ouputs similar to that of `ruby -e 'Process.kill(:SEGV, $$)'` and unexpected `TypeErorr`s. APIs such as `rb_raise` can raise `TypeError` when given invalid references.

### One common mistake

A common pitfall that causes object movement bugs is saving objects created with Ruby code into global variables. This is usually done with `rb_const_get` or similar in the extension's `Init_library_name` routine. Often the object saved into the C global is a class or a module defined in Ruby code.

The GC can decide to move the object the C global refers to, invalidating the `VALUE`. The extenion is likely to trigger a crash when it makes use of stale references.

Extensions can solve this problem by calling `rb_gc_register_mark_object` on objects created in Ruby that are saved into C globals. This API tells the GC to not move specific objects. It is worth noting that this API should be used sparingly, as limiting object movement makes compaction less effective. Also, the GC never collects objects passed to this API, so misuse can create memory leaks.

The following C APIs create modules that never move. It is not necessary to use `rb_gc_register_mark_object` on objects created with these APIs:
  - `rb_define_class`
  - `rb_define_module`
  - `rb_define_class_under`
  - `rb_define_module_under`

As an alternative to saving references into globals, extensions can fetch constants at the time they are needed using APIs such as `rb_const_get`.

### Bug exists even in absence of compaction

Extensions that follow the pattern above can cause crashes in Ruby releases that never move objects. The object saved into a constant in Ruby code can be removed from that constant via means such as `Module#remove_const` and be collected by the GC, invalidating the `VALUE` stored in the C global. See this [issue](https://github.com/msgpack/msgpack-ruby/issues/133) for an example of this happening in a popular gem.

### Examples

Here is a demo that contains the discussed failure pattern. For an exercise, try to fix the object movement bug.

```ruby
#!/bin/env ruby
# frozen_string_literal: true
# This demo contains an object movement bug.
# Run in an empty directory.

# Write out code for C extension
File.write("ext.c", <<-EOM)
#include "ruby.h"

static VALUE cLuckError;

VALUE
luck_trial(VALUE self)
{
    rb_raise(cLuckError, "insufficient luck");
}

void
Init_bad(void)
{
    cLuckError = rb_const_get(rb_cObject, rb_intern("LuckError"));
    rb_define_global_function("luck_trial", luck_trial, 0);
}
EOM

class LuckError < StandardError
end

# Compile the C extension
Process.spawn(Gem.ruby, '-rmkmf', '-e', 'create_makefile("bad")')
Process.wait
`make clean`
`make`

# Load the C extension. Initialization runs.
require_relative 'bad'

if defined?(GC.verify_compaction_references) == 'method'
  # Ask the GC to move objects around
  GC.verify_compaction_references(double_heap: true, toward: :empty)
else
  # Trigger the bug without doing any object movement by making
  # LuckError unreachable. Compatible with Ruby 2.5.x.
  LuckError = nil
  4.times { GC.start }
end

# Use the extension
begin
  luck_trial
  puts "success"
rescue => e
  puts "#{e.class}: #{e}"
end
```

For real-world references, here are a few pull requests that fix object movement bugs in popular gems:
 - https://github.com/brianmario/mysql2/pull/1115
 - https://github.com/Shopify/semian/pull/275
 - https://github.com/ohler55/oj/pull/638

### Closing thoughts

The GC's abaility to compact the heap allows for memory savings and can improve execution performance. It is key to the runtime's evolution. If you are a library maintainer, thank you for enabling people to use compaction!

*Special thanks to [Aaron Paterson](https://twitter.com/tenderlove/) for helping with this guide and for developing the compacting GC*
