---
layout: post
title: Using Tomahawk resolvers in node.js
---

*tl;dr: I wrote a node.js module so that you can use packaged Tomahawk AXE archives in your node.js application for querying music services with a unified interface, see [node-tomahawkjs](https://npmjs.org/package/tomahawkjs)*

During MusicHackDay Paris 2013, I made my hack TomaWall that scrapes your Twitter feed in search for music linkes posted by your peers.
Its main task is to detect links to songs and then transform them into a tuple of `(artist, song)` which then can be used to built up a playlist.
As this is the same task for each music service that needs to be done in a different way for each music service, I added this functionality to the Tomahawk resolvers.
In fact this task defined as a function `urlLookup: (url) → (artist, song)` is kind of an inverse of Tomahawk's main function `(urlLookup⁻¹ ≈) resolve : (artist, song) → (url)` which looks for a streamable URL given an artist and a song title.

But the usage of Tomahawk resolvers were at that time limited to either use in Tomahawk itself or by utilising the C++-library *libtomahawk*.
Both possibilties were not feasible for integration in a simple node.js webapp.
As the Tomahawk resolvers are mostly written in JavaScript, I thought that they should be able to run in a node.js environment.
Sadly due to the design of these resolver they cannot be simple included like a standard node.js module, but I implemented the same runtime that is provided in Tomahawk Player.

The resulting library [node-tomahawkjs](https://npmjs.org/package/tomahawkjs) can be used in any node.js application to utilise the full power of the Tomahawk resolver.
A full guide is given in the [README](https://github.com/xhochy/node-tomahawkjs/blob/master/README.md) but a very common task will involve resolving a `(artist, song)` tuple to a streamable URL.
The use a resolver, we need to load the axe package and create a new instance. As Tomahawk resolvers have to be run in their own JavaScript context, the `getInstance` function includes this in its callback too.

{% highlight javascript %}
var TomahawkJS = require('tomahawkjs');

TomahawkJS.loadAxe(pathtoaxe, function(err, axe) {
  // TODO: Check for error in err
  // After load the axe, we most likely want to have an instance of the resolver
  axe.getInstace(function(err, instance_context) {
    // TODO: Check for error in err
    var instance = instance_context.instance;
    // Each Resolver instance runs in its own global JavaScript context
    var context = instance_context.context;
    // Start the instance
    instance.init();
    // We can now use the instance
  });
});
{% endhighlight %}

Given this instance, we now have to define a handle if a new resolving result is found.
A simple handler can for example just log the URL to the commandline:

{% highlight javascript %}
context.on('track-result', function (qid, result) {
  console.log('Found a new streamable URL for request ' + qid + ': ' + result.url);
});
{% endhighlight %}

Given we want to hear to some Skirllex, we then simply query this resolver via:

{% highlight javascript %}
instance.resolve(1, 'Skrillex', '',  'Bangarang feat Sirah')
{% endhighlight %}

Which leads to the following commandline output that could be used in an application to play that song.

<pre>Found a new streamable URL for request 1: http://api.soundcloud.com/tracks/45719017/stream.json?client_id=</pre>
