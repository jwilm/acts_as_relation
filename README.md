# ActsAsRelation
This gem helps implement multiple-table-inheritance (MTI) methods to your ActiveRecord models.
By default, ActiveRecord only supports single-table inheritance (STI).
MTI gives you the benefits of STI but without having to place dozens of empty fields into a single table.

Take a traditional e-commerce application for example... 
A product has common attributes (`name`, `price`, `image` ...), 
while each type of product has its own attributes...
`pen` has `color`, `book` has `author` and `publisher` and so on.

## Installation
For Rails 4 installation add the following line to your `Gemfile`

```ruby
gem 'acts_as_relation', '~> 1.0'
```

and

```shell
$ bundle install
```


If you are using Rails 3 you must use '~> 0.1' version specifier in Gemfile.

## Usage
`acts_as_relation` uses a polymorphic `has_one`
association to simulate multiple-table inheritance. 
For the e-commerce example you would declare the product as a `supermodel` and all types of it as `acts_as` `:product`
(if you prefer you can use their aliases `is_a` and `is_a_superclass`)

```ruby
class Product < ActiveRecord::Base
  acts_as_superclass
end

class Pen < ActiveRecord::Base
  acts_as :product
end

class Book < ActiveRecord::Base
  acts_as :product
end
```

To make this work, you need to declare both a foreign key column and a type column
in the model that declares superclass. To do this you can set `:as_relation_superclass`
option to `true` on `products` `create_table` (or pass it name of the association):

```ruby
create_table :products, :as_relation_superclass => true do |t|
  # ...
end
```

Or declare them as you do on a `polymorphic` `belongs_to` association, it this case
you must pass name to `acts_as` in `:as` option:

```ruby
change_table :products do |t|
  t.integer :producible_id
  t.string  :producible_type
end

class Pen < ActiveRecord::Base
  acts_as :product, :as => :producible
end

class Book < ActiveRecord::Base
  acts_as :product, :as => :producible
end
```

Now `Pen` and `Book` **act** **as** `Product`. This means that they inherit `Product`
_attributes_, _associations_, _validations_ and _methods_.

To see its functionality lets add some stuff to product:

```ruby
class Product
  validates_presence_of :name, :price

  def to_s
    "#{name} $#{price}"
  end
end
```

now we can to things like this:

```ruby
Pen.create :name => "Nice Pen", :price => 1.3, :color => "Red"
Pen.where "name = ?", "Some Pen"
pen = Pen.new
pen.valid?      # => false
pen.errors.keys # => [:name, :price]
Pen.first.to_s  # => "Nice Pen $1.3"
```

When you declare an `acts_as` relation, the declaring class automatically gains parent
methods (includeing accessors) so you can access them directly.

On the other hand you can always access a specific object from its parent by calling `specific` method on it:

```ruby
Product.first.specific # will return a specific product, a pen for example
```

## Options
The `acts_as` relation support these options:

* `:as`
* `:auto_join`
* `:class_name`
* `:dependent`

when `:auto_join` option set to `true` (which is by default), every query on child
will automatically joins the parent table. For example:

```ruby
Pen.where("name = ?", "somename")
```

will result in the following SQL:

```sql
SELECT "pens".* FROM "pens" INNER JOIN "products" ON "products"."as_product_id" = "pens"."id" AND "products"."as_product_type" = 'Pen' WHERE (name = 'somename')
```

All other options are same as `has_one` options.

Note that support for `:conditions` and `:include` has been removed. In their place, you can use
+where()+ and +includes()+:

```ruby
acts_as :product, -> { where(color: "yellow") }
```

