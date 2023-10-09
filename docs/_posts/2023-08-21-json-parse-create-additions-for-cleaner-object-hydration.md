---
layout: post
title: JSON's "Create Additions" for cleaner Ruby object Hydration
date: 2023-08-21 14:00:50 +0100
categories: ruby
author: paul_danelli
---

## Inspiration

[Hydrating](https://stackoverflow.com/a/20787106/3588645) Ruby objects from isn't the most simple, unless your data is exactly the same each time.

You could pass in an `object_class` keyword parameter, to specify the name of the class you want to hydrate each JSON object into:

```ruby
require "json"

data_source = <<JSON
  [
    {
        "type": "Car",
        "wheels": 4,
        "name": "Accord",
        "maker": "Honda"
    },
    {
        "type": "Bike",
        "wheels": 2,
        "name": "Gold wing",
        "maker": "Honda"
    }
  ]
JSON

objects = JSON.parse(data_source, object_class: OpenStruct)
# => [#<OpenStruct type="Car", wheels=4, name="Accord", maker="Honda">, # #<OpenStruct type="Bike", wheels=2, name="Gold wing", maker="Honda">]
```

This requires either using a flexible `OpenStruct` like data structure, or your data being the same each time.

There are alternatives to `OpenStruct` however, because as I previously wrote, using the library is now [considered an anti-pattern](https://tech.olioex.com/ruby/2023/03/08/why-openstruct-is-an-antipattern.html). We could use our own class to be hydrated from each JSON object within the array we are parsing.

```ruby
class Vehicle
  attr_accessor :wheels, :type, :name, :maker

  def initialize(data = nil)
    @data = data || {}
  end

  def []=(name, value)
    data[name.to_sym] = value
    public_send("#{name}=", value)
  end
end

objects = JSON.parse(data_source, object_class: Vehicle)
# [#<Vehicle:0x0000000108796180 @data={:type=>"Car", :wheels=>4, :name=>"Accord", :maker=>"Honda"}, @maker="Honda", @name="Accord", @type="Car", @wheels=4>, #<Vehicle:0x00000001089c9560 @data={:type=>"Bike", :wheels=>2, :name=>"Gold wing", :maker=>"Honda"}, @maker="Honda", @name="Gold wing", @type="Bike", @wheels=2>]

objects.first.wheels
# => 4
```

## Differing datasets

However, what happens if the data you are being passed is different?

```ruby
data_source = <<JSON
  [
    {
        "type": "Car",
        "wheels": 4,
        "name": "Accord",
        "maker": "Honda",
        "year": 2014
    },
    {
        "type": "Bike",
        "wheels": 2,
        "name": "Gold wing",
        "maker": "Honda",
        "country": "Japan"
    }
  ]
JSON
```

We would need to keep adding publicly accessible attributes to cover all scenarios, or start accessing the internal data via `instance_variable_get` - this is not a scalable solution.

```ruby
class Vehicle
  attr_accessor :wheels, :type, :name, :maker, :year, :country
  # ...
end
```

We could allow access to the `data` hash, however this means we need to access all data like:


```ruby
class Vehicle
  attr_accessor :data
  # ...
end

objects.first.data[:wheels]
# => 4
```

Or, we could start using `method_missing`, but at this point we've gone too far.

## Create Additions

`JSON.parse` allows a special keyword parameter to be passed in, which can be used to simplify the process of hydrating objects, called `create_additions`.

By default, `create_additions` doesn't do anything and returns the same as the default call to `JSON.parse`.

```ruby
# Both methods produce the same result by default
JSON.parse(data_source)
# => [{"type"=>"Car", "wheels"=>4, "name"=>"Accord", "maker"=>"Honda", "year"=>2014}, {"type"=>"Bike", "wheels"=>2, "name"=>"Gold wing", "maker"=>"Honda", "country"=>"Japan"}]

JSON.parse(data_source, create_additions: true)
# => [{"type"=>"Car", "wheels"=>4, "name"=>"Accord", "maker"=>"Honda", "year"=>2014}, {"type"=>"Bike", "wheels"=>2, "name"=>"Gold wing", "maker"=>"Honda", "country"=>"Japan"}]
```

With a small change to the structure of the JSON string, we can make our lives simpler; specifying a decoding class, and nesting our object data inside a `data` sub-object:

```ruby
data_source = <<JSON
  [
    {
      "json_class": "Vehicle",
      "data": {
        "type": "Car",
        "wheels": 4,
        "name": "Accord",
        "maker": "Honda"
      }
    },
    {
      "json_class": "Vehicle",
      "data": {
        "type": "Bike",
        "wheels": 2,
        "name": "Gold wing",
        "maker": "Honda"
      }
    }
  ]
JSON
```

The Ruby JSON module sets a default method for decoding the string JSON that its passed:

```ruby
JSON.create_id
# => "json_class"
```

`json_class` is the default, however we can change this if we like:

```ruby
JSON.create_id = "resource_type"
JSON.create_id
# => "resource_type"
```

If we choose to modify the `create_id`, we then make a small change to the data structure, matching the new decoding class:

```ruby
<<JSON
  [
    {
      "resource_type": "Vehicle",
      "data": {
        "type": "Car",
        "wheels": 4,
        "name": "Accord",
        "maker": "Honda"
      }
    },
  ]
JSON
```

### So what is the `json_class` attribute relate to?

`json_class` is used by the `create_additions` to automatically hydrate a JSON object into an instance of the class specified.

For example, the following class is a simple PORO that defines some instance variables and attribute readers:

```ruby
class Vehicle
  attr_reader :type, :wheels, :name, :maker

  # Normal initializer, not special in anyway
  def initialize(data)
    @data = data
    @type = data[:type]
    @wheels = data[:wheels]
    @name = data[:name]
    @maker = data[:maker]
  end
end
```

With the addition or a couple of extra methods, we can make it produce and consume JSON:

```ruby
class Vehicle
  attr_reader :type, :wheels, :name, :maker

  # Normal initializer, not special in anyway
  def initialize(data)
    @data = data
    @type = data[:type]
    @wheels = data[:wheels]
    @name = data[:name]
    @maker = data[:maker]
  end

  # Create our customised JSON export structure, including our decoding class name
  def to_json(options = {})
    {
      json_class: self.class.name,
      data: data,
    }.to_json(options)
  end

  # This method is passed a parsed Ruby Hash and then your own logic is
  # used to create a new instance anyway that you prefer.
  def self.json_create(vehicle_hash)
    new(vehicle_hash["data"].transform_keys(&:to_sym))
  end
end
```

We can now use this class to create and consume JSON strings:

```ruby
new_vehicle = Vehicle.new({ type: "Van", wheels: 4, name: "Transit", maker: "Ford" })
# => <Vehicle:0x0000000104639890 @data={:type=>"Van", :wheels=>4, :name=>"Transit", :maker=>"Ford"}, @maker="Ford", @name="Transit", @type="Van", @wheels=4>

json = new_vehicle.to_json
# => "{\"json_class\":\"Vehicle\",\"data\":{\"type\":\"Van\",\"wheels\":4,\"name\":\"Transit\",\"maker\":\"Ford\"}}"

# Without the additional params, we extract a normal Ruby Hash
vehicle_hash = JSON.parse(json)
# => {"json_class"=>"Vehicle", "data"=>{"type"=>"Van", "wheels"=>4, "name"=>"Transit", "maker"=>"Ford"}}

# With the additional param, we create instances of the class specified by the `json_class` attribute
vehicle = JSON.parse(json, create_additions: true)
# => <Vehicle:0x000000010457af58 @data={:type=>"Van", :wheels=>4, :name=>"Transit", :maker=>"Ford"}, @maker="Ford", @name="Transit", @type="Van", @wheels=4>
```

Using the JSON that we created above, we can now start to mimic the behaviour of the `object_class` parameter and have an array of hydrated instances returned to us.

```ruby
# And creating Vehicle instances from our JSON payload
vehicles = JSON.parse(data_source, create_additions: true)
# => [#<Vehicle:0x000000010463f9c0 @data={:type=>"Car", :wheels=>4, :name=>"Accord", :maker=>"Honda"}, @maker="Honda", @name="Accord", @type="Car", @wheels=4>, #<Vehicle:0x000000010463f970 @data={:type=>"Bike", :wheels=>2, :name=>"Gold wing", :maker=>"Honda"}, @maker="Honda", @name="Gold wing", @type="Bike", @wheels=2>]
```

## Create Additions and differing data

Where this gets really interesting though, is what happens if we change the decoding `json_class` object to be different classes, essentially using STI within our JSON data source:

```ruby
data_source = <<JSON
  [
    {
      "json_class": "Car",
      "data": {
        "type": "Car",
        "wheels": 4,
        "name": "Accord",
        "maker": "Honda",
        "year": 2023
      }
    },
    {
      "json_class": "Bike",
      "data": {
        "type": "Bike",
        "wheels": 2,
        "name": "Gold wing",
        "maker": "Honda",
        "country": "Japan"
      }
    }
  ]
JSON
```

Now we can create two subclasses for our vehicles, both of which inherit from `Vehicle` and share the functionality it provides. However, we can update the `initialize` method, as well as add in any custom functionality required by the subclasses - this is where the real power of this technique becomes apparent.

```ruby
class Car < Vehicle
  attr_reader :year

  def initialize(...)
    super(...)
    @year = data[:year]
  end
end

class Bike < Vehicle
  attr_reader :country

  def initialize(...)
    super(...)
    @country = data[:country]
  end
end
```

Now, when we re-parse the JSON string, we are provided two different instances, and able to easily access our different data, whilst retaining the inherited functionality.

```ruby
vehicles = JSON.parse(data_source, create_additions: true)
# => [#<Car:0x0000000103ef2eb0 @data={:type=>"Car", :wheels=>4, :name=>"Accord", :maker=>"Honda", :year=>2023}, @maker="Honda", @name="Accord", @type="Car", @wheels=4, @year=2023>, #<Bike:0x0000000103ef2e60 @country="Japan", @data={:type=>"Bike", :wheels=>2, :name=>"Gold wing", :maker=>"Honda", :country=>"Japan"}, @maker="Honda", @name="Gold wing", @type="Bike", @wheels=2>]

car = vehicles.first
# => #<Car:0x0000000103ef2eb0

bike = vehicles.last
# => #<Bike:0x0000000103ef2e60

car.year
# => 2023

bike.country
# => "Japan"
```

## Benchmarks

To be 100% transparent about this approach and any pitfalls, below is a benchmark script that I used to see how each of the methods did when processing the same data - the results can be found below.

```ruby
require "benchmark"
require "json"

iterations = 100_000

hash1 = { "type": "Car", "wheels": 4, "name": "Accord", "maker": "Honda", "year": 2023 }
hash2 = { "type": "Bike", "wheels": 2, "name": "Gold wing", "maker": "Honda", "country": "Japan" }
hash_array = []
iterations.times { hash_array << [hash1, hash2] }

hash_json = hash_array.flatten.to_json

additions_array = []
additions_hash1 = { "json_class": "Car", "data": hash1 }
additions_hash2 = { "json_class": "Bike", "data": hash2 }
iterations.times { additions_array << [additions_hash1, additions_hash2] }

additions_json = additions_array.flatten.to_json

class VehicleObjectClass
  attr_accessor :wheels, :type, :name, :maker, :year, :country

  def initialize(data = nil)
    @data = data || {}
  end

  def []=(name, value)
    @data[name.to_sym] = value
    public_send("#{name}=", value)
  end
end

class Vehicle
  attr_reader :type, :wheels, :name, :maker

  def initialize(data)
    @data = data
    @type = data[:type]
    @wheels = data[:wheels]
    @name = data[:name]
    @maker = data[:maker]
  end

  def self.json_create(vehicle_hash)
    new(vehicle_hash["data"].transform_keys(&:to_sym))
  end
end

class Car < Vehicle
  attr_reader :year

  def initialize(...)
    super(...)
    @year = data[:year]
  end
end

class Bike < Vehicle
  attr_reader :country

  def initialize(...)
    super(...)
    @country = data[:country]
  end
end

Benchmark.bmbm(10) do |b|
  # Hash is our base standard, but doesn't provide anywhere near the functionality a class could
  b.report("Hash") do
    objs = JSON.parse(hash_json)
    objs.map { |obj| Vehicle.new(obj) }
  end

  # The "recommended" way when Googling how to hydrate hashes into Objects
  b.report("Open Struct") do
    JSON.parse(hash_json, object_class: OpenStruct)
  end

  # An imperfect _super_ class where each new key needs to be added as an attribute accessor
  b.report("Object Class") do
    JSON.parse(hash_json, object_class: VehicleObjectClass)
  end

  # Our preferred solution, utilising child classes
  b.report("Create Additions") do
    JSON.parse(additions_json, create_additions: true)
  end
end
```

### Results

Interestingly, the warm up runs seems to show a pretty big difference in the times recorded compared with the later non-rehearsal runs - this might be an internal Ruby caching mechanism.

+ OpenStruct is orders of magnitude slower than any other approach - this method should be avoided
+ Hash is obviously the fastest way of parsing JSON, but provides the most minimal set of functionality
+ Object Class is twice as slow as the Hash approach
+ Create Additions is a little slower than Object Class, but provides greater flexibility and object encapsulation

#### 1,000 iterations

```
Rehearsal ----------------------------------------------------
Hash               0.002012   0.000030   0.002042 (  0.002041)
Open Struct        0.033986   0.000965   0.034951 (  0.034954)
Object Class       0.004101   0.000032   0.004133 (  0.004138)
Create Additions   0.004628   0.000058   0.004686 (  0.004688)
------------------------------------------- total: 0.045812sec

                       user     system      total        real
Hash               0.001564   0.000004   0.001568 (  0.001567)
Open Struct        0.026477   0.000471   0.026948 (  0.026946)
Object Class       0.003395   0.000028   0.003423 (  0.003422)
Create Additions   0.004385   0.000059   0.004444 (  0.004441)
```

#### 10,000 iterations

```
Rehearsal ----------------------------------------------------
Hash               0.044436   0.001575   0.046011 (  0.046015)
Open Struct        0.275587   0.015429   0.291016 (  0.291028)
Object Class       0.072656   0.000939   0.073595 (  0.073603)
Create Additions   0.048366   0.001526   0.049892 (  0.049897)
------------------------------------------- total: 0.460514sec

                       user     system      total        real
Hash               0.017947   0.000303   0.018250 (  0.018258)
Open Struct        0.231194   0.011322   0.242516 (  0.242517)
Object Class       0.033373   0.000318   0.033691 (  0.033691)
Create Additions   0.043272   0.000448   0.043720 (  0.043721)
```

#### 100,000 iterations

```
Rehearsal ----------------------------------------------------
Hash               0.282297   0.016722   0.299019 (  0.299020)
Open Struct        3.830766   0.400253   4.231019 (  4.231380)
Object Class       0.337428   0.081456   0.418884 (  0.419158)
Create Additions   0.541805   0.024302   0.566107 (  0.566385)
------------------------------------------- total: 5.515029sec

                       user     system      total        real
Hash               0.157892   0.001598   0.159490 (  0.159501)
Open Struct        3.235008   0.438133   3.673141 (  3.674261)
Object Class       0.453773   0.017481   0.471254 (  0.471257)
Create Additions   0.488088   0.018344   0.506432 (  0.506454)
```

## Appendix

+ https://ruby-doc.org/stdlib-3.0.0/libdoc/json/rdoc/JSON.html#module-JSON-label-Custom+JSON+Additions
+ https://docs.ruby-lang.org/en/3.2/JSON.html#module-JSON-label-JSON+Additions
+ https://gist.github.com/esquinas/85df587f7faa05095a2ff4414cfb59b8
