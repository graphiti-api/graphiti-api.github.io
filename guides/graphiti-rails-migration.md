---
layout: page
---

Migrating to graphiti-rails
===========================

You can continue using Graphiti 1.0 without this, but the migration
is only a few simple steps.

## Application Code

First, bump to Graphiti 1.2 - everything should work as-is, but you'll
start seeing deprecation notices.

Then add `graphiti-rails` to your `Gemfile`:

```ruby
gem 'graphiti-rails', '~> 0.1'
```

You'll see slightly different deprecation notices at this point. To fix:

* Remove `include Graphiti::Rails` from `ApplicationController`
* Remove `include Graphiti::Responders` from `ApplicationController`
* Remove the `register_exception` for `404` errors if you have one. Keep
anything else, but this is now built-in by default.
* If you're using [responders](https://github.com/plataformatec/responders), Add `include Graphiti::Rails::Responders` from `ApplicationController`.
* Remove the `rescue_from` block.

Finally, if you were passing the `show_raw_error` option to
`#handle_exception`, replace this with `#show_detailed_exceptions?`:

```ruby
def show_detailed_exceptions?
  current_user.admin?
end
```

> Keep in mind, we'll always show detailed exceptions in development
> mode, per Rails conventions. If this is confusing or not desirable,
> set `config.consider_all_requests_local = false` in
> `config/environments/development.rb`

> If you're not running Rails in API-only mode, be sure to set
> `config.debug_exception_response_format = :api` in
> `config/application.rb`

## Testing

Add `config.include GraphitiSpecHelpers::Sugar` to
`spec/rails_helper.rb`.

Replace

{% highlight ruby %}
config.before :each do
  GraphitiErrors.disable!
end
{% endhighlight %}

With

{% highlight ruby %}
config.before :each do
  handle_request_exceptions(false)
end
{% endhighlight %}

Change any calls to `GraphitiErrors.enable!/disable!` with
`handle_request_exceptions(true|false)`.
