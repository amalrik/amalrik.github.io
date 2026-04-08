---
layout: post
title: "Having the cake and eating it too: Introduction to delegated types in rails"
date: 2026-04-07 10:40:00 -0300
categories: [ruby, refactoring, design-patterns]
tags: [ruby-on-rails, delegated-types, active-record]
---

Imagine you work for _Connection CRM_ a vital tool for your sales and support team. Your boss asks you to develop an easy way
to visualize different events in a single screen.
The event types involve PurchaseActivities, CommentActivities and many more.

```ruby
# db/migrate/..._create_accounts.rb
create_table :accounts do |t|
  t.string :name
  t.timestamps
end

# db/migrate/..._create_comment_activities.rb
create_table :comment_activities do |t|
  t.references :account, null: false, foreign_key: true
  t.text :comment_body
  t.integer :post_id
  t.timestamps
end

# db/migrate/..._create_purchase_activities.rb
create_table :purchase_activities do |t|
  t.references :account, null: false, foreign_key: true
  t.string :product_sku
  t.decimal :amount, precision: 8, scale: 2
  t.timestamps
end
```

The first implementation of a controller to gather this data would be something like this:
```ruby
# app/controllers/feed_controller.rb
class FeedController < ApplicationController
  def index
    @account = Account.find(params[:id])

    comments = @account.comment_activities
    purchases = @account.purchase_activities

    @feed = (comments + purchases).sort_by(&:created_at).reverse
  end
end
```

Well so far so good, you think your job is done. But what if some feeds have thousands of activities?
Pagination is impossible in this scenario since you can't pull several tables at once and use a consistent
LIMIT/OFFSET.
You could put a hard-coded LIMIT on the queries like this to save you database:

```ruby
  LIMIT_ACTIVITIES = 50
  comments = @account.comment_activities.limit(LIMIT_ACTIVITIES)
  purchases = @account.purchase_activities.limit(LIMIT_ACTIVITIES)
```

But that creates a poor user experience and truth be told looks like a hack, not a solution.
We can do better than that.

## Meet Delegated Types

Delegated types are Rails' answer to a classic problem: when your models share some common attributes but have significantly different specific ones. 

It sits between two extremes:
- **Single-Table Inheritance (STI)**: All subclasses share one table with nullable columns for everything. Works when subclasses are similar, becomes a mess when they're different.
- **Class-Table Inheritance**: Each model has its own table, but you lose the ability to query across them easily.

Delegated types gives you the best of both worlds: a central table for shared attributes, specific tables for unique attributes, and the ability to query, paginate, and sort across all types as if they were one.

## A Visual Comparison

Let's see how the three approaches look at the database level:

### Single-Table Inheritance (STI)

One table to rule them all. Everything lives together, even the columns you don't need:

![visual example STI](/assets/img/rails-delegated-types/sti-2026-04-08-0942.png)


Works well when subclasses are similar. But look at those `NULL` columns—`product_sku` is always empty for comments, `comment_body` is always empty for purchases. Not ideal.

### Polymorphic Associations

Each type gets its own table. Clean separation, but now you can't query them together:

![visual example Polymorphic Association](/assets/img/rails-delegated-types/polymorphic-associations-2026-04-08-0942.png)

No more null columns, but no way to get "the last 50 activities" in one query. You're stuck combining collections in Ruby.

### Delegated Types

The best of both worlds: shared data in one place, specific data in separate tables:

![visual example Delegated type](/assets/img/rails-delegated-types/delegated-types-2026-04-08-0942.png)

The central `activities` table holds what all types share (`account_id`, timestamps). Each specific table holds only what makes it unique.

### At a Glance

| Aspect | STI | Polymorphic | Delegated Types |
|--------|-----|-------------|-----------------|
| Unified query | ✅ | ❌ | ✅ |
| Pagination | ✅ | ❌ | ✅ |
| No null columns | ❌ | ✅ | ✅ |
| Type-specific tables | ❌ | ✅ | ✅ |
| Easy joins | ✅ | ❌ | ✅ |

## The Refactoring

Let's refactor our implementation to use delegated types.

### The Migrations

We keep our specific tables but now they'll only contain what makes each type unique. We also create a central table to hold shared data:

```ruby
# db/migrate/..._create_activities.rb
create_table :activities do |t|
  t.references :account, null: false, foreign_key: true
  t.string :activityable_type
  t.bigint :activityable_id
  t.timestamps
end

# db/migrate/..._create_comment_activities.rb
create_table :comment_activities do |t|
  t.text :comment_body
  t.integer :post_id
end

# db/migrate/..._create_purchase_activities.rb
create_table :purchase_activities do |t|
  t.string :product_sku
  t.decimal :amount, precision: 8, scale: 2
end
```

