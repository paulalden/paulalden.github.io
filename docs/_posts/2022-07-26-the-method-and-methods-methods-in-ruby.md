---
layout: post
title: Debugging in Ruby using 'method' and 'methods'
date: 2022-07-26 14:00:50 +0200
categories: ruby
author: paul_danelli
---

## The `methods` method

*The problem; My memory is not great and I can't remember the name of every method defined on an instance*

I was trying to work out the cause of the exception below and because of all the `rescue` calls built into DD Trace, ActiveAdmin, Rails etc, the output was being suppressed. So, I managed to find a place in the code where I could see the exception, but wanted to see all of the stacktrace output.

```ruby
# An error occured here:
[128, 137] in ./ruby/2.7.2/lib/ruby/gems/2.7.0/gems/ddtrace-0.54.2/lib/ddtrace/contrib/action_pack/action_controller/instrumentation.rb
   128:                 status = datadog_response_status
   129:                 payload[:status] = status unless status.nil?
   130:                 result
   131:               # rubocop:disable Lint/RescueException
   132:               rescue Exception => e
=> 133:                 payload[:exception] = [e.class.name, e.message]
   134:                 payload[:exception_object] = e
   135:                 raise e
   136:               end
   137:             # rubocop:enable Lint/RescueException

# My error from the exception
$> e

<NoMethodError: undefined method `times' for 2.0:Float>
```

I can never remember what the method is called, so my first guess was...

```ruby
$> e.trace

*** NoMethodError Exception: undefined method `trace' for #<NoMethodError: undefined method `times' for 2.0:Float>
```

Obviously thats wrong, so a totally random second guess:

```ruby
$> e.stacktrace

*** NoMethodError Exception: undefined method `stacktrace' for #<NoMethodError: undefined method `times' for 2.0:Float>
Did you mean?  set_backtrace
```

That didn't work and the suggested method wasn't what I wanted either - now I've now decided that guessing isn't helping. Luckily there are more logical approaches, like using Ruby's `methods` method, which shows all methods on the class or instance (depending on what you call it on, there is an instance method as well as a class method):

`methods` gives us an array of symbols, ordered as they were included in the code, but its still an array, so we can `sort` it:

```ruby
$> e.methods.sort

[:!, :!=, :!~, :<=>, :==, :===, :=~, :__bb_context, :__binding__, :__id__,
:__send__, :acts_like?, :args, :as_json, :backtrace, :backtrace_locations,
:blame_file!, :blamed_files, :blank?, :bullet_key, :bullet_primary_key_value,
:byebug, [...], :to_s, :to_yaml, :trust, :try, :try!, :unfriendly_id?,
:unloadable, :untaint, :untrust, :untrusted?, :with_options, :yield_self]
```

If a class includes a lot of modules, `methods` can return an unhelpfully large array of unsorted method names - but its still an array, so we also can search for similar method names:

```ruby
$> e.methods.sort.grep /trace/

[:backtrace, :backtrace_locations, :set_backtrace]
```

Finally, we have the method that we were looking for... and I should probably have known that. Obviously, the array that `backtrace` returns is super long and mostly unhelpful, so I've shortened it below for the sake of brevity, but you can also use `grep` on the array values to do the same:

```ruby
$> e.backtrace

["./api/app/helpers/collections/replacer.rb:35:in `create_adhoc_replacements!'", "./api/app/controllers/api/v2/collections_controller.rb:155:in `block in paused_replacements'", [...], ]
```

We can make the output slightly more useful by creating a string and printing that to the screen:

```ruby
$> puts e.backtrace.join("\r\n")
```
```log
./api/app/helpers/collections/replacer.rb:35:in `create_adhoc_replacements!'
./api/app/controllers/api/v2/collections_controller.rb:155:in `block in paused_replacements'
# ...
```

And with that, I now know the issue I have and can start to resolve it.

## The `method` method

*The problem; It's annoying having to Google for method information online.*

I was recently working with some dates and wanted to know what exactly the `since` method did - luckily Ruby provides a handy way of viewing source code for exactly this reason. 

Below is the original code snippet I wanted to understand:

```ruby
$> first_adhoc_collection = 1.week.since(final_collection)

Fri, 24 Jun 2022 07:52:25.000000000 UTC +00:00
```

I can get the class name, which means I could Google for the documentation:

```ruby
$> 1.week.class

