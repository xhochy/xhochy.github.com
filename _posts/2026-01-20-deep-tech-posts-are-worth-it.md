---
layout: post
title: >
  Deep technical blog posts are still worth it in the AI age
feature_image: "/images/wendeltreppe-fotografie-5Z5As4cdE5k.jpeg"
---

AI summaries mean that you need to click less on search results to find the actual information you're looking for. 
But for me, as someone who writes a lot of deep tech in blog posts,
I have seen that it's still worth it to write the blog post,
and it's still worth it for the readers you want to read your articles to actually read those blog posts and get there with AI.
You won't reach millions, but you still reach the people you really care about.

In my example, I have triggered a bug in the macOS Linker in the conda-forge setting in [mysql-feedstock#115](https://github.com/conda-forge/mysql-feedstock/pull/115).
This is a hard-to-debug issue because your compiler is segfaulting and not reporting an error.
In my case, the segmentation fault sadly led only to a very short stacktrace:

```
% lldb "/Users/…/arm64-apple-darwin20.0.0-ld"
(lldb) target create "/Users/…/arm64-apple-darwin20.0.0-ld"
Current executable set to '/Users/…/arm64-apple-darwin20.0.0-ld' (arm64).
(lldb) run -demangle -lto_library …
* thread #1, queue = 'com.apple.main-thread', stop reason = EXC_BAD_ACCESS (code=1, address=0x0)
    frame #0: 0x000000019d488a80 libsystem_platform.dylib`_platform_strncmp$VARIANT$Base + 176
libsystem_platform.dylib`_platform_strncmp$VARIANT$Base:
->  0x19d488a80 <+176>: ldr    q0, [x0], #0x10
    0x19d488a84 <+180>: ldr    q1, [x1], #0x10
    0x19d488a88 <+184>: cmeq.16b v1, v0, v1
    0x19d488a8c <+188>: and.16b v0, v0, v1
Target 0: (arm64-apple-darwin20.0.0-ld) stopped.
(lldb) bt
* thread #1, queue = 'com.apple.main-thread', stop reason = EXC_BAD_ACCESS (code=1, address=0x0)
  * frame #0: 0x000000019d488a80 libsystem_platform.dylib`_platform_strncmp$VARIANT$Base + 176
    frame #1: 0x00000001001c0538 arm64-apple-darwin20.0.0-ld`ld::passes::dylibs::doPass(Options const&, ld::Internal&) + 412
    frame #2: 0x0000000100013c04 arm64-apple-darwin20.0.0-ld`main + 732
    frame #3: 0x000000019d0b5d54 dyld`start + 7184
(lldb)
````

Given the sparse information, I felt quite lost.
I am used to debugging foreign software, but this sadly didn't yield any useful information.
Still, in my effort to leverage the newly available LLM-based tools more, I pasted the stacktrace with some additional information into Gemini and Perplexity.
At first glance, neither of them led to useful responses, independently of how much information I gave them.
At a second look, the following (simpler) prompt [on Perplexity](https://www.perplexity.ai/search/i-have-a-segmentation-fault-he-S16M9qvcRGStpSc7Ff6srA#0) provided a crucial hint in the sources:

```
I have a segmentation fault here:
 thread #1, queue = 'com.apple.main-thread', stop reason = EXC_BAD_ACCESS (code=1, address=0x0)
 frame #0: 0x000000019d488a80 libsystem_platform.dylib`_platform_strncmp$VARIANT$Base + 176
 frame #1: 0x00000001001c0538 arm64-apple-darwin20.0.0-ld`ld::passes::dylibs::doPass(Options const&, ld::Internal&) + 412
 frame #2: 0x0000000100013c04 arm64-apple-darwin20.0.0-ld`main + 732
 frame #3: 0x000000019d0b5d54 dyld`start + 7184
```

I was surprised to find a reference to [lucascolley.github.io](https://lucascolley.github.io/blog/2025-09-26-macos-26-c-compiler/).
While my stack trace looks very generic, Lucas is someone I recognised due to his involvement in conda-forge.
In his article, he actually mentioned that he changed the line that segfaulted for me
If this had been hidden in a GitHub commit, I would not have expected Perplexity to pick it up.
But since Lucas has written the blog post, I did get a very crucial hint (or possibly the solution) at what is breaking here.

This enabled me to resolve the issue in [cctools-and-ld64-feedstock#103](cctools-and-ld64-feedstock/pull/103) by adding a `nullptr` check to the existing patch. Instead of several hours of debugging and understanding a complex codebase, this could be fixed in less than 15 minutes.

## Conclusion

While AI-assisted search may mean the overall number of readers of a blog post goes down, it still exposes the right readers to the right posts. 
In my case, I would even argue that I would not have (re-)discovered that post if it were not for AI-assisted search.
My traceback provided very limited information, and a search on classic search engines didn't yield any results related to my issue.
This makes me personally more confident in writing more of this kind of blog posts.
They will not reach large masses, but actually the people I want to share my knowledge with and that could profit heavily from it.

_Feature image from <a href="https://unsplash.com/de/@zhpix?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Pascal Meier</a> on <a href="https://unsplash.com/de/fotos/wendeltreppe-fotografie-5Z5As4cdE5k?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a>_