Notice how `account_id` and `timestamps` moved from the specific tables to the central `activities` table. This is the key insight: these are shared attributes that belong in one place.

### The Models

First, we create a concern that our specific models will include:

```ruby
# app/models/concerns/activityable.rb
module Activityable
  extend ActiveSupport::Concern

  included do
    has_one :activity, as: :activityable, dependent: :destroy
  end
end
```

This concern establishes the relationship between each specific activity type and the central Activity model.

Now our models:

```ruby
# app/models/activity.rb
class Activity < ApplicationRecord
  delegated_type :activityable, types: %w[CommentActivity PurchaseActivity]
  belongs_to :account
end

# app/models/comment_activity.rb
class CommentActivity < ApplicationRecord
  include Activityable
end

# app/models/purchase_activity.rb
class PurchaseActivity < ApplicationRecord
  include Activityable
end
```

The `delegated_type` macro does the heavy lifting: it creates a polymorphic `belongs_to :activityable` and generates several convenience methods we'll explore shortly.

### The Controller

Here's how our controller now looks:

```ruby
# app/controllers/feed_controller.rb
class FeedController < ApplicationController
  def index
    @account = Account.find(params[:id])
    @activities = @account.activities.order(created_at: :desc).limit(50)
  end
end
```

One query. One collection. Pagination works naturally. The N+1 problem is solved with the usual `includes`:

```ruby
@activities = @account.activities
  .order(created_at: :desc)
  .limit(50)
  .includes(:activityable)
```

## Rendering Polymorphic Views

To render each activity type correctly, we use a pattern similar to STI but with separate files:

```erb
<%# app/views/activities/_activity.html.erb %>
<%= render "activities/activityables/#{activity.activityable_name}", activity: activity %>
```

```erb
<%# app/views/activities/activityables/_comment_activity.html.erb %>
<div class="activity comment-activity">
  <span class="icon">💬</span>
  <p><%= activity.comment_activity.comment_body %></p>
  <small>on post <%= activity.comment_activity.post_id %></small>
</div>
```

```erb
<%# app/views/activities/activityables/_purchase_activity.html.erb %>
<div class="activity purchase-activity">
  <span class="icon">🛒</span>
  <p><%= activity.purchase_activity.product_sku %></p>
  <strong>R$ <%= activity.purchase_activity.amount %></strong>
</div>
```

The `activityable_name` method returns the underscored class name, so `CommentActivity` becomes `comment_activity`.

## Creating Records with Nested Attributes

Delegated types work seamlessly with nested attributes:

```ruby
# app/models/activity.rb
class Activity < ApplicationRecord
  delegated_type :activityable, types: %w[CommentActivity PurchaseActivity]
  accepts_nested_attributes_for :activityable
  belongs_to :account
end
```

Now you can create everything in one step:

```ruby
Activity.create!(
  account: account,
  activityable_type: "CommentActivity",
  activityable_attributes: {
    comment_body: "Hello world",
    post_id: 42
  }
)
```

Using the traditional approach (pre-Rails 8):

```ruby
# In your controller
def activity_params
  params.require(:activity).permit(:account_id, :activityable_type, activityable_attributes: [:comment_body, :post_id, :product_sku, :amount])
end
```

Or with Rails 8's new `params.expect`:

```ruby
# In your controller - Rails 8+
def activity_params
  params.expect(activity: [:account_id, :activityable_type, activityable_attributes: [:comment_body, :post_id, :product_sku, :amount]])
end
```

## Convenience Methods

The `delegated_type` macro generates several useful methods:

```ruby
# Type checking
activity.comment_activity?  # => true/false
activity.purchase_activity? # => true/false

# Direct access (returns nil if wrong type)
activity.comment_activity
activity.purchase_activity

# Class-level scopes
Activity.comments    # => Activity.where(activityable_type: "CommentActivity")
Activity.purchases  # => Activity.where(activityable_type: "PurchaseActivity")
```

## When Delegated Types Shine

Delegated types are the right choice when:

1. **Shared metadata exists**: Multiple models have common fields like `account_id`, `created_at`, `creator_id` that should live in one place.

2. **Subtypes diverge significantly**: Unlike STI where you'd have many null columns, each table only has what it needs.

3. **You need unified queries**: Feeds, timelines, and activity streams become simple queries instead of complex unions.

4. **Polymorphism isn't enough**: Regular polymorphic associations don't give you type-specific queries or easy joins.

## Wrapping Up

Delegated types give you a third path between STI's simplicity and class-table inheritance's flexibility. You get a central table for shared concerns, specific tables for unique attributes, and the ability to treat different models as one when you need to.

The next time you're building a feed, an audit log, or any system with "things that are all the same kind of thing but different in specific ways," consider this pattern.

