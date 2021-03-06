# Hashing

Gives you an easy way to specify which instances vars of your objects should be
used as `key => value` to serialize it into a hash returned by the `#to_h`
method. Also gives you a `YourClass::from_hash` to reconstruct the instances.

## Status
[![Gem Version](https://badge.fury.io/rb/hashing.svg)](http://badge.fury.io/rb/hashing)
[![Build Status](https://travis-ci.org/ricardovaleriano/hashing.png?branch=master)](http://travis-ci.org/ricardovaleriano/hashing?branch=master)
[![Code Climate](https://codeclimate.com/github/ricardovaleriano/hashing.png)](https://codeclimate.com/github/ricardovaleriano/hashing)

## Installation

Add this line to your application's Gemfile:

    gem 'hashing'

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install hashing

## Usage

Given a `File` class like this:

```ruby
class File
  def initialize(path, commit = nil, content = nil)
    @path, @commit, @content = path, commit, content
  end
end
```

And I want to turn instances of it in a `Hash` like the one below, so I can, for
example, turn this `Hash` into a `YAML` to save it in a file.

```ruby
{
  path: @path,
  commit: @commit,
  content: @content
}
```

I just need to include the `Hashing` module, and then indicates which
`instance vars` (`ivars`) should be used in the serialization using the class
method `hasherize`:

```ruby
class File
  include Hashing
  hasherize :path, :commit, :content
end
```

### #to_h

Then I will be able to call the `#to_h` method in any instance of `File` to
obtain the "hasherized®" version of it:

```ruby
file = File.new 'README.md', 'cfe9aacbc02528b', '#Hashing\n\nWow. Such code...'

file.to_h
# {
#   path: 'README.md',
#   commit: 'cfe9aacbc02528b',
#   content: '#Hashing\n\nWow. Such code...'
# }
```

### ::from_hash

And I can tell `Hashing` how one can create an instance of `File` given a
valid `Hash` like the one created by a `#to_h` call:

```ruby
class File
  include Hashing
  hasherize :path, :commit, :content

  loading ->(hash) {
    new hash[:path], hash[:commit], hash[:content]
  }

  # ...
end
```

So I can transform instances of any Ruby class into hashes and rebuild objects
of any type from hashes, accordingly with my business rules, without expose
these rules outside my classes.

Note that both `#to_h` and `::from_hash` methods are public, and you can and
should call them whatever you need in your programs. But the `::from_hash`
method will be called by `Hashing` when building your instances from hashes (more
about this: [nested hasherizing](#nested-hasherized-objects)).

#### Hasherize

`Hashing` comes with an alternative way to indicate what fields should be used to
represent your objects as a `Hash`. This can be done by the `Hasherize` class.
The previous example can be writen like this:

```ruby
class File
  include Hasherize.new :path, :commit, :content

  loading ->(hash) {
    new hash[:path], hash[:commit], hash[:content]
  }

  # ...
end
```

It's just a matter of taste. Use the one that is more appealing to you.

### Custom "hasherizing" and loading strategies

Many times you will need apply some computation over the contents of your
`ivars` transform them in a primitive that can be stored as a `Hash`.

Following the `File` class example, maybe you want to store the `@content` as a
`Base64` enconded string.

`Hashing` allows you to specify the strategy of serialization and loading when
indicating which `ivars` should be part of the final `Hash`:

```ruby
require 'base64'

class File
  include Hashing

  hasherize :path, :commit

  hasherize :content,
    to_hash: ->(content) { Base64.encode64 content },
    from_hash: ->(content_string) { Base64.decode64 content_string }

  loading ->(hash) {
    new hash[:path], hash[:commit], hash[:content]
  }

  # ...
end
```


#### Nested hasherized objects

But if your transformations are a little more complicated than a simple `Base64`
encoding, chances are there that you have a nested objects to be serialized.

Extending our example, maybe our `File` class have an internal collection of
`Annotation`, to allow the user to indicate what is in which line of a file:

```ruby
# annotation.rb

class Annotation
  include Hasherize.new :lines, :annotation

  def initialize(lines, annotation)
    @lines = lines
    @annotation = annotation
  end
end
```

And the `File` class could have a method to add new annotations:

```ruby
# file.rb

class File
  # ...

  def annotate(lines, annotation)
    @annotations ||= []
    @annotations << Annotation.new(lines, annotation)
  end
  # ...
end
```

So in this case, if you wants a file to be `hasherized®` with it's internall
`@annotations` preserved, you just indicate this in the `File` class. The
example now can be rewriten as:

```ruby
class File
  include Hashing

  hasherize :path, :commit, :annotations

  # ...
end
```

Since the `Annotation` class has it's own notion of `#to_h` and `::from_hash`,
this is all that `Hashing` needs to build a `File` instances from a valid `Hash`.

## Contributing

1. Fork it ( https://github.com/[my-github-username]/hasher/fork )
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create a new Pull Request
