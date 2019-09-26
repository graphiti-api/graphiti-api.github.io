---
layout: page
---

{% include tutorial-toc.html %}

<div markdown="1" class="col-md-8">

### Step 4: Customizing Queries

> [View the Diff](https://github.com/graphiti-api/employee_directory/compare/step_3_departments...step_4_customizations)

So far, we've done fairly straightforward queries. If a user filters on
`first_name`:

`/api/v1/employees?filter[first_name]=Foo`

We'll query the equivalent database column:

{% highlight ruby %}
Employee.where(first_name: 'Foo')
{% endhighlight %}

But what if there's more complex logic? Let's say we want to sort
Employees on their `title` - which comes from the `positions` table.
How would that work?

### The Rails Stuff 🚂

First, we need to get data for an Employee's **current** position.
Let's start by defining what `current` means

{% highlight ruby %}
# app/models/position.rb
scope :current, -> { where(historical_index: 1) }
{% endhighlight %}

> See the [ActiveRecord Scopes](https://guides.rubyonrails.org/active_record_querying.html#scopes) documentation if you're unfamiliar with this concept.

Reference this scope in a new association:

{% highlight ruby %}
has_one :current_position,
  -> { current },
  class_name: 'Position'
{% endhighlight %}

Before moving on, let's review what we need to do. The `ActiveRecord`
code for sorting Employees on their current position's title would be:

{% highlight ruby %}
Employee.joins(:current_position).merge(Position.order(title: :asc))
{% endhighlight %}

Let's wire this up to Graphiti:

### The Graphiti Stuff 🎨

We're only going to **sort** and **filter** on the `title` attribute -
never display or persist. So start by defining the attribute as such:

{% highlight ruby %}
attribute :title, :string, only: [:filterable, :sortable]
{% endhighlight %}

Then the `sort` DSL to place our custom query:

{% highlight ruby %}
# app/resources/employee_resource.rb
sort :title do |scope, direction|
  scope.joins(:current_position).merge(Position.order(title: direction))
end
{% endhighlight %}

That's it! When a request to sort on the title comes in, we'll alter our
scope to join on the `positions` table, and order based on the current
position `title`.

The solution for filtering is similar:

{% highlight ruby %}
# app/resources/employee_resource.rb
filter :title do
  eq do |scope, value|
    scope.joins(:current_position).merge(Position.where(title: value))
  end
end
{% endhighlight %}

We can now filter on title:

`/api/v1/employees?filter[title]=Foo`

Let's do one more example - how would we order Employees by department
name? We *could* start the same way:

{% highlight ruby %}
attribute :department_name, :string, only: [:sortable]
{% endhighlight %}

But if we're ***only*** sorting, this is actually redundant. Whenever we
use the `sort` or `filter` DSL, we're just creating a sort-only or
filter-only attribute under the hood. So let's define everything in one
shot:

{% highlight ruby %}
sort :department_name, :string do |scope, value|
  scope.joins(current_position: :department)
    .merge(Department.order(name: value))
end
{% endhighlight %}

Remember: you only need to pass the type as the second argument when an
attribute doesn't already exist. And if you ever get an error saying
something is unfilterable or unsortable, check to see if you've already
defined a filter-only or sort-only attribute using these methods.

#### Digging Deeper 🧐

There's a critical part of Graphiti that makes everything easier: start
by imagining it doesn't exist.

In other words, the meat of the logic above had nothing to do with
Graphiti code - we're simply "wiring up" independent ActiveRecord
queries. If you're ever confused about query logic, get things working
without Graphiti first.

We could have changed the above to ActiveRecord scopes like
`.order_by_title(title)`, making the wiring code even simpler. Consider
doing this when the logic is reusable or particlar complex, but be aware
of the tradeoffs of [double-testing units](https://www.graphiti.dev/guides/concepts/testing#double-testing-units).

</div>

<div class="clearfix">
  <h2 id="next">
    <a href="/tutorial/step_5">
      NEXT - 
      <small>Step 5: Has One</small>
      &raquo;
    </a>
  </h2>
</div>
