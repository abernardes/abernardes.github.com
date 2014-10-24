---
layout: post
title:  "Serving static assets with Rack"
date:   2014-10-24 09:38:00
---

I think this is a pretty common scenario, and yet I struggled to find an example on how to do it. I was trying to build a single page application using AngularJS on the front-end and [Cuba](http://cuba.is), a minimalistic web framework for Ruby running on top of Rack.

I have a server running a REST API and I want to serve my assets from the same server, to avoid any [CORS](http://en.wikipedia.org/wiki/Cross-origin_resource_sharing) hassle.

This solution will probably apply to any framework running on top of Rack: [Grape](https://github.com/intridea/grape), [Sinatra](http://www.sinatrarb.com/) and even [Rails API](https://github.com/rails-api/rails-api).

```ruby
# This is the config.ru file.

# Serve any requests to the root path ('/') with public/index.html.
use Rack::Static,
  :urls => "/",
  :root => 'public',
  :index => 'index.html'

# Serve requests to assets in /images, /js and /css with public/images, public/js and public/css.
use Rack::Static,
  :urls => ["/images", "/js", "/css"],
  :root => 'public'

# Below is the usual Rack stuff to run your application. Check framework docs for details on this.
require "./my_app"

run MyApp
```

Notes:

* One thing that I like about this approach is that it only changes the way Rack handle requests to `/`, `/images`, `/css` and `/js`. Requests to any other path will be handled by your application.
* This approach uses the `Rack::Static` middleware. There's a pretty decent introduction to Rack middlewares [here](http://stackoverflow.com/questions/2256569/what-is-rack-middleware).

Hope that helps! Cheers!

Thoughts? Comments? Post them to [this Reddit thread](#) or hit me up on [twitter](http://twitter.com/abernardes).
