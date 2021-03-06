---
layout: layouts/post.njk
title: Java's Project Loom
date: 2021-06-21T07:07:17.028Z
tags:
  - dev
  - java
  - loom
---
An upcoming Java feature I'm excited about and looks like a game-changer is [Project Loom](https://wiki.openjdk.java.net/display/loom/Main). 

It's an improvement to the concurrency model which introduces lightweight **virtual threads** (aka fibers, aka user-mode threads) into the Thread API. These will exist alongside the existing **platform threads** (aka kernel threads).

The upshot is that Java can manage your threaded programs more efficiently and allow you to write simpler concurrent code. The majority of the work is done under the hood with very minimal API changes.

For example, where a developer would have to consider sizing and managing thread pools, Loom allows you to simply use a **thread per task**. Java, instead of the OS, will take care of scheduling these virtual threads and mapping them to platform threads. Benefiting from this may be as easy as replacing:

```java
Executors.newFixedThreadPool(NTHREADS);
```

with this:

```java
Executors.newVirtualThreadExecutor();
```

For highly-concurrent applications ― high-traffic web servers say ― the current limitations of expensive platform threads can warrant a major design choice to use an asynchronous paradigm, such as [Reactive.](https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html) Loom proponents contest that this is now unnecessary because you can instead create *millions* of virtual threads and write the same straightforward code that would previously have only served *thousands* of requests.

I recently attended a [Manchester Java Community](https://twitter.com/mcrjava) Q&A event with the Project Loom lead, Ron Pressler ― a highly recommended watch...

<iframe width="560" height="315" src="https://www.youtube.com/embed/h7VoiMNo67o" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>