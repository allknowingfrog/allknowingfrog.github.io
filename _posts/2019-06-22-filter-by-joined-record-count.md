---
layout: post
title:  "Filter by Joined Record Count"
date:   2019-06-22 10:00:00 -0500
---
 
I ran into an interesting query problem with one of my Rails projects recently. I have a pair of models where one acts as a grouping of the other. Each parent object has a definied capacity, and I needed to return groups where the capacity had not yet been reached.

Imagine, for example, loading ships of various sizes with containers. Shipping containers themselves are standardized, but different quantities will fit on each vessel. As you group containers into shipments, it would be helpful to know which ones have space remaining. We can capture the core problem with two simple models:

```ruby
# app/models/shipment.rb
class Shipment < ApplicationRecord
  has_many :containers
end
```

```ruby
# app/models/container.rb
class Container < ApplicationRecord
  belongs_to :shipment
end
```

Overall presence or absence of associated records is a much more common problem, and a much easier one to solve. If I want all empty shipments or all non-empty shipments, I can use a basic join for the former and a left outer join on null for the latter.

```ruby
# app/models/shipment.rb
class Shipment < ApplicationRecord
  has_many :containers

  def self.not_empty
    joins(:containers)
  end

  def self.empty
    left_joins(:containers).where(containers: { id: nil })
  end
end
```

After a number of failed attempts to create a scope that would count and join at the same time, I determined that aggregating a left join is hard. I couldn't use an inner join, because that would eliminate shipments with no containers (which is about as "not full" as you can get). I finally realized that every field involved needed to be accounted for in the grouping to avoid ambiguity.

```ruby
Shipment
  .joins('LEFT JOIN "containers" ON "containers"."shipment_id" = "shipments"."id"')
  .group('"shipments"."id", "shipments"."capacity", "containers"."shipment_id"')
  .having('COUNT("containers"."shipment_id") < "shipments"."capacity"')
```

This is nearly what I wanted, but the grouping causes some unexpected ActiveRecord behavior. Calling `count()` on this scope yields a hash of values by id, rather than the total number of records. It probably contains other suprises as well. I considered plucking the list of ids and using it to filter a second query, but that idea smells. After some light reading, I learned that an ActiveRecord query passed as a where value auto-magically becomes a subquery, avoiding a second trip to the database.

```ruby
# app/models/shipment.rb
class Shipment < ApplicationRecord
  # ...
  def self.not_full
    where(id: Shipment
      .joins('LEFT JOIN "containers" ON "containers"."shipment_id" = "shipments"."id"')
      .group('"shipments"."id", "shipments"."capacity", "containers"."shipment_id"')
      .having('COUNT("containers"."shipment_id") < "shipments"."capacity"')
    )
  end
end
```

```
irb(main):010:0> Shipment.not_full
  Shipment Load (1.4ms)  SELECT  "shipments".* FROM "shipments" WHERE "shipments"."id" IN (SELECT "shipments"."id" FROM "shipments" LEFT JOIN "containers" ON "containers"."shipment_id" = "shipments"."id" GROUP BY "shipments"."id", "shipments"."capacity", "containers"."shipment_id" HAVING (COUNT("containers"."shipment_id") < "shipments"."capacity")) LIMIT $1  [["LIMIT", 11]]
```

It's not nearly as elegant as the empty/not_empty scopes, but I'm not sure it can be improved. Feel free to [prove me wrong](https://www.twitter.com/allknowingfrog). I refuse to believe that no one else has solved this, but I scoured the web for answers to no avail.

In case this *is* the right answer, note that scopes for full and over_full are created by replacing the less-than with an equaltiy and a greater-than comparison, respectively. This would result in three scopes sharing the bulk of their code, which implies the possibility of extracting a function, but I haven't explored it fully. I did push the code up to a [repository](https://github.com/allknowingfrog/boats) for anyone else who cares to experiment.
