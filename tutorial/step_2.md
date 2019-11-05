---
layout: page
---

{% include tutorial-toc.html %}

<div markdown="1" class="col-md-8">

## Step 2: Has Many

> [View the Code](https://github.com/graphiti-api/employee_directory/compare/step_1_employees...step_2_positions)

We'll be adding the database table `positions`:

<table class="table table-small text-center">
  <thead>
    <tr>
      <th class="text-center">id</th>
      <th class="text-center">employee_id</th>
      <th class="text-center">title</th>
      <th class="text-center">active</th>
      <th class="text-center">historical_index</th>
      <th class="text-center">created_at</th>
      <th class="text-center">updated_at</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>1</td>
      <td>900</td>
      <td>Engineer</td>
      <td>true</td>
      <td>1</td>
      <td>2018-09-04</td>
      <td>2018-09-04</td>
    </tr>
    <tr>
      <td>2</td>
      <td>900</td>
      <td>Intern</td>
      <td>true</td>
      <td>2</td>
      <td>2018-09-04</td>
      <td>2018-09-04</td>
    </tr>
    <tr>
      <td>3</td>
      <td>800</td>
      <td>Manager</td>
      <td>true</td>
      <td>1</td>
      <td>2018-09-04</td>
      <td>2018-09-04</td>
    </tr>
  </tbody>
</table>

Because this table tracks all historical positions, we have the
`historical_index` column. This tells the order the employee moved
through each position, where `1` is most recent.

### The Rails Stuff 🚂

Generate the `Position` model:

{% highlight bash %}
$ bin/rails g model Position title:string active:boolean historical_index:integer employee:belongs_to
$ bin/rails db:migrate
{% endhighlight %}

Update the `Employee` model with the association, too:

{% highlight ruby %}
# app/models/employee.rb
has_many :positions
{% endhighlight %}

And update our seed data:

{% highlight ruby %}
# db/seeds.rb
[Employee, Position].each(&:delete_all)

100.times do
  employee = Employee.create! first_name: Faker::Name.first_name,
    last_name: Faker::Name.last_name,
    age: rand(20..80)

  (1..2).each do |i|
    employee.positions.create! title: Faker::Job.title,
      historical_index: i,
      active: i == 1
  end
end
{% endhighlight %}

{% highlight bash %}
$ bin/rails db:seed
{% endhighlight %}

### The Graphiti Stuff 🎨

Let's start by running the same command as before to create
`PositionResource`:

{% highlight bash %}
$ bin/rails g graphiti:resource Position title:string active:boolean
{% endhighlight %}

We'll need to add the association, just like ActiveRecord:

{% highlight ruby %}
# app/resources/employee_resource.rb
has_many :positions
{% endhighlight %}

...and a corresponding filter:

{% highlight ruby %}
# app/resources/position_resource.rb
filter :employee_id, :integer
{% endhighlight %}

If you visit `/api/v1/employees`, you'll see a number of HTTP
[Links](https://www.graphiti.dev/guides/concepts/links)
that allow lazy-loading positions. Or, if you visit
`/api/v1/employees?include=positions`, you'll load the employees and
positions in a single request. We'll dig a bit deeper into this logic
in the section below.

Before we get there, let's revisit the `historical_index` column. For now, let's
treat this as an implementation detail that the API should not expose -
let's say we want to support sorting on this attribute but nothing else:

{% highlight ruby %}
attribute :historical_index, :integer, only: [:sortable]
{% endhighlight %}

We're almost done, but if you run your tests you'll see two outstanding
errors. This is because [Rails 5 belongs_to associations are required by
default](https://blog.bigbinary.com/2016/02/15/rails-5-makes-belong-to-association-required-by-default.html). We can't save a `Position` without its corresponding `Employee`.

We can solve this in three ways:

* Turn this off globally, with [config.active_record.belongs_to_required_by_default](https://edgeguides.rubyonrails.org/configuring.html#configuring-active-record). You may want to do this in test-mode only.
* Turn this off for the specific association: `belongs_to :employee,
optional: true`.
* Associate an `Employee` as part of the API request.

We'll take for the last option. Look at
`spec/resources/position/writes_spec.rb`:

{% highlight ruby %}
RSpec.describe PositionResource, type: :resource do
  describe 'creating' do
    let(:payload) do
      {
        data: {
          type: 'positions',
          attributes: { }
        }
      }
    end

    let(:instance) do
      PositionResource.build(payload)
    end

    it 'works' do
      expect {
        expect(instance.save).to eq(true)
      }.to change { Position.count }.by(1)
    end
  end
end
{% endhighlight %}

When running our tests, let's make sure the `historical_index` column
reflects the order we created the positions. This code recalculates
everything after a record is saved:

{% highlight ruby %}
# spec/factories/position.rb
FactoryBot.define do
  factory :position do
    employee

    title { Faker::Job.title }

    after(:create) do |position|
      unless position.historical_index
        scope = Position
          .where(employee_id: position.employee.id)
          .order(created_at: :desc)
        scope.each_with_index do |p, index|
          p.update_attribute(:historical_index, index + 1)
        end
      end
    end
  end
end
{% endhighlight %}

Let's associate an `Employee`. Start by seeding the data:

{% highlight ruby %}
let!(:employee) { create(:employee) }
{% endhighlight %}

And associate via `relationships`:

{% highlight ruby %}
let(:payload) do
  {
    data: {
      type: 'positions',
      attributes: { },
      relationships: {
        employee: {
          data: {
            id: employee.id.to_s,
            type: 'employees'
          }
        }
      }
    }
  }
end
{% endhighlight %}

To ensure the `PositionResource` will process this relationship, the
last step is to add it:

{% highlight ruby %}
# app/resources/position_resource.rb
belongs_to :employee
{% endhighlight %}

This will associate the `Position` to the `Employee` as part of the
creation process. The test should now pass - make the same change to
`spec/api/v1/positions/create_spec.rb` to get a fully-passing test
suite.

#### Digging Deeper 🧐

Why did we need the `employee_id` filter above? To explain that, let's dive deeper into the logic connecting Resources.

If you hit `/api/v1/employees`, you'll see a number of
[Links](https://www.graphiti.dev/guides/concepts/links) in the
response. These are useful for lazy-loading, but the same logic
applies to eager loading. Let's take a look at a Link to see how these
Resources connect together:

{% highlight ruby %}
{
  ...
  relationships: {
    positions: {
      links: {
        related: "http://localhost:3000/api/v1/positions?filter[employee_id]=1"
      }
    }
  }
  ...
}
{% endhighlight %}

The salient bit: `/positions?filter[employee_id]=1`. In other words,
fetch all Positions for the given Employee id.That means, whether we're lazy-loading data in separate requests or
eager-loading in a single request, **the same logic fires
under-the-hood**:

{% highlight ruby %}
PositionResource.all({
  filter: { employee_id: 1 }
})
{% endhighlight %}

This means we need `filter :employee_id, :integer` to satisfy the query.

We can customize the logic connecting Resources in a few different
ways. First some simple options:

{% highlight ruby %}
has_many :positions, foreign_key: :emp_id, primary_key: :eid
{% endhighlight %}

So far so good. The logic, and corresponding Link, both update as you'd
expect (though we'd of course need a corresponding `filter
:emp_id, :integer` on `PositionResource`).

Those options are just simple versions of parameter customization.
You can customize parameters connecting Resources with the `params` block:

{% highlight ruby %}
has_many :positions do
  params do |hash, employees|
    hash[:filter] # => { employee_id: employees.map(&:id) }
    hash[:filter][:active] = true
    hash[:sort] = '-created_at'
  end
end
{% endhighlight %}

Customizing these params affects the Link as well as the eager-load
logic. Remember the parameters here should reflect the JSON:API
specification, or anything `PositionResource.all` accepts.

These are the most common options, but there's a bunch more. Check
out the [Resource Relationships Guide]({{site.github.url}}/guides/concepts/resources#relationships) to dig even deeper.

</div>

<div class="clearfix">
  <h2 id="next">
    <a href="/tutorial/step_3">
      NEXT - 
      <small>Step 3: Belongs To</small>
      &raquo;
    </a>
  </h2>
</div>
