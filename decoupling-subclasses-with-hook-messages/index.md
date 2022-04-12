# Decoupling Subclasses With Hook Messages

Lucas Chizzoli - April 12, 2022 - Ruby, Inherit, Hook Messages

Imagine the next scenario. You have a _subclass_ that inherits its _superclass_ behavior.
The _superclass_ has an _initialize_ method that initializes some properties.
Those properties then are being accessed by the _subclasses_ by a method.

Lets see an example

```ruby
  class Vehicle
    attr_reader :engine, :doors

    def initialize(**opts)
      @engine = opts[:engine]
      @doors = opts[:doors] || default_doors
    end

    def default_doors
      4
    end

    def characteristics
      {
        engine: engine,
        doors: doors
      }
    end
  end
```

You proceed to create a _subclass_; you inherit from the _superclass_ and implement the corresponding methods.

```ruby
  class Car < Vehicle
    attr_reader :tires_count

    def initialize(**opts)
      @tires_count = opts[:tires_count]
      super # <---- You call the initilize of its superclass
    end

    def characteristics
      super.merge(tires_count: tires_count)
    end
  end

  puts Car.new(engine: 'electric', tires_count: 4).characteristics
  # { :engine => "electric", :doors => 4, :tires_count => 4 }
```

So far so good. At this point you might be temted to stop right here. However it still contains a trap.
Notice that `Car` subclass knows about it self (its custom **characteristics**) and things about its _superclass_
(that **characteristics** retuns a hash and that it responds to **initialize**).
Knowing things about other classes creates dependencies, and dependencies couple objects. In the code above the traps are created by the sends
of `super` in the subclasses.

The next example illustrates the trap. If someone creates a subclass and forgets to send `super` when initializing,
we encounter this problem:

```ruby
  class Bike < Vehicle
    attr_reader :tires_count

    def initialize(**opts)
      @tires_count = opts[:tires_count]
      # <---- Forgets to send super
    end

    def characteristics
      super.merge(tires_count: tires_count)
    end
  end

  puts Car.new(engine: 'gas', tires_count: 2).characteristics
  # { :engine => nil, :doors => nil, :tires_count => 2 } <---- engine and doors didn't get initialized
```

The same could happend if someone forgets to send super in **characteristics** method. Nothing breaks and that makes these errors hard to debug.

## The Solution, Hook Messages

We can use the Hook Message pattern to remove the subclass dependencies. Instead of allowing subclasses to know their superclasses interface,
superclasses can instead send `hook messages`. These messages will allow the subclasses to implement the matching methods and to remove knowledge of its superclass interface.

Lets see an example:

```ruby
  class Vehicle
    attr_reader :engine, :doors

    def initialize(**opts)
      @engine = opts[:engine]
      @doors = opts[:doors] || default_doors

      post_initialize(opts)
    end

    # hook method
    def post_initialize(opts)
    end

    def default_doors
      4
    end

    def characteristics
      {
        engine: engine,
        doors: doors
      }.merge(specific_characteristics)
    end

    # hook method for the subclass to override
    def specific_characteristics
      {}
    end
  end

  class Car < Vehicle
    attr_reader :tires_count

    def post_initialize(opts)
      @tires_count = opts[:tires_count] # <--- we remove the need to send super
    end

    def specific_characteristics
      {
        tires_count: tires_count
      } # <--- It's easier to create new subclasses this way
    end
  end

  puts Car.new(engine: 'electric', tires_count: 4).characteristics
  # { :engine => "electric", :doors => 4, :tires_count => 4 }
```

## Conclusion

This solution allows `Car` to know less about `Vehicle` and remove its dependencies.
New subclasses need only to implement hook methods.
