---
layout: post
title:  "Tired of # in your angular app urls? Let's remove them."
date:   2016-04-25 10:18:00
categories: angularjs html5mode
author: vnodecg
---

By default [Angular JS](http://angularjs.org) will route using a '#'.

Your URL will look like `http://example.com/#/test`.

We need to do three things in order to remove the '#' from the URL.
  1. Configuring `$locationProvider`.
  2. Setting up base reference for relative links
  3. Redirecting routes to `index.html`.


### 1. Configuring `$locationProvider`

We need to use `$locationProvider` module and set html5mode to true

{% highlight javascript %}
  angular.module('appname', [])
    .config(['$locationProvider', '$routeProvider',
      function ($locationProvider, $routeProvider) {
        $routeProvider
          .when('/', {
            templateUrl: 'partials/index.html',
            controller: 'MainController'
          })
          .when('/posts', {
            templateUrl: 'partials/posts.html',
            controller: 'PostsController'
          });

        // set html5mode
        $locationProvider.html5mode(true);
      }]);
{% endhighlight %}

[Angular JS](http://angularjs.org) uses HTML5's history api to achieve this.

History API is a standardized way to manipulate browser history  through scripts.

Further reading:

* [History API documentation](https://developer.mozilla.org/en-US/docs/Web/API/History_API)
* [$locationProvider API documentation](https://docs.angularjs.org/api/ng/provider/$locationProvider)


### 2. Setting up base reference for relative links

[Angular JS](http://angularjs.org) needs to know to root of your application which you can set using `base` tag as shown below.

{% highlight html %}
<!doctype html>
<html>
  <head>
    <!-- base tag -->
    <base href="/">
  </head>
  <body>
  </body>
</html>
{% endhighlight %}


### 3. Configuring server to reroute requests to index.html

html5mode works very well as long as you don't reload your browser. Once you reload browser can't find the route specified since server don't know about history based routes. We can fix it by redirecting all requests to `index.html` (startpage of your angular app).


#### Using nginx

{% highlight nginx %}
  server {
    listen 127.0.0.1:80;

    location / {
      root /path/to/your/app;

      # reroute to index.html if $uri is not available
      try_files $uri $uri/ /index.html =404;
    }
  }
{% endhighlight %}
