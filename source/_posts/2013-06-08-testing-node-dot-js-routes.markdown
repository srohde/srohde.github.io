---
layout: post
title: "Testing Node.js Express Routes"
date: 2013-06-08 06:47
comments: true
categories: node.js, coffeescript
---

Lately I needed some tests for an [Express](http://expressjs.com/) [Node.js](http://nodejs.org/) project I was working on and I was surprised how easy and minimalilstic the code can look like with [CoffeeScript](http://coffeescript.org). Check out this gist:

{% gist 4229691 %}

I am using [should.js](https://github.com/visionmedia/should.js/) as the assertion library and [mocha](http://visionmedia.github.io/mocha/) as the testing framework. In this example I am testing a login function. My Node.js route (api.coffee) is being imported and then I'll test first if the login function actually exists. After that I am invoking the login function and make assertions against the result. The implemented function signature looks like this:

``` coffeescript
exports.login = (req, res) ->
```

The parameters are request and response. This route is using GET so for the request parameter we'll specify the query parameters.
When the implementation is done with the login execution it'll call `res.send(result)` so it's perfect to hook into that as well and we can do assertions against the result type and value.