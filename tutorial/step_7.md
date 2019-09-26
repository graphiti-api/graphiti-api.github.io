---
layout: page
---

{% include tutorial-toc.html %}

<div markdown="1" class="col-md-8">

### Step 7: Many to Many

> [View the Diff](https://github.com/graphiti-api/employee_directory/compare/step_6_write_customization...step_7_many_to_many)

Let's add a `Team` relationship: a `Team` can have many `Employee`s, an
`Employee` can have many `Team`s. Let's also say a `Team` belongs to a
`Department`.

<table class="table table-small text-center">
  <thead>
    <tr>
      <th class="text-center">id</th>
      <th class="text-center">department_id</th>
      <th class="text-center">name</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>1</td>
      <td>1</td>
      <td>The A Team</td>
    </tr>
    <tr>
      <td>2</td>
      <td>1</td>
      <td>The B Team</td>
    </tr>
    <tr>
      <td>3</td>
      <td>2</td>
      <td>The C Team</td>
    </tr>
  </tbody>
</table>

To satisfy this many-to-many use case, we'll need a join model,
`TeamMembership`:

<table class="table table-small text-center">
  <thead>
    <tr>
      <th class="text-center">id</th>
      <th class="text-center">team_id</th>
      <th class="text-center">employee_id</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>1</td>
      <td>1</td>
      <td>1</td>
    </tr>
    <tr>
      <td>2</td>
      <td>2</td>
      <td>1</td>
    </tr>
    <tr>
      <td>3</td>
      <td>3</td>
      <td>2</td>
    </tr>
  </tbody>
</table>

### The Rails Stuff 🚂

{% highlight bash %}
$ bin/rails g model Team name:string department:belongs_to
$ bin/rails g model TeamMembership employee:belongs_to team:belongs_to
$ bin/rails db:migrate
{% endhighlight %}

Graphiti supports `has_many :through`:

{% highlight ruby %}
# app/models/employee.rb
has_many :team_memberships
has_many :teams, through: :team_memberships
{% endhighlight %}

{% highlight ruby %}
# app/models/department.rb
has_many :teams
{% endhighlight %}

{% highlight ruby %}
class Team < ApplicationRecord
  belongs_to :department
  has_many :team_memberships
  has_many :employees, through: :team_memberships
end
{% endhighlight %}

{% highlight ruby %}
class TeamMembership < ApplicationRecord
  belongs_to :team
  belongs_to :employee
end
{% endhighlight %}

Finally, we'll need a new seed file to handle these new associations:

{% highlight ruby %}
[
  Employee,
  Position,
  Department,
  TeamMembership,
  Team
].each(&:delete_all)

departments = []
def create_department(name)
  dept = Department.create! name: name
  dept.teams.create!(name: 'Engineering Team B')
  dept.teams.create!(name: 'Engineering Team C')
  dept
end

departments << create_department('Engineering')
departments << create_department('Safety')
departments << create_department('QA')

100.times do
  employee = Employee.create! first_name: Faker::Name.first_name,
    last_name: Faker::Name.last_name,
    age: rand(20..80)

  (1..2).each do |i|
    employee.positions.create! title: Faker::Job.title,
      historical_index: i,
      active: i == 1,
      department: departments.sample
  end

  employee.teams << employee.positions[0].department.teams.sample
end
{% endhighlight %}

### The Graphiti Stuff 🎨

{% highlight bash %}
$ bin/rails g graphiti:resource Team name:string
{% endhighlight %}

Let's flesh out our `TeamResource`:

{% highlight ruby %}
# app/resources/team_resource.rb
class TeamResource < ApplicationResource
  attribute :department_id, :integer, only: [:filterable]
  attribute :name, :string

  belongs_to :department
  many_to_many :employees
end
{% endhighlight %}

The trick here is the `many_to_many` relationship. Let's add the reverse
as well:

{% highlight ruby %}
# app/resources/employee_resource.rb
many_to_many :teams
{% endhighlight %}

And for good measure:

{% highlight ruby %}
# app/resources/department_resource.rb
has_many :teams
{% endhighlight %}

We can now get all the usual functionality: fetch Employees and their
Teams in a single request (or vice versa).

#### Digging Deeper 🧐

The `many_to_many` relationship is the only one where Graphiti modifies
a separate Resource "under the hood". When we said `many_to_many
:employees`, the `EmployeeResource` got a `team_id` filter, and
`many_to_many :teams` created an `employee_id` filter on `TeamResource`.

This is because the logic is more complex than the default use case. We
don't have a simple `WHERE` clause; we need to join tables and look at
the appropriate primary/foreign keys. If the name of your API
association doesn't match the name of your ActiveRecord association, try
`has_many :things, as: :my_activerecord_relationship` to make the
introspection work correctly - or, write your own filter.

Sometimes you'll have multiple levels of `has_many :through`. In this
case, a simple `many_to_many` isn't enough - check out our [Hopping
Relationships]({{site.github.url}}/cookbooks/hopping-relationships) Cookbook.

Think hard before reaching for `many_to_many`. Imagine one Team is the
"primary" Team for an Employee. We'd add a `primary` boolean column to
the `team_memberships` table...but that table isn't exposed to the API!
Consider if there's a hidden domain concept there.

</div>

<div class="clearfix">
  <h2 id="next">
    <a href="/tutorial/step_8">
      NEXT - 
      <small>Step 8: Polymorphic Relationships</small>
      &raquo;
    </a>
  </h2>
</div>
