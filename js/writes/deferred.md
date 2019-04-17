---
layout: page
---

{% include js-header.html %}
{% include js-toc.html %}

<div markdown="1" class="col-md-8 col-md-offset-1">
### Deferred Action

If your update or destroy action takes a long time then the server can respond with status code `202 Accepted` and include background job object in the payload. 

Example response:
<div markdown="1" class="code-tabs">
  {% highlight http %}
HTTP/1.1 202 Accepted
Content-Type: application/vnd.api+json

{
  "data": {
    "type": "background_jobs",
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "attributes": {
      "status": "pending"
    }
  }
}
  {% endhighlight %}
</div>

You will need to give the model object a callback called `onDeferredDestroy` or `onDeferredUpdate`. Spraypaint will then call your callback with the deserialized object included in the payload.

{% include js-code-tabs.html %}
<div markdown="1" class="code-tabs">
  {% highlight typescript %}
let person = new Person({ firstName: 'Jane' })
person.onDeferredUpdate = (job: any) => {
  handleBackgroundJob(job);
}
person.save()

person.onDeferredDestroy = (job: any) => {
  handleBackgroundJob(job);
}
person.destroy()
  {% endhighlight %}

  {% highlight javascript %}
const person = new Person({ firstName: 'Jane' });
person.onDeferredUpdate = (job) => {
  handleBackgroundJob(job);
};
person.save();

person.onDeferredDestroy = (job) => {
  handleBackgroundJob(job);
};
person.destroy();
  {% endhighlight %}
</div>


<div class="clearfix">
  <h2 id="next">
    <a href="{{site.github.url}}/js/middleware">
      NEXT:
      <small>Middleware</small>
      &raquo;
    </a>
  </h2>
</div>
