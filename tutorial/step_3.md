---
layout: page
---

{% include tutorial-toc.html %}

<div markdown="1" class="col-md-8">

### Step 3: Belongs To

> [View the Diff](https://github.com/graphiti-api/employee_directory/compare/step_2_positions...step_3_departments)

We'll be adding the database table `departments`:

<table class="table table-small text-center">
  <thead>
    <tr>
      <th class="text-center">id</th>
      <th class="text-center">name</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>1</td>
      <td>Engineering</td>
    </tr>
    <tr>
      <td>2</td>
      <td>Safety</td>
    </tr>
    <tr>
      <td>3</td>
      <td>QA</td>
    </tr>
  </tbody>
</table>

We'll also be adding a `department_id:integer` foreign key column to
the `positions` table.

### The Rails Stuff 🚂

Generate the `Department` model:

{% highlight bash %}
$ bin/rails g model Department name:string
{% endhighlight %}

To add the foreign key to `positions`:

{% highlight bash %}
$ bin/rails g migration add_department_id_to_positions
{% endhighlight %}

{% highlight ruby %}
class AddDepartmentIdToPositions < ActiveRecord::Migration[5.2]
  def change
    add_foreign_key :positions, :departments
  end
end
{% endhighlight %}

Update the database:

{% highlight bash %}
$ bin/rails db:migrate
{% endhighlight %}

Update our seed file:

{% highlight ruby %}
[Employee, Position, Department].each(&:delete_all)

engineering = Department.create! name: 'Engineering'
safety = Department.create! name: 'Safety'
qa = Department.create! name: 'QA'
departments = [engineering, safety, qa]

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
end
{% endhighlight %}

Make sure to update `spec/factories/departments.rb` with randomized
data. Then, since this is also a required relationship, update
`spec/factories/positions.rb` to always seed a department when we ask to
create a position:

{% highlight ruby %}
factory :position do
  employee
  department

  # ... code ...
end
{% endhighlight %}

### The Graphiti Stuff 🎨

You should be used to this by now:

{% highlight bash %}
bin/rails g graphiti:resource Department name:string
{% endhighlight %}

Add the association:

{% highlight ruby %}
# app/resources/position_resource.rb
belongs_to :department
{% endhighlight %}

And review the end of [Step 2]({{site.github.url}}/tutorial/step_2) to get all your specs
passing (add the department to the request payload). Practice makes perfect!

#### Digging Deeper 🧐

Note that we didn't need a filter like we did in step two. That's
because the primary key connecting the Resources is `id` by
default. In other words, the Link would be something like:

{% highlight bash %}
/departments?filter[id]=1
{% endhighlight %}

Which we get out-of-the-📦

But remember, you can customize these relationships just like the
previous `has_many` section.

</div>

<div class="clearfix">
  <h2 id="next">
    <a href="/tutorial/step_4">
      NEXT - 
      <small>Step 4: Customizing Queries</small>
      &raquo;
    </a>
  </h2>
</div>
