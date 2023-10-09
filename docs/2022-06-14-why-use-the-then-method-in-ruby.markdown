---
layout: post
title: "Why use the 'then' method in Ruby?"
date: 2022-06-14 14:00:50 +0200
categories: ruby
author: paul_danelli
---

# Then / Yield Self

In this quick read, I'm going to discuss a Ruby method called [then](https://docs.ruby-lang.org/en/master/Kernel.html#method-i-then), which is a much better named alias of [yield_self](https://docs.ruby-lang.org/en/master/Kernel.html#method-i-yield_self).

## TL;DR

Just in case you want some quick ammunition for a discussion, the benefits of using `then` are:

+ Variable encapsulation
+ Neater code
+ Easier to read
+ Object Oriented

## The Task

We are going to make a quick API request to Olio's staging server and parse the JSON it returns into a Ruby Hash.

```ruby
url = "https://s3-eu-west-1.amazonaws.com/olio-staging-images/developer/test-articles-v4.json"
```

We will need to include a couple of extra Ruby modules before we can continue, so I've added those below:

```ruby
require "json"
require "net/http"
require "uri"
```

Now we can perform our request...

## The Simple Example

In our first example, we just want to make the request, collect the response body and parsing the JSON into a hash of products.

### The Procedural Example

A simple approach to might be to use individual variables to store the result of each function call. This would be fine for a quick example, but could get unwieldy quickly if you needed to extend the functionality in the future.

```ruby
uri = uri(url)
res = net::http.get_response(uri)
json = res.body
products = json.parse(json || "{}")
```

### A One-liner

To make the code more concise, we could combine it into one line and create a single variable to store the result. This however, becomes unreadable with anything more than a couple of function calls in our expression.

```ruby
products = JSON.parse(Net::HTTP.get_response(URI(url)).body || "{}")
```

### Using `then` (`yield_self`)

Using `then` we get the best of both worlds, a clean and easy to read set of expressions, but without the variable pollution of the procedural example. You might notice that our blocks take a single argument which is the result from the previous blocks execution - so by naming our block arguments as such, we can explain the purpose of each section and what is returned.

```ruby
products = url.
  then { |string| URI(string) }.
  then { |uri| Net::HTTP.get_response(uri).body || "{}" }.
  then { |json| JSON.parse(json) }
```

#### A Quick Debugging Aside

It's really easy to debug with loads of variables holding onto a single value, but when using an approach like that described above, we could use the `tap` method to inspect the content at any point, so that debugging is not a cumbersome chore.

```ruby
products = url.
  then { |string| URI(string) }.
  then { |uri| Net::HTTP.get_response(uri).body || "{}" }.
  tap { |json| puts json }. # ... or byebug here to see what `json` is
  then { |json| JSON.parse(json) }
```

## A More Complicated Example

In this example, we build from the previous expression, but this time we want to get the photos of all Christmas related products:

### Procedural example

As you can see, our procedural example is now starting to getting a little hard to read and we have a number of variables that will never be used, except by the function on the next line.

```ruby
uri = URI(url)
res = net::http.get_response(uri)
json = res.body
products = json.parse(json || "{}")
christmas_products = products.filter { |product| product["title"].match?(/christmas|xmas/i) }
christmas_images = christmas_products.map { |product| product.dig("photos") }.flatten
images = christmas_images.flatten.map { |photo| photo.dig("files", "medium") }
```

### Using `then` (`yield_self`)

We do not need to _force_ the use of `then` here, we can use simple method chaining to process the data as we want, but keep the object oriented approach and reduce the cluster of the procedural example.

```ruby
images = url.
  then { |string| URI(string) }.
  then { |uri| Net::HTTP.get_response(uri).body || "{}" }.
  then { |json| JSON.parse(json) }.
  filter { |product| product["title"].match?(/christmas|xmas/i) }.
  map { |product| product.dig("photos") }.
  flatten.
  map { |photo| photo.dig("files", "medium") }
```

There is much more we could do, but for this quick read that is enough for now. I hope this has helped you in some small way.

