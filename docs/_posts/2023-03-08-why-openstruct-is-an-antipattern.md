---
layout: post
title: Why OpenStruct is an Anti Pattern
date: 2023-03-08 14:00:50 +0200
categories: ruby
author: paul_danelli
---

## Inspiration

I recently read an [article](https://towardsdev.com/alternatives-for-rubys-openstruct-ec27f4e20ea7) on why we shouldn't be using Ruby's `OpenStruct` and wanted to expand on it a little.

Before we begin, for those of you new to `OpenStruct` and data structures, I'll dig a little deeper.

## What is a data structure?

[This article](https://www.techtarget.com/searchdatamanagement/definition/data-structure) explains it more concisely than I could:

> A data structure is a specialized format for organizing, processing, retrieving and storing data. There are several basic and advanced types of data structures, all designed to arrange data to suit a specific purpose. Data structures make it easy for users to access and work with the data they need in appropriate ways. Most importantly, data structures frame the organization of information so that machines and humans can better understand it.

So a data structure is how you store the information you collect so that you can access it quickly and easily later on. An example of this in Ruby (and most programming languages) is an Array.

```ruby
arr = [:foo, 5, "bar"]
puts arr[0]
=> :foo
```

## What's an OpenStruct?

An `OpenStruct` is a data structure, similar to a Hash, that allows you to create arbitrary attributes, each with an accompanying value. It does this by using Ruby's meta programming to define methods on the class itself.

So, how is this different from a normal Struct?

When using a "normal" `Struct`, you define the attributes you wish to store, then using that template, you can create instances of the `Struct`, each with their own values.

```ruby
# First we describe our Struct and the data that we will be passing in
vehicle = Struct.new(:wheels, :mileage)

# Then we can access it using the dot-method access
bike = vehicle.new(2, 1000)
=> #<struct  wheels=2, mileage=1000>

bike.wheels
=> 2
```

`OpenStruct` is more flexible than this, allowing us to define our attributes and data at the same time:

```ruby
car = OpenStruct.new(wheels: 4, mileage: 5000)
=> #<OpenStruct wheels=4, mileage=5000>

car.wheels
=> 4
```

The downside to this flexibility however is that `OpenStruct` is SLOOOOOOOOOOOOW... an order of magnitude slower than using a normal `Struct` or a Ruby `Hash` - see below for some benchmarks.

The reason `OpenStruct` is so much slower than other data structures in Ruby, is that it uses...

> Ruby’s method lookup structure to and find and define the necessary methods for properties. This is accomplished through the method `method_missing` and `define_method`.

According to [Rubocop](https://www.rubydoc.info/gems/rubocop/0.61.1/RuboCop/Cop/Performance/OpenStruct)...

> Instantiation of an OpenStruct invalidates Ruby global method cache as it causes dynamic method definition during program runtime. This could have an effect on performance, especially in case of single-threaded applications with multiple OpenStruct instantiations.

## Alternatives to OpenStruct

+ [Struct](https://ruby-doc.org/core-3.1.1/Struct.html)
+ [Hash](https://ruby-doc.org/core-3.1.1/Hash.html) - No dot access however
+ [Class](https://ruby-doc.org/core-3.1.1/Class.html)
+ [Dry Rb Struct](https://dry-rb.org/gems/dry-struct/1.0/)
+ [ActiveSupport Ordered Options](https://api.rubyonrails.org/classes/ActiveSupport/OrderedOptions.html)
+ [Hashie](https://github.com/hashie/hashie)
+ [Ruby 3 Data Class](https://docs.ruby-lang.org/en/master/Data.html)

After being inspired by the initial article above, I wanted to benchmark the additional alternatives so we can see which was the quickest approach, whilst covering the majority of the `OpenStruct` use cases - named attributes and method access to attribute values.

## The Script

This is the script I used - which can be copied and pasted, however, you'll need to Gem install dry-struct. I wanted to ensure that not only was the speed of storing data accounted for, but also access speed, so after the items are created, we iterate over them and access the mileage attribute.

```ruby
require 'benchmark'
require 'dry-struct'
require 'ostruct'
require 'hashie'
require 'active_support'

class ClassCar
  attr_accessor :wheels, :mileage
end

car_struct = Struct.new(:wheels, :mileage)

module Types
  include Dry.Types()
end
class DryCar < Dry::Struct
  attribute :wheels, Types::Integer
  attribute :mileage, Types::Integer
end

Car = Data.define(:wheels, :mileage)

iterations = 250_000

Benchmark.bmbm(10) do |b|
  b.report("OStruct") do
    items = []
    iterations.times do
      car = OpenStruct.new
      car.wheel = 4
      car.mileage = rand((1..100))
      items << car
    end

    items.each { |item| item.mileage }
  end

  b.report("Class") do
    items = []
    iterations.times do
      car = ClassCar.new
      car.wheels = 4
      car.mileage = rand((1..100))
      items << car
    end

    items.each { |item| item.mileage }
  end

  b.report("Struct") do
    items = []
    iterations.times do
      items << car_struct.new(4, rand((1..100)))
    end

    items.each { |item| item.mileage }
  end

  b.report("OrderedOptions") do
    items = []
    iterations.times do
      car = ActiveSupport::OrderedOptions.new
      car.wheels = 4
      car.mileage = rand((1..100))
      items << car
    end

    items.each { |item| item.mileage }
  end

  b.report("Hash") do
    items = []
    iterations.times do
      items << { wheels: 4, mileage: rand((1..100)) }
    end

    items.each { |item| item[:mileage] }
  end

  b.report("DryStruct") do
    items = []
    iterations.times do
      items << DryCar.new(wheels: 4, mileage: rand((1..100)))
    end

    items.each { |item| item.mileage }
  end

  b.report("Hashie") do
    items = []
    iterations.times do
      items << Hashie::Mash.new(wheels: 4, mileage: rand((1..100)))
    end

    items.each { |item| item.mileage }
  end

  b.report("Array") do
      items = []
    iterations.times do
      items << [4, rand((1..100))]
    end

    items.each { |item| item[1] }
  end

  b.report("Ruby3 Data Class") do
    items = []
    iterations.times do
      items << Car.new(wheels: 4, mileage: rand((1..100)))
    end

    items.each { |item| item.mileage }
  end
end

```

## Results

And here are the results:

```
                       user     system      total        real        difference
OStruct            2.207787   0.184141   2.391928 (  2.397279)     
Class              0.050523   0.000413   0.050936 (  0.050945)    47.05x faster
Struct             0.044087   0.000263   0.044350 (  0.044365)    54.03x faster 
OrderedOptions     0.243925   0.004669   0.248594 (  0.248626)     9.64x faster
Hash               0.047014   0.001815   0.048829 (  0.048835)    49.08x faster
DryStruct          0.409692   0.005251   0.414943 (  0.414971)     5.77x faster 
Hashie             0.647923   0.018294   0.666217 (  0.666253)     3.59x faster 
Array              0.030275   0.000376   0.030651 (  0.030661)    78.18x faster 
Ruby3 Data Class   0.073789   0.000631   0.074420 (  0.074437)    32.20x faster
```

As you can see, `OpenStruct` is considerably slower than using a Hash or a normal Ruby Class as a data store. This is obviously fine if you're writing one-time scripts, or performance is not a consideration.

Each of the Gems and classes above will have their own pros and cons, but if performance is a consideration, `Hash` should be your goto data store.
