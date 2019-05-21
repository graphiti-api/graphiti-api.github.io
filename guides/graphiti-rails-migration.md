---
layout: page
---

Migrating to graphiti-rails
===========================

First, bump to Graphiti 1.2 - everything should work as-is, but you'll
start seeing deprecation notices.

Then add `graphiti-rails` to your `Gemfile`:

```ruby
gem 'graphiti-rails', '~> 0.1'
```

You'll see slightly different deprecation notices at this point. To fix:

* Remove `include Graphiti::Rails` from `ApplicationController`
* Remove `include Graphiti::Responders` from `ApplicationController`
* Remove the `register_exception` for `404` errors if you have one.
* Add `include Graphiti::Rails::Responders` from `ApplicationController`
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
