---
layout: page
---

{% include tutorial-toc.html %}

<div markdown="1" class="col-md-8">

## Step 0: Bootstrapping

> [View the Code](https://github.com/graphiti-api/employee_directory/commit/e2552ce212c68b41a3eb8161deb822fff3e159d6)

Let's start by creating a new Rails project. For help with an existing
project, check out [Installation: From
Scratch]({{site.github.url}}/guides/getting-started/installation).

We'll use the `-m` option to install from a template, which will add a few gems and apply some setup boilerplate. Accept all the default options.

{% highlight bash %}
$ rails new employee_directory --api -m https://www.graphiti.dev/template
$ cd employee_directory
{% endhighlight %}

> Note: if a network issue prevents you from pointing to this URL directly, you can download the file and and run this command as `-m /path/to/template`

Feel free to run `git diff` to see what the generator did, otherwise commit the result. You can now head to [Step 1: Basic Resource]({{site.github.url}}/tutorial/step_1), or continue reading to better understand the code.

#### Digging Deeper 🧐

You'll see some boilerplate in `config/routes.rb`:

{% highlight ruby %}
scope path: ApplicationResource.endpoint_namespace, defaults: { format: :jsonapi } do
  # your routes go here
end
{% endhighlight %}

This tells Rails that our API routes will be be prefixed - `/api/v1` by default. It also
says that if no extension is in the URL (`.json`, `.xml`, etc), default
to the [JSONAPI Specification](http://jsonapi.org).

Let's look at the above `ApplicationResource`:

{% highlight ruby %}
class ApplicationResource < Graphiti::Resource
  self.abstract_class = true

  # We'll be using ActiveRecord
  self.adapter = Graphiti::Adapters::ActiveRecord

  # Links are generated from base_url + endpoint_namespace
  self.base_url = Rails.application.routes
    .default_url_options[:host]
  self.endpoint_namespace = '/api/v1'
end
{% endhighlight %}

This should be pretty self-explanatory except for

{% highlight ruby %}
self.base_url = Rails.application.routes
  .default_url_options[:host]
{% endhighlight %}

This is configured in `config/application.rb`:

{% highlight ruby %}
module EmployeeDirectory
  class Application < Rails::Application
    routes.default_url_options[:host] = ENV.fetch('HOST', 'http://localhost:3000')
    # ... code ...
  end
end
{% endhighlight %}

When deriving and validating [Links]({{site.github.url}}/guides/links), we'll use the `HOST` variable if
present, falling back to the Rails development default of
`http://localhost:3000`. This means our Links will look like:

{% highlight ruby %}
"#{ENV['HOST']}/#{ApplicationRecord.endpoint_namespace}/#{Resource.type}"
{% endhighlight %}

For example:

{% highlight ruby %}
http://my-website.com/api/v1/employees
{% endhighlight %}

Read more in the [Links Guide]({{site.github.url}}/guides/links).

Finally, there's some boilerplate in `ApplicationController`:

{% highlight ruby %}
class ApplicationController < ActionController::API
  # Support JSON, XML, JSON:API
  include Graphiti::Rails::Responders
end
{% endhighlight %}

This gets `respond_with` working, via the [Responders gem](https://github.com/plataformatec/responders). To render simple nested JSON like default Rails, we'll only need to add `.json` to the URL.

That's it for basic setup!

</div>

<div class="clearfix">
  <h2 id="next">
    <a href="{{site.github.url}}/tutorial/step_1">
      NEXT - 
      <small>Step 1: Basic Resource</small>
      &raquo;
    </a>
  </h2>
</div>
