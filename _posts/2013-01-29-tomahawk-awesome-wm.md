---
layout: post
title: Use Media Keys to control Tomahawk in Awesome WM
---

Nowadays for controling a mediaplayer the [MPRIS specification](http://www.mpris.org) exists,
sadly this interface seems unsupported by awesome. One solution would be to add
some lines to the configuration of `xbindkeys` and to start it in the background. But as
awesome already can handle global keybindings adding these lines to your `.config/awesome/rc.lua`
will transmit the actions of the media keys to Tomahawk (they will work with every
MPRIS2-compatible media, you just need to replace "tomahawk" with the matching name).

***Note:*** This code should be inserted before(!) `root.keys(globalkeys)`.

<script src="https://gist.github.com/4665216.js">/* comment */</script>

