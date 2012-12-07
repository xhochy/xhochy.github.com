---
layout: post
title: ktoblzcheck bindings for Node.JS
---

Checking the correctness of a combination of bank identification number (BLZ)
and account number is a complicated task in Germany. There are about 140 
different methods how this combination is checked depending on the bank from
which the combination originates. A library which solves this problem is
[ktoblzcheck](http://ktoblzcheck.sourceforge.net/). At the moment you could
use it simply as a C++ library or with the python wrapper but there isn't
a simple way to use it in Node.JS. Due to this lack of support I wrote 
[node-ktoblzcheck](https://github.com/xhochy/node-ktoblzcheck). Until now
it supports the two main functions I needed.

If you know a BLZ you could get the name of the corresponding bank via
{% highlight javascript %}
ktoblzcheck.getNameForBLZ(37010050)
{% endhighlight %}

If you are supplied a combination of a BLZ and an account number, checking
if they are valid together is done with one handy call:
{% highlight javascript %}
ktoblzcheck.checkBLZAccount(50010517, 0648489890)
{% endhighlight %}
For a correct combination the above code should return 0.

Using [node-ktoblzcheck](https://github.com/xhochy/node-ktoblzcheck) is as easy
as calling
{% highlight bash %}
$ npm install ktoblzcheck
{% endhighlight %}
