---
layout: post
title:  "Using Inherited Resources with strong parameters"
date:   2013-08-21 19:18:00
---

I just started a new project where I'm going to use Rails 4 and the [inherited_resources](https://github.com/josevalim/inherited_resources) gem to speed up development a little.

Since inherited_resources is not fully compatible with Rails 4's Strong Parameters, the way to whitelist attributes is:

{% highlight ruby %}
class CustomersController < InheritedResources::Base
  protected

  def permitted_params
    params.permit(:customer => [:name, :email, :phone_number])
  end
end
{% endhighlight %}

Notes:

* The method name should always be `permitted_params`
* The way we whitelist is slightly different than the Rails default notation of `params.require().permit()`.

There's an [open pull request](https://github.com/josevalim/inherited_resources/pull/286) to integrate with Strong Parameters and solve the weird notation issue, but since it isn't merged yet, I guess we'll have to stick with this notation for now.