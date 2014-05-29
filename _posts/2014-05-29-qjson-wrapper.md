---
layout: post
title: Replace QJson with Qt's own JSON handling in Qt5
---

*tl;dr: A simple wrapper to use QJson for Qt4 and the built-in JSON parser for Qt5 so that QJson is not required if built with Qt5: [qjson-qt5json-wrapper](https://github.com/xhochy/qjson-qt5json-wrapper) (MIT-licensed, no `#ifdef` in your code).*

One of the biggest hurdles to port an application to Qt5 is that you will have Qt4 versions of your dependencies installed on your system.
In most cases these dependencies are used by other programs that are installed (and probably only available yet) for Qt4.
Linking your Qt5 application to Qt4 libraries may work sometimes but will lead than to random crashes at runtime as major Qt data structures have changed between 4 and 5.
A typical approach to circumvent the problem of linking to Qt4 libraries with a Qt5 binary is to name the library differently w.r.t. the Qt version.
For example Phonon installs `/usr/lib64/libphonon.so.4.7.50` for Qt4 and `/usr/lib64/libphonon4qt5.so.4.7.50` for Qt5.
These different library names are respectively represented in the configuration files (CMakeConfig, pkg-config) needed for the build system so that always the right library is picked up.

With QJson, we could follow the same path in our applications like Tomahawk.
But as Qt5 provides a built-in JSON parser, we do not necessarily need to utilize QJson in a binary that is linked against Qt5.
Although QJson and Qt5's JSON have similar interfaces there are still some difference in their behavior.
For example QJson would serialize QVariantHash directly into JSON data whereas Qt5 refuses to serialize them and we need to convert them first to QVariantMap.

### Usage

For parsing JSON data, the following code snippet was needed for QJson.

{% highlight cpp %}
QByteArray jsonData = …;

bool ok;
QJson::Parser p;
QVariant variant = p.parse(jsonData, &ok);
{% endhighlight %}

With the wrapper this needs to be changed to the very similar expression:

{% highlight cpp %}
QByteArray jsonData = …;

bool ok;
QVariant variant = QJsonWrapper::parseJson(jsonData, &ok);
{% endhighlight %}

For writing JSON data, the following QJson snippet was a usual method.

{% highlight cpp %}
QVariant variant = …;

bool ok;
QJson::Serializer serializer;
QByteArray jsonData = serializer.serialize(variant, ok);
{% endhighlight %}

This can be transformed in the following wrapped code:

{% highlight cpp %}
QVariant variant = …;

bool ok;
QByteArray jsonData = QJsonWrapper::toJson(variant, ok);
{% endhighlight %}

Note: Although we automatically convert QVariantHash to QVariantMap inside `toJson`, it is better if you already pass a QVariantMap.

Additionally to wrapping the JSON in/out functions, we have created Qt5 versions of `qvariant2qobject` and `qobject2qvariant`, so that one can directly serialize QObject to JSON.
These functions are complete reimplementations with the full use of the possibilities in Qt5 and may be less reliable than those of QJson.

