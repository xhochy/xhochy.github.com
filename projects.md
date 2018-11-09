---
feature_image: "https://picsum.photos/2560/600?image=1026&blur"
feature_text: |
  ## Projects
  Software projects I am currently part of or have been working on.
---

**[Active Projects](#active-projects)** \| **[Past open-source projects](#past-open-source-projects)** \| **[Past (small) projects and weekend hacks](#past-small-projects-and-weekend-hacks)**

Asides the products and services that are the customer-facing deliverables,
I devote some time to opensource projects like Apache Arrow or have spent some
(private) time building small hacks. In addition to the projects I'm actively
involved in, this page also lists my past projects and involvements.

The history of projects I have worked has a tight coupling with my CV. While
in recent years I have focused on topics in the world of Data Science and Data
Engineering, I was involved in a number of MusicTech projects during my times
as a computer science student. Intially, while I still was at school, I made
my first baby steps in (publically visible) software development with some
web-based projects.

## Active Projects

#### [Apache Arrow](https://arrow.apache.org/)

*since March 2016*

The landscape of `Data X` products is very heterogeneous, most of Big Data analytics happens in the Java/Scala-based world whereas machine learning and data aanlysis are mostly done with Python and R. Building data products requires integrating these different stacks together in an efficient manner. Apache Arrow
defines in-memory columnar structures that are already implemented and used in over 10 languages. In addition to the data structures, it provides libraries for efficient computations and communications based on these data structures.

Since the beginning of the project, I have been actively involved in the development of the C++ implementation and the Python bindings on top of it. My main focus of work has been on the Parquet integration, packaging setup, Java interoperability and the building of a community around the project. I'm also part of the project management committee (PMC) of the Apache Arrow project.

Used technologies: `Python`, `C++`, `pandas`, `jvm`

#### [Apache Parquet](https://parquet.apache.org/)

*since March 2016*

Apache Parquet has been the de-facto standard in the Hadoop ecosystem for efficient file storage of columnar data. While being used as the preferred option in the JVM-based Big Data ecosystem, it lacked support for Python and C++. As part of the project, I have started contributing some missing bits to the read path and then wrote a major chunk of the code for writing Parquet files in C++. Together with other contributors we then implemented a Parquet-to-Arrow read in C++ to also bring full support of the Parquet format to Python. At the moment, I'm a maintainer of the C++ codebase and part of the Project Management Committee (PMC) of the Apache Parquet project.

#### [fletcher](https://github.com/xhochy/fletcher)

*since March 2018*

While Apache Arrow already served as a viable library for accelerating I/O in Python at the beginning of 2018, it was yet lacking any analytical capabilities. When `pandas` introduced the concept of `ExtensionArray`, an interface to use its algorithms on top of a user-defined storage, we directly implemented the basic interface with Apache Arrow as the backing storage. This serves as a prototype on how a future pandas version could use Arrow for its memory instead of using NumPy.

Although pandas provides a huge set of algorithms on top of the `ExtensionArray` interface, it requires the underlying storage to support some basic operations. As not all of them are supported in `pyarrow`, we use `numba` to implement these operations in pure Python but take advantage of the just-in-time compiler to get native performance.

Used technologies: `Apache Arrow`, `pandas`, `numba`

## Past open-source projects

#### [Tomahawk Player](https://www.tomahawk-player.org/)

*December 2012 - April 2015*

Tomahawk decouples the name of the song from the source it was shared from - and fulfills the request using all of your available sources.

My major contributions to Tomahawk include the extension of the networking backend to improve connection handling between two player instances and the addition of IPv6 support. Furthermore I added the Trending section which gathers the Charts among your peers as well as determines which Playlists and Artists are trending.

In the end, I devoted my Tomahawk time to improve the functionality of the resolvers, Tomahawk's plugin system which gives unified acces to music services. Recent advances here are the support of a unified API of transforming an music service URL into more useful information like the content of a playlist it represents.

Used techniques: `C++`, `Qt`, `JavaScript`, `CMake`

#### [Gentoo](https://www.gentoo.org/)

*spring 2011 - spring 2015*

As Gentoo was the operating system on my laptop for a long time, I contributed patches and packages to the Qt, KDE and Science overlay for software that had a recent release or a change in the build system in the upstream source tree.

My major work here was adjusting packages for the support of building them with Qt4 and/or Qt5. This did not only involve adjusting the packages in Gentoo itself but also meant that upstream sources needed to be patched as many packages were needed in parallel in Qt4 and Qt5 installed but upstream had sometimes not addressed this problem.

Used techniques: `ebuild`, `bash`, `patch/diff`, `CMake`

#### [node-tomahawkjs](https://github.com/xhochy/node-tomahawkjs), [node-tomahk](https://github.com/xhochy/node-tomahk), [hubot-tomahk](https://github.com/xhochy/hubot-tomahk)

*July 2013 - 2015*

As an addition to the Tomahawk ecosystem, I created some Node.JS packages which build upon technologies which were born out of the player. This includes (in the order of the title): a wrapper library to use the content/service resolvers from Tomahawk unmodified in Node, a wrapper to the toma.hk API and a hubot plugin that responds with the metadata to music links it detects in a chat.

Used techniques: `JavaScript`, `CoffeeScript`, `Hubot`, `Node.JS`

#### [VLC / VideoLAN](http://www.videolan.org/), [libav](https://libav.org/), [Phonon(-VLC)](http://phonon.kde.org/)

*March 2014 - 2015*

As another direction started from my work on Tomahawk, some of my time went into fixing various problems in our multimedia stack. In Tomahawk, we use Phonon as our playback library with VLC as the prefferred backend. In VLC, most of the codecs are provided through libav/ffmpeg.

The most impactful contribution in this stack was the improvement of audio-only playback encapsulated in the rtmp:// protocol. This included fixing the detection of the number of media streams present in an rtmp stream that did not send metatdata. Furthermore we had to ensure that all possible backends support the same URL schema so we could pass this information seamlessly through all layers. (This was required for Beats Music support in Tomahawk)

Used techniques: `C/C++`, `Wireshark / tcpdump`, `rtmp`

#### [Rainpress](https://github.com/xhochy/rainpress)

*2007 - 2008*

Before the trend for ever larger website designs began, the sizes of CSS files were normally not a problem. Still, in these days bandwidth was more of a bottleneck than it was today. CSS were still manually written back then and not compiled with something like SASS. Rainpress was a small tool I have written that applied a set of rules that shaved off some bytes by removing comments and replacing typical style expressions by shorter but equivalent forms. It was inspired by the newly appearing libraries that did the same on JavaScript files.

Used techniques: `ruby`, `CSS`

#### [Schoorbs](https://github.com/xhochy/schoorbs)

*2007 - 2008*

Schoorbs ("school room booking system") was my first larger software project I did totally on my own. As my "Facharbeit" in school, I have taken another opensource room booking solution and adapted it to the school's needs. This involved fixing some security issues along the way but also largely adding features that made it useful in a scholar fashion.

Used techniques: `php`, `JavaScript`, `SQL`

## Past (small) projects and weekend hacks

These projects were built in a short period of time, e.g. a HackDay or a University course and may not be developed anymore thereafter. Nevertheless I will try to keep them running and everything uploaded for documentation.

#### A Book of Music 

*MusicHackDay London, December 2013*

Last summer I read relaxed on my couch the biography of Blur bassist Alex James. Despite being a good book, I really wanted to smell the cheese he consumes and produces and the music Blur made over time.

For reading I really like to use my eBook reader as it is more handy and the lighting is better than any lamp at night. As it does not has a headphone jack or a speaker, it cannot play music files. Even if it had, it would really prefer to use my quality speakers instead of a crappy built-in speaker. Neither I want to be wired with my reader to speaker nor do I want to walk to my computer to lookup a track and play it.

To solve this problem, A Book of Music parses an ebook and inserts links to its webservice to start the playback directly from your eBook reader but tracks will be played on your favorite machine which is connected to high quality speakers. As an input source for eBooks I'm using Wikipedia and preindexed all sites that describe songs to have a knowledge which links point to music.

Used techniques: `JavaScript`, `CoffeeScript`, `Wikipedia`, `ePUB`

#### TomaWall 

*MusicHackDay Paris, April 2013*

Just one week after the introduction of Twitter Music, MHD Paris took place. On of the major complaints I heard from my peers about it was the missing feature to just play the tracks that appear in your timeline, even if the are from Soundcloud, Deezer or other services not included in the initial Twitter Music Release. This hack simply solves this problem by parsing your Timeline and providing all posted tracks in a toma.hk playlist. (This hack lead later on to the creation of the UrlLookup capability in Tomahawk resolvers)

Used techniques: `JavaScript`, `CoffeeScript`, `Twitter`, `Node.JS`, `heroku`

#### Oh my Youth!

*MusicHackDay London, November 2012*

As my first experience on a MusicHackDay I did one of the typical hacks that people do at their first event like this: an automatic playlist generator. It takes as input two of your favorite artist when your were really young and optionally a release year. With this information it searches the MusicBrainz Database to find a compilation that features both artists and was released in the specified year. Altough this might not be the music you still hear today, childhood memories will definitely come back!

Used techniques: `JavaScript`, `Spotify`, `MusicBrainz`
