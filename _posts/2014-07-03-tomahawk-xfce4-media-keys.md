---
layout: Post
title: How to get global media keys support for Tomahawk in XFCE4
---


Although there seems to be no native support for controlling a media player via
the [MPRIS specification](http://www.mpris.org) in [XFCE](http://www.xfce.org/), you
can still set up global shortcuts to use the media keys on your keyboard to control
Tomahawk regardless of which application currently has focus.

To add the shortcuts, go into the XFCE settings menu and open the *Keyboard* settings.
Select there the *Applications shortcuts* tab(see image below).
Now add an entry each for the following combinations of *Command* and *Shortcut* (you will need to enter first the command, then press the respective key):

* `qdbus org.mpris.MediaPlayer2.tomahawk /org/mpris/MediaPlayer2 org.mpris.MediaPlayer2.Player.PlayPause` **XF86AudioPlay**
* `qdbus org.mpris.MediaPlayer2.banshee /org/mpris/MediaPlayer2 org.mpris.MediaPlayer2.Player.Next` **XF86AudioNext**
* `qdbus org.mpris.MediaPlayer2.banshee /org/mpris/MediaPlayer2 org.mpris.MediaPlayer2.Player.Previous` **XF86AudioPrev**
* `qdbus org.mpris.MediaPlayer2.banshee /org/mpris/MediaPlayer2 org.mpris.MediaPlayer2.Player.Stop` **XF86AudioStop**

After setting this, it should look like the following (many thanks to [Lorenz](https://4z2.de/) for the screenshot):

![XFCE4 Applications shortcuts dialog](/images/xfce-tomahawk.png)

Small note: This will work with every MPRIS2-compatible media player,
 you just need to replace "tomahawk" with the matching name.

