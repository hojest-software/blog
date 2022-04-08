# The Danger Of Test Doubles

Lucas Chizzoli - April 7, 2022 - Ruby, Test

Imagine the next scenario. You have a class that has an outgoing message.
You want to isolate the test to that particular class so the test won't need much change in a future.

```ruby
  class Person
    attr_reader :name, :friend

    def initialize(name:, friend:)
      @name   = name
      @friend = friend
    end

    def hello
      friend.hello  # <---- Outgoing message
    end
  end
```

You create a fake object, or _test double_, to play the outgoing message role.
A test double is an instance of a role player that is used for testing. The double _stubs_
the hello method and always returns 'Hey!'.

```ruby
  class FriendDouble
    def hello
      'Hey!'
    end
  end

  class PersonTest < Minitest::Test
    def test_say_hello
      person = Person.new(
        name: 'Lucas',
        friend: FriendDouble.new)

      assert('Hey!', person.hello)
    end
  end
```

So far so good. You isolated the _Person_ test _Friend_ and the test works as expected.
But what happen if _Friend_ class changes his public interface
and now instead of responding to _hello_ responds to _say_hi_?
If you run the test it will continue to pass though the application is definitely broken!

## Posible Solution

Friend is playing a role here. The role is hard to see because is nearly invisible. _Friend_ plays the _say hello role_.
One way to raise the role's visibility is to assert that _Friend_ plays it.

```ruby
  class PersonTest < Minitest::Test
    def setup
      @friend = Friend.new
    end

    def test_say_hello
      person = Person.new(
        name: 'Lucas',
        friend: FriendDouble.new)

      assert('Hey!', person.hello)
    end

    def test_implements_the_say_hello_interface
      assert_respond_to(@friend, :hello)
    end
  end
```

This is a step in the right direction but it is not yet the best approach. First it cannot be shared with other _say hello_. Other players of this role would need to duplicate this test. Secondly it doesn't prevent the _double_ to became obsolete and allow the test to erroneously pass.

To fix the first issue you can test the role in a separete module and include it on the test that plays that role.

```ruby
  module SayHelloInterfaceTest
    def test_implements_say_hello_interface
      assert_respond_to(@object, :hello)
    end
  end

  class PersonTest < Minitest::Test
    include SayHelloInterfaceTest

    def setup
      @friend = @object = Friend.new
    end

    def test_say_hello
      person = Person.new(
        name: 'Lucas',
        friend: FriendDouble.new)

      assert('Hey!', person.hello)
    end
  end
```

This will allow you and future developers to reuse the test in every object that plays the role. It also serves as documentation, it raises the visibility of the role.

And finally, to prevent the Double to became obsolete, you can use the same test module.

```ruby
  class PersonTest < Minitest::Test
    include SayHelloInterfaceTest

    def setup
      @object = FriendDouble.new
    end

    def test_say_hello
      person = Person.new(
        name: 'Lucas',
        friend: FriendDouble.new)

      assert('Hey!', person.hello)
    end
  end
```

## Conclusion

The need to write your own role test is neccesary because in dynamic typed languages, the roles are virtual. In static typed languages you can relay on the compiler to enforce the use of roles. Adding role tests will keep all players of a role communicating as expected.
