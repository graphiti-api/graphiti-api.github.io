---
layout: page
---

<div markdown="1" class="toc col-md-3">
Error Handling
==========

* 1 [Overview](#overview)
  * [Setup](#setup)
  * [Displaying Raw Errors](#displaying-raw-errors)
* 2 [Usage](#usage)
  * [Basic](#basic)
  * [Advanced](#advanced)
* 3 [Testing](#testing)

</div>

<div markdown="1" class="col-md-8">

{% include h.html tag="h2" text="1 Overview" a="overview" %}

Whenever we have an application error, we want to respond with a
[JSONAPI-compliant errors payload](http://jsonapi.org/format/#errors).
This way clients have a predictable response detailing information about
the error.

<p align="center">
  <img width="100%" src="https://user-images.githubusercontent.com/55264/45879320-06096300-bd72-11e8-8920-6e83f4a2fce3.png">
</p>

We'll also need a way to customize this payload. For instance, if a
`NotAuthorized` error is raised, the response should have a `403` status
code. For other errors, we may want to render a helpful error message:

{% highlight ruby %}
class ApplicationController < ActionController::API
  register_exception NotAuthorized, status: 403
  register_exception ShipmentDelayed,
    message: ->(e) { "Contact us at 123-456-7899" }
  # ... code ...
end
{% endhighlight %}

Correctly handling exceptions happens in [graphiti-rails](https://github.com/graphiti-api/graphiti-rails). Customizing the behavior based on error class happens in the [RescueRegistry](https://github.com/wagenet/rescue_registry) dependency.

{% include h.html tag="h3" text="1.1 Setup" a="setup" %}

Just make sure you have [graphiti-rails](https://github.com/graphiti-api/graphiti-rails) installed:

{% highlight ruby %}
gem 'graphiti-rails'
{% endhighlight %}

And you're all set!

{% include h.html tag="h4" text="1.1.1 Displaying Raw Errors" a="displaying-raw-errors" %}

<p align="center">
  <img width="100%" src="https://user-images.githubusercontent.com/55264/45879208-9bf0be00-bd71-11e8-8427-282c5a426394.png">
</p>
<br />

It can be useful to display the raw error as part of the JSON response -
but you probably don't want to expose your stack trace to customers.
Let's only show raw errors for the `staging` environment:

{% highlight ruby %}
class ApplicationController < ActionController::API
  # ... code ...

  def show_detailed_exceptions?
    Rails.env.staging?
  end
end
{% endhighlight %}

Another common pattern is to only show raw errors when the user is
privileged to see them:

{% highlight ruby %}
class ApplicationController < ActionController::API
  # ... code ...

  def show_detailed_exceptions?
    current_user.admin?
  end
end

{% endhighlight %}

When `#show_detailed_exceptions?` returns `true`, you'll get the raw error class,
message, and backtrace in the JSON response.

{% include h.html tag="h2" text="2 Usage" a="usage" %}

{% include h.html tag="h3" text="2.1 Basic" a="basic" %}

Let's register an error with a custom response code:

{% highlight ruby %}
register_exception Errors::NotAuthorized, status: 403
{% endhighlight %}

Now if we `raise Errors::NotAuthorized`, the response code will be
`403`.

Additional options:

{% highlight ruby %}
register_exception Errors::NotAuthorized,
  status: 403,
  title: "You cannot perform this action",
  message: true, # render the raw error message
  message: ->(error) { "Invalid Action" } # message via proc
{% endhighlight %}

[See full documentation in the RescueRegistry README](https://github.com/wagenet/rescue_registry).

All controllers will inherit any registered exceptions from their parent. They can also add their own. In this example, `FooError` will only throw a custom status code when thrown from `FooController`:

{% highlight ruby %}
class FooController < ApplicationController
  register_exception FooError, status: 422
end
{% endhighlight %}

{% include h.html tag="h3" text="2.2 Advanced" a="advanced" %}

The final option `register_exception` accepts is `handler`. Here you can inject your own error handling class that customize `RescueRegistry::ExceptionHandler`. For example:

{% highlight ruby %}
class MyCustomHandler < RescueRegistry::ExceptionHandler
  # self.exception accessible within all instance methods

  def status_code
    # ...customize...
  end

  def error_code
    # ...customize...
  end

  def title
    # ...customize...
  end

  def detail
    # ...customize...
  end

  def meta
    # ...customize...
  end
end

register_exception FooError, handler: MyCustomHandler
{% endhighlight %}

If you would like to use the same custom handler for all errors, override `default_exception_handler`:

{% highlight ruby %}
# app/controllers/application_controller.rb
def self.default_exception_handler
  MyCustomHandler
end
{% endhighlight %}

{% include h.html tag="h2" text="3 Testing" a="testing" %}

This pattern of globally rescuing exceptions makes sense when
running our live application...but during testing, we may want to
raise real errors and bypass this rescue logic.

This is why we turn off error-handling during tests by default:

{% highlight ruby %}
# spec/rails_helper.rb
RSpec.configure do |config|
  config.include Graphiti::Rails::TestHelpers
  # ... code ...

  config.before :each do
    handle_request_exceptions(false)
  end
end
{% endhighlight %}

If you want to turn this on for an individual test (so you can test
error codes, etc):

{% highlight ruby %}
before do
  handle_request_exceptions(true)
end
{% endhighlight %}

<br />
<br />

</div>
