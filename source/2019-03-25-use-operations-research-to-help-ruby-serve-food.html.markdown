---
title: Use Operations Research to help Ruby serve food
date: 2019-03-25 08:13 UTC
tags: ruby, optimization
author: guillaume
published: false
---

In this article I will try and give you a taste of how you can use Operations Research to solve a
food serving problem we had at Zesty, in San Francisco.

We wanted to serve local companies their meals on a weekly basis. Companies would subscribe to our
program, and we would bring them restaurant food that suits their needs, according to some
requirements they had: the number of people, their dietary preferences, the variety they wanted
etc.

In this article I will simplify our requirements to and I will propose a way to find which
restaurant meals to give to companies for each of their orders.

I will use ruby to solve this, and a gem called `ruby-cbc` that is a wrapper around the Open Source
Integer Linear Programming library Coinor-Cbc.

# The problem

In this section, I will describe our 2 main actors here, clients and restaurants.

## Clients

A client is a company that wants some meals to be delivered. It has a name (id), a number of
people that will eat (headcount), and some dates it want food on (orders_on).

```ruby
class Client
  attr_reader :id, :headcount, :orders_on
  def initialize(id:, headcount:, orders_on:)
    @id = id
    @headcount = headcount
    @orders_on = orders_on
  end
end
```

## Restaurants

A restaurant is providing food for the clients. It has a name (id), a number of meals it can
provide per date (capacity), and is open on certain dates (opens_on).

```ruby
class Restaurant
  attr_reader :id, :capacity, :opens_on
  def initialize(id:, capacity:, opens_on:)
    @id = id
    @capacity = capacity
    @opens_on = opens_on
  end
```

## The objective

Our aim here is to make sure each client gets a restaurant assigned for each of its orders, and
that each restaurant serves food that doesn't exceed its capacity.

In this problem we focus mainly on clients, we don't mind if a restaurant doesn't serve any food.

## Intermediate models

As you see in previous section, we spoke about orders. I will define 2 more models that will help
define our problem more precisely, one related to a client (an order), and one related to a
restaurant (a slot).

An order is given by a client at a certain date. It is pretty trivial to create orders from a
client, we just loop through the dates a client wants orders on, and create as many orders.

```ruby
class Order
  attr_reader :id, :client, :date
  def initialize(id:, client:, date:)
    @id = id
    @client = client
    @date = date
  end
end

class CreateOrders
  def self.call(clients:)
    new(clients).call
  end

  attr_reader :clients
  def initialize(clients)
    @clients = clients
  end

  def call
    orders = Collection.new(Order) # a collection is like a hash of objects with and id
    clients.each do |client|
      client.orders_on.each do |date|
        orders.add(order_id(client, date), client: client, date: date)
      end
    end

    orders
  end

  def order_id(client, date)
    "#{shorten(client.id)}-#{shorten(date)}"
  end

  def shorten(id)
    id[0..2].upcase
  end
end
```

A slot is a opening date for a restaurant. Like for orders, it is easy to create our list of slots:
each restaurant has as many slots as the dates it opens on.

```ruby
class Slot
  attr_reader :id, :restaurant, :date
  def initialize(id:, restaurant:, date:)
    @id = id
    @restaurant = restaurant
    @date = date
  end
end
```

Now we can redefine our problem with: given a set of orders and slots, find a slot for each order
that satisfies its client's headcount, making sure that orders on a given slot don't exceed the
slot's restaurant capacity.

A solution to our problem will be a set of couples (order, slot) that satisfies our constraints of
headcount and capacity.

## Solution

A solution is a set of couples (order, slot). We will call such a couple an planned meal.

```ruby
class PlannedMeal
  attr_reader :id, :order, :slot
  def initialize(id:, order:, slot:)
    @id = id
    @order = order
    @slot = slot
  end
end
```

# A brute-force approach

How would you solve such a problem? If we only have a handful of clients and restaurants, solving
it with a brute-force approach will be enough probably. We will try this first.

Here is how it can look like:

We take each order one by one, and we try to assign it a slot.

- If we can (i.e. there is a slot that has enough capacity to serve the order), we move on to the
  next order.
- If there is no slot left, we backtrack to the last order, and assign another slot to that order
  in the hope it will work.
Once all orders are assigned a slot, we are done and we found a solution.

Let's have a small example to illustrate it:

We have 2 clients (Quantcast and MuleSoft, clients we had when I was working at Zesty), and 2
restaurants (BunMee and Dosa, we were working with them too).

Here are them:

```ruby
quantcast = Client.new(id: 'Quantcast', headcount: 50, orders_on: %w[monday tuesday])
mulesoft = Client.new(id: 'MuleSoft', headcount: 200, orders_on: %w[monday tuesday])

bunmee = Restaurant.new(id: 'BunMee', capacity: 230, opens_on: %w[monday tuesday])
dosa = Restaurant.new(id: 'Dosa', capacity: 120, opens_on: %w[monday tuesday])
```

We will have 4 orders, monday and tuesday for Quantcast and MuleSoft, and 4 slots, monday and
tuesday for BunMee and Dosa.

For monday, our backtracking algorithm could search like this:

```
Quantcast-Monday <-- BunMee  (BunMee-Monday still has 230-50=180 capacity)
MuleSoft-Monday <-- ? Not enough capacity for the restaurants
Backtracking to Quantcast-Monday again

Quantcast-Monday <-- Dosa  (Dosa-Monday still has 120-50=70 capacity)
MuleSoft-Monday <-- BunMee (BunMee-Monday still has 230-200=30 capacity)
```

It will be the same for tuesday. In our small example with almost no constraint, we can solve the
different days separately. In later examples, some constraints will span the whole week and we will
have to solve the week entirely.

Using a backtracking algorithm only doesn't work when our data grows. Optimization algorithms like
Constraint Programming approaches, and Integer Linear Programming approaches use some backtracking
alongside other refinement techniques to find the solution.

We will use ILP here to avoid having to write a complicated algorithm for our problem.

## Integer Linear Programming approach

We will start this section with a little introduction to ILP, and we will then model our problem
with this approach. I will then show you how you can use `ruby-cbc` to model it in Ruby.

### What is ILP?

ILP is an optimization 
