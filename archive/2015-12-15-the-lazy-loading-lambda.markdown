---
title:  "The Lazy-Loading Lambda"
date:   2015-12-15 21:36:20 +0000
---
We've all seen Ruby code like this:

```ruby
def expensive_thing
  @expensive_thing ||= ExpensiveThing.new
end
```

Where `ExpensiveThing` is instantiated only if the `expensive_thing` method is called. The resultant object is stored in an instance variable, a neat way of ensuring that all subsequent calls to `expensive_thing` do not re-create the object.

It then follows that this...

```ruby
def expensive_thing
  @expensive_thing ||= begin
    ExpensiveThing.new
  end
end
```

...is valid Ruby code. The newly birthed `ExpensiveThing` is returned as it is the value of the block.

But what if you need to somehow molest or mutate `ExpensiveThing`?

```
def expensive_thing
  @expensive_thing ||= begin
    e = ExpensiveThing.new
    e.expensive_update
    return e 
  end
end
```

This would fail with the esoteric `void value expression`. The Ruby compiler is upset that it doesn't get the chance to fully complete the block, and therefore compute its return value, so throws its toys out of the proverbial pram. 

An easy way to avoid this tantrum is to use a lambda block:

```
def expensive_thing
  @expensive_thing ||= lambda { 
      e = ExpensiveThing.new
      e.expensive_update
      return e 
    }.call
end
```
