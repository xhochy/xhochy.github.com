---
layout: post
title: Beats Music Support in Tomahawk (and the long journey on how we got there)
---

*tl;dr:
With the latest nightlies ([Win](http://download.tomahawk-player.org/nightly/windows/tomahawk-latest.exe), [Mac](http://download.tomahawk-player.org/nightly/mac/Tomahawk-latest.dmg)) you can now use your Beats Music Subscription in [Tomahawk](http://www.tomahawk-player.org/).
To use it just install the [Beats Music Resolver](http://teom.org/axes/nightly/beatsmusic-0.1.1.axe).
Although Beats has a nice API, supporting it was a though cruise through our underlying multimedia stack.*

Some months ago, Beats Music released their [API](https://developer.beatsmusic.com/) to the public.
In contrast to most other music services, you get a direct playback URL without the need for any closed source library that will handle DRM.
Since this was not via plain, simple HTTP, we had some hurdles that I'll mention later.

To try out the Beats Music support ([source on Github](https://github.com/tomahawk-player/tomahawk-resolvers/tree/master/beatsmusic)) in Tomahawk Player, you need to download one of our latest nightlies or build Tomahawk from source:

* [Windows Nightly](http://download.tomahawk-player.org/nightly/windows/tomahawk-latest.exe)
* [Mac OS X (10.6+) Nightly](http://download.tomahawk-player.org/nightly/mac/Tomahawk-latest.dmg)
* [Build instructions for various systems](http://wiki.tomahawk-player.org/index.php/Main_Page)

After installing one of the above versions, grab a packaged version of the [Beats Music Resolver](http://teom.org/axes/nightly/beatsmusic-0.1.1.axe).
To install it, start Tomahawk and go in the menu to *Settings -> Configure Tomahawk…*.
You will now see a list with all services that are officially supported in the recent release of Tomahawk.
As Beats Music needs some functionality that was not part of the last Tomahawk release (that's why you need the nightly), you need to install the resolver manually by selecting [Install from file] and chose the resolver file you just downloaded.
After installing, you should have the following entry in the list of available services:

![Beats Music Resolver listing](/images/beatsmusic-resolver-settings-delegate.png)

To set up the Beats Music support, simply click on the wrench on the right and enter your username/password.
Once you have entered your credentials, simply press [OK] to close the dialog and you are ready to use Beats Music in Tomahawk.
For example, you can now search for tracks using Tomahawk's search bar (located in the top middle of the window) and you will be presented with results from Beats.
Alternatively, you can just open one of the lovely curated playlist from Beats by dragging it from the browser to Tomahawk or passing it on the commandline: `tomahawk <url>`.

If you use other music services in Tomahawk besides Beats Music, Tomahawk will always try to select the best version for a song among those services. 
For example for a certain Norwegian variety show group, their most famous song will be played by a Swedish streaming service whereas all other songs have been selected to be played from Beats Music:

![They seem to be not only good at funny music tracks!](/images/ylvis-resolved.png)

We will integrate some more features of Beats Music into Tomahawk in the future (like My Library) but if you experience problems, don't hesitate and contact us in [IRC on irc.freenode.net in #tomahawk](irc://irc.freenode.net/tomahawk) or open a [new bug](https://bugs.tomahawk-player.org/secure/CreateIssue!default.jspa).

### What had to be done to get Beats beating

As I'm a data science student during daytime and contribute to the network and content resolving parts of Tomahawk as recreational activity, I'm totally new to any multimedia development.
Although this meant that I would have to read through a lot of code to understand the core concepts of the multimedia libraries, it was a very satisfying experience as it now works smoothly.

As Beats Music distributes its music to developers via [RTMP](http://en.wikipedia.org/wiki/Real_Time_Messaging_Protocol), the relevant software projects to look at (for us) are [VLC Player/libvlc](http://www.videolan.org/), [FFmpeg](http://ffmpeg.org/)/[libav](https://libav.org/) and [librtmp](http://rtmpdump.mplayerhq.hu/).
The implementation of RTMP in VLC depends on whether we built with or without librtmp.
As we cannot control how Linux distribution package VLC, we had to take care that we support both.
Because we package our Windows, OS X and Android builds ourselves, we have more a bit more control there.

The first obstacle we had to face is that we can only pass a single URL to the media backend.
This is in contrast to a typical RTMP track that is specified by `location` and `resource` (and possibly even more) like returned from the Beats Music API:

{% highlight json %}
{
    "data": {
        "location": "rtmp://...com",
        "resource": "mp3:/mp3/../downloads/../../../..?..",
        "codec": "MP3",
        "bitrate": 320,
        "type": "audio_asset",
        "refs": {
            "track": {
                "ref_type": "track",
                "id": "tr51760477",
                "display": "Blood On The Tracks"
            }
        }
    },
    "code": "OK"
}
{% endhighlight %}

The documentation on how to concatenate these attributes into an RTMP-URL is sadly quite short.
The normal concatenation for a pair of location and resource is to just write them down, one after the other.
But as a normal location for RTMP is expected to be formed like `rtmp://host/app/appinstance` and Beats Music neither has an app nor an appinstance in their URL, the normal parsing failed.
Digging through librtmp's code, we found out that you can alternatively append location and resource by using `?slist=` as a separator.
With `slist=` we can explicitly specify the resource (also called playpath) and from the remaining part, app and appinstance are then inferred (which are then correctly empty in our case).
In contrast to librtmp, in FFmpeg/libav there was no such facility so that it was impossible to specify this.
After a small chat with the developer, I submitted [a patch](https://github.com/libav/libav/commit/7ce3bd9614717e545af8fb8455032c807e389b78) so that we could use the same scheme when VLC/libav/FFmpeg was compiled without librtmp.

With playback now possible as we can pass an URL describing the track we want to play to the media backend, we faced some stuttering at the start of each song.
Looking at the logs of VLC, it always complained about the not supported video codec "undf".
As we definitely only have a media stream that contains audio and no video, this lead to the suspicion that something in our multimedia stack added video, regardless of the stream's metadata.
This suspicion was partly true as the FLV decoder module of FFmpeg/libav by default always instantiated an audio and a video stream for an RTMP stream.
The problem was further extended as the RTMP provided by Beats did not include any metadata so that we could not know about the existence of audio/video streams until the actual data for them arrived.
Luckily FFmpeg/libav's architecture allowed the addition of the respective streams only if they really appear later on.
With my patches [1](https://github.com/libav/libav/commit/59cb5747ec3c5cd842b94e574c37889521c97cc4), [2](https://github.com/libav/libav/commit/3b18857ab301d2a0b3e86e9d85eed76f0798a29c), [3](https://github.com/libav/libav/commit/a1859032e39d96352687186fd179e1559dea2aca) by default, we indicate from RTMP to the FLV decoder which streams should be actually allocated that were supplied in the RTMP metadata package.
In the case of Beats Music, where we have no metadata, we do not allocate any stream when calling the FLV decoder but peek a bit into the packages.
As only an audio stream is among these packages, now only an audio is forwarded to VLC for that we know the codec so that we have a smooth start.

After this fixing and digging in libav, then there was one last bug in VLC which stopped playback about 1s before the end and spun up the CPU to 100% on one core.
The problem here was that the end of the audio stream was not correctly reported in the VLC stack.
After digging through the cycle with gdb, I reported the malicious section upstream.
This was then promptly [fixed](http://git.videolan.org/?p=vlc.git;a=blobdiff;f=modules/access/avio.c;h=47615e6d939ddded86fc24ea7cddc28b1228532b;hp=98393662220955f09aec7eabf472e68dfd7f44cd;hb=01c1d49be8c2b9eb375327c79145fa74e58c2f21;hpb=5b8095894633bb7148315a54c89daaecaae8c847) so that we now have comfortable playback of Beats Music's audio streams.

At the end I want shout out a big "Thank you" to [j-b](http://jbkempf.com/), [courmisch](http://www.remlab.net/) and especially [wbs/mstorsjo](https://github.com/mstorsjo) who were very helpful on IRC.
As a side note: seeking is not really working, so I will probably dive into the world of multimedia formats and protocols once more.
Although this is not really my specialised area, it is a nice adventure.