ActiveSupport::Duration
```

But, I would need to try and find the method and make sure I'm looking at the correct version of Rails, or I could use `method`.

`method` returns an instance of `Method`, which encapsulates all of the method's information, including its location on disk, the line number and the method body. It takes a symbol as an argument, so it would be called like so:

```ruby
$> 1.week.method(:since)

<Method: ActiveSupport::Duration#since(time=...) ./.asdf/installs/ruby/2.7.2/lib/ruby/gems/2.7.0/gems/activesupport-6.1.5.1/lib/active_support/duration.rb:409>
```

To see what other information is available to use, we can check the `public_methods` on the object:

```ruby
# `false` prevents us from seeing inherited methods
$> 1.week.method(:since).public_methods(false)

=> [:clone, :original_name, :<<, :>>, :to_proc, :==, :===, :to_s, :receiver, :[], :call, :eql?, :name, :inspect, :duplicable?, :arity, :curry, :source_location, :parameters, :hash, :owner, :unbind, :super_method]
```

So, I could call `source_location` and see exactly where it is...

```ruby
$> 1.week.method(:since).source_location

["./.asdf/installs/ruby/2.7.2/lib/ruby/gems/2.7.0/gems/activesupport-6.1.5.1/lib/active_support/duration.rb", 409]
```

Or I could just output the methods body and read through it using the `source` method:

```ruby
$> puts 1.week.method(:since).source

def since(time = ::Time.current)
  sum(1, time)
end
```

This helpful method isn't included in the list above, however since `method` is available on every instance, we can call it on the Method instance we had returned and see that the `source` method is defined elsewhere:

```ruby
$> 1.week.method(:since).method(:source)
=> #<Method: Method(MethodSource::MethodExtensions)#source() ./.asdf/installs/ruby/2.7.2/lib/ruby/gems/2.7.0/gems/method_source-1.0.0/lib/method_source.rb:109>
```

So, back to the problem at hand; From the `since` method, we see it calls `sum` and so we need more information. So, `method` to the rescue again...

```ruby
$> puts 1.week.method(:sum).source

def sum(sign, time = ::Time.current)
  unless time.acts_like?(:time) || time.acts_like?(:date)
    raise ::ArgumentError, "expected a time or date, got #{time.inspect}"
  end

  if parts.empty?
    time.since(sign * value)
  else
    parts.inject(time) do |t, (type, number)|
      if type == :seconds
        t.since(sign * number)
      elsif type == :minutes
        t.since(sign * number * 60)
      elsif type == :hours
        t.since(sign * number * 3600)
      else
        t.advance(type => sign * number)
      end
    end
  end
end
```

But calling `method` to get back each method that is referenced seems like hard work, so an alternative is to just output the file entirely:

```ruby
$> puts File.read(1.week.method(:sum).source_location.first)

# frozen_string_literal: true

require "active_support/core_ext/array/conversions"
require "active_support/core_ext/module/delegation"
require "active_support/core_ext/object/acts_like"
require "active_support/core_ext/string/filters"

module ActiveSupport
  # Provides accurate date and time measurements using Date#advance and
  # Time#advance, respectively. It mainly supports the methods on Numeric.
  #
  #   1.month.ago       # equivalent to Time.now.advance(months: -1)
  class Duration
    class Scalar < Numeric #:nodoc:
      attr_reader :value
      delegate :to_i, :to_f, :to_s, to: :value

      def initialize(value)
        @value = value
      end

      def coerce(other)
        [Scalar.new(other), self]
      end

      def since(time = ::Time.current)
        sum(1, time)
      end

# ... Removed for brevity

      private

      def sum(sign, time = ::Time.current)
        unless time.acts_like?(:time) || time.acts_like?(:date)
          raise ::ArgumentError, "expected a time or date, got #{time.inspect}"
        end

        if parts.empty?
          time.since(sign * value)
        else
          parts.inject(time) do |t, (type, number)|
            if type == :seconds
              t.since(sign * number)
            elsif type == :minutes
              t.since(sign * number * 60)
            elsif type == :hours
              t.since(sign * number * 3600)
            else
              t.advance(type => sign * number)
            end
          end
        end
      end
    end
  end
end
```

Ruby is a beautifully flexible language that allows you to monkey patch methods, include modules and use inheritance to extend your classes. However, with this flexibility it can quickly become hard to work out exactly what is being executed on the server. Thankfully, it also provides mechanisms for us to work that out, like `method` and `methods`.
