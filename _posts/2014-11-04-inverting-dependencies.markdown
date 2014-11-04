---
layout: post
title:  "Inverting Dependencies"
date:   2014-11-04 09:00:00
---

The “D” in the SOLID principles stands for [Dependency Inversion](http://en.wikipedia.org/wiki/Dependency_inversion_principle) (not Injection). I recently stumbled on such a good example of this principle being applied that I had to blog about it.

## The dependency inversion principle

The principle states the following (quote from Wikipedia):

> A. High-level modules should not depend on low-level modules. Both should depend on abstractions.

> B. Abstractions should not depend on details. Details should depend on abstractions.

The example in this article is about item **B**. Let’s jump straight into the example and everything will be clearer.

## The direct dependency

This example came up when I was giving a shot at a open issue in the awesome [Ruby Object Mapper](http://rom-rb.org/) project.

ROM has the advantage of supporting several different databases and it does so through [adapters](http://en.wikipedia.org/wiki/Adapter_pattern). If, for instance, my application needs to store data in MongoDB, I’d use ROM with the Mongo adapter.

ROM also has an `Adapter` superclass. This class defines the standard interface for adapters and it’s responsible for initialising the database-specific implementations. It does so through the `setup` method. Here’s what this method [used to look like](https://github.com/rom-rb/rom/blob/ab8c4cf0544cab07c92843d5972e2cbb5b91e56d/lib/rom/adapter.rb):

```ruby
class Adapter
  ...
  def self.setup(uri_string)
    uri = Addressable::URI.parse(uri_string)
    adapter =
        case uri.scheme
        when 'sqlite', 'jdbc' then Adapter::Sequel
        when 'memory' then Adapter::Memory
        when 'mongo' then Adapter::Mongo
        else
          raise ArgumentError, "#{uri_string.inspect} uri is not supported"
        end

    adapter.new(uri)
  end
  ...
end
```

The interesting bit for the case in point is the `case` statement. From a dependency inversion perspective, there are two issues with this code:

* `Adapter` is an abstract concept, and it is dependent on low-level implementations. This means every time a new adapter is added or removed, the `Adapter` class has to change.
* The case statement wouldn’t scale well. It might be OK for three adapters, but the issue I was working on would add 19 more branches to what would become one gigantic conditional.

[Piotr Solnica](http://solnic.eu/), ROM’s core developer, cleverly suggested inverting the dependency, and so I did.

## The inverted dependency

Piotr’s idea was to remove the case statement and add a `register` method to the `Adapter` class. Each implementation would call `Adapter.register(self)` when loaded. The adapter classes would also have a `schemes` method that returns a list of schemes supported by the adapter.

With this concept implemented, `Adapter` could look up and initialise the correct implementation based on the scheme. Here’s what the relevant part of the code looks like:

```ruby
# adapter.rb

class Adapter
  @adapters = []

  def self.setup(uri_string)
    uri = Addressable::URI.parse(uri_string)

    unless adapter = self[uri.scheme]
      raise ArgumentError, "#{uri_string.inspect} uri is not supported"
    end

    adapter.new(uri)
  end

  def self.register(adapter)
    @adapters << adapter
  end

  def self.[](scheme)
    @adapters.detect { |adapter| adapter.schemes.include?(scheme.to_sym) }
  end
end

# adapter/memory.rb
class Adapter
  class Memory
    def schemes
      [:memory]
    end

    # Methods to communicate with DB omitted.

    Adapter.register(self)
  end
end

Adapter.setup("memory://test").class
# => Adapter::Memory
```

Things to note in this example:

* The `Adapter` class has no knowledge of specific implementations. This means it doesn’t have to change when new adapters are added or removed.
* Finding a suitable implementation to a given scheme is now a matter of traversing an array. Code is much simpler and scales beautifully.
* I don’t have to load all adapter implementations, only those needed by the client code.
* `Adapter` class is can be tested in isolation. I can easily create and `register` a mock adapter. No need to touch the database in tests.

## Conclusion

In this example, I had a direct dependency from an high level `Adapter` class with low-level, specific implementations.

Then I turned the dependency around: made the low level implementations depend on the high level superclass.

The inverted dependency reduced code coupling, leading to more flexible, testable and maintainable code. And who doesn’t love that?

In case you’re interested, [here’s the full diff](https://github.com/rom-rb/rom/commit/5567e12413987d62570e5763318ae10e90a97ae0).

Questions? Thoughts? Comment on this [Reddit thread](http://www.reddit.com/r/ruby/comments/2l91t3/inverting_dependencies/) or hit me up on [twitter](http://twitter.com/abernardes).
