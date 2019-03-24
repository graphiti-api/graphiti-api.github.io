---
layout: page
---

<div markdown="1" class="toc col-md-3">
Remote Resources
================

* 1 [Overview](#overview)
  * [How It Works](#how-it-works)
  * [Customizing](#customizing)
  * [Error Handling](#error-handling)
* 2 [Testing](#testing)
</div>

<div markdown="1" class="col-md-8">

{% include h.html tag="h2" text="1 Overview" a="overview" %}

Too often it seems the thinking is, "🤔 *We want separate services,
we'll figure out our API contract along the way*". Graphiti approaches
from the opposite angle - figure out the API contract, and you get separate services as a side-effect 😃💡

<p align="center">
  <img width="100%" src="https://labs.sogeti.com/wp-content/uploads/2016/11/microservices.gif">
</p>

Resources have a defined query contract, and connect together with [Links]({{site.github.url}}/guides/concepts/links). Put two and two together, and you'll see a Resource doesn't need to be local to a single application. We can have Remote Resources as well:

{% highlight ruby %}
has_many :comments,
  remote: 'http://blog-api.com/api/v1/comments'
{% endhighlight %}

You might want separate applications that are independently deployable,
or you might want to break apart that slow test suite. The draw of
isolated services is clear. Though you should be aware of the tradeoffs
when breaking apart a [Majestic Monolith](https://m.signalvnoise.com/the-majestic-monolith),
Graphiti helps you ***lessen*** those tradeoffs.

The most common service problem I see is a **breakdown in cross-service
communication**:

* 🚫 No consistent or flexible query interface
* 🚫 No consistent error handling
* 🚫 No clear patterns on when and why to separate services
* 🚫 No types or backwards-compatibility checks (unless GraphQL)
* 🚫 A fair amount of [glue code](https://www.apollographql.com/docs/graphql-tools/schema-stitching.html) required

Graphiti was built from the ground up to address all these points. We
have a defined query contract, errors payload, a schema with types and
backwards-compatibility checks, and organize code into RESTful
Resources. Not only can we ***facilitate*** cross-service communication,
we can ***automate*** it.

> Note: Remote Resources are for **read** operations only.

{% include h.html tag="h3" text="How it Works" a="how-it-works" %}

Let's take a simple association:

{% highlight ruby %}
class PostResource < ApplicationResource
  has_many :comments
end
{% endhighlight %}

This would generate a [Link]({{site.github.url}}/guides/concepts/links) for
lazy-loading comments:

{% highlight ruby %}
{
  related: "http://my-api.com/api/v1/comments?filter[post_id]=123"
}
{% endhighlight %}

Critically, **those same lazy-loading parameters are used when
eager-loading**:

{% highlight ruby %}
# under the hood
posts = PostResource.all.data
CommentResource.all(filter: { post_id: 123 })
{% endhighlight %}

OK, and we also know Resources support [any backend]({{site.github.url}}/guides/concepts/backends-and-models), and we can build an [Adapter]({{site.github.url}}/cookbooks/without-activerecord#adapters) if our backend supports common operations like filtering, sorting, and pagination.

So, that means we can build an Adapter that makes an HTTP request to another Graphiti Resource that lives in a separate API. That adapter is built into Graphiti and comes out-of-the-box: `Graphiti::Adapters::GraphitiAPI`

{% highlight ruby %}
class CommentResource < ApplicationResource
  self.remote = "http://my-api.com/api/v1/comments"
  # under-the-hood, this sets:
  # self.adapter = Graphiti::Adapters::GraphitiAPI
end
{% endhighlight %}

This Resource works as normal. We can execute queries:

{% highlight ruby %}
comments = CommentResource.all({
  sort: '-id',
  filter: { active: true }
})

# The model instances are OpenStructs
comments.data # => [#<OpenStruct>, #<OpenStruct>, ...]

# Those models reflect all the properties returned from the API:
comments.data.map(&:author) # => ["Jane Doe", "John Doe", ...]
{% endhighlight %}

And we can sideload just like we always do:

{% highlight ruby %}
class PostResource < ApplicationResource
  # Nothing to see here!
  has_many :comments
end
{% endhighlight %}

We'll still support Deep Querying - let's fetch the Post and its
active comments, ordered by `created_at`:

`/posts?include=comments&sort=comments.created_at&filter[active]=true`

Let's say `CommentResource` has an association to `Author`. If
`AuthorResource` is defined in the remote API, we can fetch it as
normal - no special configuration needed to fetch the `Post`, `Comment`s
and `Author`s in a single request.

But maybe only `CommentResource` is remote, and `Authors` are local.
We need only define the association locally:

{% highlight ruby %}
class CommentResource < ApplicationResource
  self.remote = "http://my-api.com/api/v1/comments"

  belongs_to :author
end
{% endhighlight %}

Let's say we need to tweak the display of a property coming from the
remote API. Again, works just like normal:

{% highlight ruby %}
class CommentResource < ApplicationResource
  self.remote = "http://my-api.com/api/v1/comments"

  attribute :body, :string do
    @object.body.truncate(100)
  end
end
{% endhighlight %}

You only need to define attributes when overriding this logic -
otherwise we'll take them directly from the API response. This means you
don't have to update two repos and coordinate deploys - as soon as you
add a property to the remote API and deploy it, it will be reflected in
the local API response.

For the typical use case, we don't even *need* to create this Resource
class. The sideload definition accepts a `remote:` option, which will
create a Remote Resource under-the-hood:

{% highlight ruby %}
class PostResource < ApplicationResource
  has_many :comments, remote: 'http://my-api.com/api/v1/comments'
end

# Equivalent to:
#
# class PostResource < ApplicationResource
#   has_many :comments
# end
#
# class CommentResource < ApplicationResource
#   self.remote = 'http://my-api.com/api/v1/comments'
# end
{% endhighlight %}

{% include h.html tag="h3" text="Customizing"
a="customizing" %}

We use [Faraday](https://github.com/lostisland/faraday) under-the-hood,
which allows for various adapters and middleware. In addition:

{% include h.html tag="h4" text="Configure Timeout" a="configure-timeout" %}

{% highlight ruby %}
class CommentResource < ApplicationResource
  self.remote = "..."

  # Customize faraday timeout
  self.timeout = 10
  self.open_timeout = 20
end
{% endhighlight %}

{% include h.html tag="h4" text="Configure Request" a="configure-request" %}

{% highlight ruby %}
class CommentResource < ApplicationResource
  self.remote = "..."

  def make_request(url)
    # request here is from Faraday:
    #
    # conn.get do |req|
    #   yield req
    # end
    #
    super do |request|
      request.headers["Custom"] = "Header"
    end
  end
end
{% endhighlight %}

{% include h.html tag="h4" text="Configure Headers" a="configure-headers" %}

By default we're going to *forward* the `Authorization` header of the request to the remote API. To override the default headers sent:

{% highlight ruby %}
# app/resources/comment_resource.rb
def request_headers
  { "Some-Foo" => "bar" }
end
{% endhighlight %}

{% include h.html tag="h3" text="Error Handling" a="error-handling" %}

If the remote API has an error, we want to re-raise that same error. But
unless you've enabled [displaying raw errors](/guides/concepts/error-handling#displaying-raw-errors), we won't be able to - the only information we have is what's returned from the API.

You're encouraged to display raw errors when an internal or privileged
user:

{% highlight ruby %}
rescue_from Exception do |e|
  handle_exception(e,  show_raw_error: current_user.developer?)
end
{% endhighlight %}

If you do this, we'll be able to re-raise the original error, including
stacktrace. If raw errors are not enabled, we'll raise whatever
information is given.

Both styles will be wrapped in `Graphiti::Errors::Remote`, so you can
differentiate between a local error and a remote one.

{% include h.html tag="h2" text="Testing" a="testing" %}

When testing a remote resource, we need to mock the API request and
response. Graphiti gives you a spec helper to do just that -
`include_context "remote api"`:

{% highlight ruby %}
describe 'comments' do
  include_context 'remote api'

  let(:api_response) do
    {
      id: '1',
      type: 'comments',
      attributes: { body: 'hello' }
    }
  end

  it 'does something' do
    mock_api('http://my-api.com/api/v1/comments', api_response)
    # ... test ...
  end
end
{% endhighlight %}

This shows all the pieces needed to test remote APIs. We want to test

* The correct URL is hit
* When given a valid response, the rest of the flow works as expected.

Here's a slightly longer version, showing that `Post` can sideload
`Comment`s:

{% highlight ruby %}
describe 'sideloading' do
  describe 'comments' do
    include_context 'remote api'

    let!(:post) { create(:post) }

    let(:api_response) do
      {
        id: '789',
        type: 'comments',
        attributes: { body: 'hello' }
      }
    end

    before do
      params[:include] = 'comments'
    end

    it 'does something' do
      url = "http://my-api.com/api/v1/comments"
      url += "?filter[post_id]=#{post_id}"
      mock_api(url, api_response)
      render
      sl = d[0].sideload(:comments)
      expect(sl.map(&:id)).to eq(['789'])
      expect(sl.map(&:jsonapi_type).uniq)
      .to eq(['comments'])
    end
  end
end
{% endhighlight %}

