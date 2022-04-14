---
layout: page
---

<div markdown="1" class="toc col-md-3">
Links
==========

* 1 [Overview](#overview)
  * [Linking Relationships](#linking-relationships)
* 2 [Resource Endpoints](#resource-endpoints)
  * [Validation](#validation)
* 3 [Configuration](#configuration)
  * [Autolinking](#autolinking)
  * [Endpoint Validation](#endpoint-validation)
  * [Links-On-Demand](#links-on-demand)
  * [Pagination Links](#pagination-links)
  * [Custom Endpoint URLs](#custom-endpoint-urls)

</div>

<div markdown="1" class="col-md-8">

<p align="center">
  <img width="100%" src="https://pbs.twimg.com/media/CeKlfWPUIAATWZZ.jpg">
</p>

{% include h.html tag="h2" text="1 Overview" a="overview" %}

Links are useful for:

* Discoverability
* Lazy-loading
* Hiding implementation details

Let's say we're loading a Post and its "Top Comments". On Day One, we
might **eager load** the data like so:

`/posts/123?include=top_comments`

This fetches all the data in a single request. But after some UI
testing, we decide to add a "show comments" button. This way our
page can load more quickly - it only needs to load the Post
initially, and loading Top Comments can be deferred. We
want to **lazy load** the relationship.

How would we do this? We *could* bake this logic into our next request:

`/comments?filter[post_id]=123&filter[upvotes][gte]=100`

But this requires the client to have knowledge of what a "Top Comment"
is. If this logic ever changed, we'd have to update every client - our
desktop app, mobile apps, reports, etc… Not to mention, third parties who
just want to display Top Comments are required to have this knowledge
and update their implementations as well.

Maybe we could hide what "Top Comment" means with a special endpoint:

`/top_comments?filter[post_id]=123`

But now clients *need to know* to hit this special endpoint instead of
the normal `/comments` endpoint. How would they know? What if
`top_comments` had special caching rules and *shouldn't* be used for
this purpose?

The main problem here is **there is no way to guarantee our lazy-loaded data will
match our eager loaded data**. Whether we fetch the Post and its Top
Comments in a single request, or lazy-load that data in a separate
request, the same data should always be returned.

<p align="center">
  <img width="100%" src="/assets/img/rest2.gif">
</p>

[Links](http://jsonapi.org/format/#document-links) solve this problem. When we fetch the Post, the `top_comments`
relationship will contain a URL. Clients can simply follow that URL to
lazy-load the same data. We can now change the definition of a Top
Comment - 500 upvotes, factor in recency, apply downvotes - and no
clients need to change. They simple continue to follow a Link.

{% include h.html tag="h3" text="1.1 Linking Relationships" a="linking-relationships" %}

When defining a relationship, we get a Link for free:

{% highlight ruby %}
class PostResource < ApplicationResource
  has_many :comments
end
{% endhighlight %}

> `/comments?filter[post_id]=123`

And when customizing a relationship with `params`, our Link will be
updated:

{% highlight ruby %}
has_many :comments do
  params do |hash|
    hash[:filter][:upvotes] = { gte: 100 }
  end
end
{% endhighlight %}

> `/comments?filter[post_id]=123&filter[upvotes][gte]=100`

Note: if you use the `scope` block directly, it may cause incorrect
links. Avoid using `scope` directly and instead use `params` and
`pre_load` if possible.

To manually generate a Link:

{% highlight ruby %}
has_many :comments do
  link do |post|
    helpers = Rails.application.routes.url_helpers
    helpers.comments_url(params: { filter: { post_id: post.id } })
    # or
    # http://example.com/api/v1/comments?filter[post_id]=123
  end
end
{% endhighlight %}

To avoid a Relationship Link altogether:

{% highlight ruby %}
has_many :comments, link: false
{% endhighlight %}

{% include h.html tag="h2" text="2 Resource Endpoints" a="resource-endpoints" %}

To generate links, we need to associate a Resource to a URL. By default,
this happens automatically:

{% highlight ruby %}
class ApplicationResource < Graphiti::Resource
  # ... code ...
  self.endpoint_namespace = '/api/v1'
end

class PostResource < ApplicationResource
  # under the hood:
  primary_endpoint 'posts',
    [:index, :show, :create, :update, :destroy]
end
{% endhighlight %}

Which would generate links to `/api/v1/posts`.

{% include h.html tag="h3" text="2.1 Validation" a="validation" %}

Associating a Resource to an Endpoint serves two purposes. We've gone
over link generation. But we also want to make sure we're not linking to
something that doesn't actually exist. That's why we perform **Endpoint
Validation**.

If we tried to access the above resource at a `/comments` endpoint:

{% highlight ruby %}
class CommentsController < ApplicationController
  def index
    PostResource.all(params)
    # ...
  end
end
{% endhighlight %}

We'd get a `Graphiti::Errors::InvalidEndpoint` error. Endpoint
validation ensures that our auto-generated Links are actually valid.

To change the endpoint associated to a Resource:

{% highlight ruby %}
primary_endpoint 'special_posts', [:index, :show]
{% endhighlight %}

Or to alter only the **path**:

{% highlight ruby %}
self.endpoint[:path] = 'special_posts'
{% endhighlight %}

Or to alter only the **actions** supported:

{% highlight ruby %}
self.endpoint[:actions] = [:index, :show]
{% endhighlight %}

A resource may be accessible by multiple endpoints. Maybe `PostResource`
is also used at `/top_posts`. We want to keep all auto-generated links
pointing to `/posts` (the primary endpoint), but *allow* accessing
`PostResource` from the `/top_posts` endpoint:

{% highlight ruby %}
secondary_endpoint '/top_posts', [:index]
{% endhighlight %}

{% include h.html tag="h2" text="3 Configuration" a="configuration" %}

{% include h.html tag="h3" text="3.1 Autolinking" a="autolinking" %}

To turn off automatically generated links:

{% highlight ruby %}
class ApplicationResource < Graphiti::Resource
  self.autolink = false
end
{% endhighlight %}

{% include h.html tag="h3" text="3.2 Endpoint Validation" a="endpoint-validation" %}

To turn off Endpoint Validation:

{% highlight ruby %}
class ApplicationResource < Graphiti::Resource
  self.validate_endpoints = false
end
{% endhighlight %}

{% include h.html tag="h3" text="3.3 Links-on-Demand" a="links-on-demand" %}

To only render links when requested in the URL with `?links=true`:

{% highlight ruby %}
Graphiti.configure do |c|
  c.links_on_demand = true
end
{% endhighlight %}

{% include h.html tag="h3" text="3.4 Pagination Links" a="pagination-links" %}

Requesting big collections can result into slow responses sometimes. In order to avoid this, you could use [pagination](https://jsonapi.org/format/#fetching-pagination). It'll break your response into smaller pieces that will make your server responds faster. Paginations links can be present in your response in the following ways:

{% include h.html tag="h4" text="3.4.1 Showing by default" a="pagination-links-showing-by-default" %}

With this configuration, all the responses will return the pagination links

{% highlight ruby %}
Graphiti.configure do |c|
  c.pagination_links = true
end
{% endhighlight %}

{% include h.html tag="h4" text="3.4.2 When requested" a="pagination-links-when-requested" %}

You can showing the pagination links when it was requested in the URL with `?pagination_links=true`

{% highlight ruby %}
Graphiti.configure do |c|
  c.pagination_links_on_demand = true
end
{% endhighlight %}

Pagination links won't show up for *#show* actions.

{% include h.html tag="h3" text="3.5 Custom Endpoint URLs" a="custom-endpoint-urls" %}

To change the URL associated with a Resource:

{% highlight ruby %}
class PostResource < ApplicationResource
  # Most commonly seen in ApplicationResource
  self.endpoint_namespace = '/api/v1'

  primary_endpoint '/posts', [:index, :show]
  # OR
  self.endpoint[:path] = '/posts'
  # OR
  self.endpoint[:actions] = [:index, :show]
end
{% endhighlight %}

To generate a Relationship Link manually:

{% highlight ruby %}
has_many :comments do
  link do |post|
    helpers = Rails.application.routes.url_helpers
    helpers.comments_url(params: { filter: { post_id: post.id } })
    # or
    # http://example.com/api/v1/comments?filter[post_id]=123
  end
end
{% endhighlight %}

</div>
