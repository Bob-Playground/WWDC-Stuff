# WWDC17 - Session413: \<App Startup Time: Past, Present, and Future\>

<https://developer.apple.com/videos/play/wwdc2017/413/>

---

- [WWDC17 - Session413: \<App Startup Time: Past, Present, and Future\>](#wwdc17---session413-app-startup-time-past-present-and-future)
  - [Overview](#overview)
  - [Preamble](#preamble)
    - [Terminology](#terminology)
      - [Startup time](#startup-time)
      - [Launch closure](#launch-closure)
  - [Improving App Startup Time](#improving-app-startup-time)
    - [Do less](#do-less)
    - [Use more Swift](#use-more-swift)
  - [Instruments: Static Initializer Tracing](#instruments-static-initializer-tracing)
    - [Demo: Static Initializer Usage](#demo-static-initializer-usage)
  - [A brief history of dyld](#a-brief-history-of-dyld)
    - [dyld 1](#dyld-1)
      - [Prebinding](#prebinding)
    - [dyld 2.0](#dyld-20)
      - [Security issues: limited sanity checking](#security-issues-limited-sanity-checking)
      - [Reduce the amount of prebinding](#reduce-the-amount-of-prebinding)
    - [dyld 2.x](#dyld-2x)
      - [More architectures and platforms](#more-architectures-and-platforms)
      - [Improved security](#improved-security)
      - [Improved performance](#improved-performance)
      - [Shared Cache](#shared-cache)
        - [Rearranges binaries to improve load speed](#rearranges-binaries-to-improve-load-speed)
        - [Pre-links dylibs](#pre-links-dylibs)
        - [Pre-builds data structures used by dyld and ObjC](#pre-builds-data-structures-used-by-dyld-and-objc)
    - [dyld 3](#dyld-3)
      - [Performance](#performance)
      - [Security](#security)
      - [Testability and Reliability](#testability-and-reliability)
  - [How `dyld 2` launches an app](#how-dyld-2-launches-an-app)
    - [Parse Mach-O Headers & Find Dependencies](#parse-mach-o-headers--find-dependencies)
    - [Map Mach-O files](#map-mach-o-files)
    - [Perform symbol lookups](#perform-symbol-lookups)
    - [Bind and rebase](#bind-and-rebase)
    - [Run initializers](#run-initializers)
  - [`dyld 3` is three components](#dyld-3-is-three-components)
    - [`dyld 3` is an out-of-process mach-o parser](#dyld-3-is-an-out-of-process-mach-o-parser)
    - [`dyld 3` is a small in-process engine](#dyld-3-is-a-small-in-process-engine)
    - [`dyld 3` is a launch closure cache](#dyld-3-is-a-launch-closure-cache)
      - [System app](#system-app)
      - [Third-party app](#third-party-app)
      - [macOS: side load applications](#macos-side-load-applications)
  - [Preparing for `dyld 3`](#preparing-for-dyld-3)
    - [`dyld 3` potential issues](#dyld-3-potential-issues)
      - [Fully compatible with dyld 2.x](#fully-compatible-with-dyld-2x)
      - [Stricter linking semantics](#stricter-linking-semantics)
    - [Unaligned pointers](#unaligned-pointers)
      - [Demo: Unaligned pointers](#demo-unaligned-pointers)
    - [Eager symbol resolution](#eager-symbol-resolution)
      - [Linker Flag: _bind_at_load](#linker-flag-_bind_at_load)
    - [dlopen, dlsym and dladdr](#dlopen-dlsym-and-dladdr)
    - [dlclose](#dlclose)
    - [all_image_infos](#all_image_infos)
  - [Best practices](#best-practices)

Hello everybody.

Thanks for coming out this morning. I'm **Louis Gerbarg**. I work on the **dyld Team**, and today we're going to talk about *App Startup: Past, Present and Future*.

So we got a lot to go through, so I'm just going to get into it.

## Overview

So first off I want to do an **overview** of what we're going to be talking about today.

- So **first** we're going to review some advice we gave from last year.
- **Then** I want to talk about some **new tooling** we've developed to make finding certain types of app startup time problems easier.
- **After that** I want to take a side tour into **a brief history of dyld** on our platforms,
- **and then** I want to **discuss the all new dyld** that we're going to be shipping in **macOS High Sierra** and **iOS 11**.
- **And then finally**, I want to talk about **best practices for this new dyld**.

## Preamble

So before that I just want to do a little bit of bookkeeping.

So first off, we want your feedback. So if you have anything you want to tell us, please file bugs with the title *DYLD USAGE*, and hopefully they will get back to us.

And now I want to talk about some **terminology** that I'm going to use in the rest of this talk.

### Terminology

So first off, what does startup time mean?

#### Startup time

And **startup time for the purposes of this talk means time spent before main()**.

Now, if you are writing an app, you have to do more than that. After that happens, there will be *nib loading* and other things like that and you have codes to run after you -- in *UI application delegates* and what not, but you have more visibility into that and there are many other talks about that.

**Today we just want to talk about what happens before your main executes and how you can speed that up.**

#### Launch closure

Additionally, I want to define a launch closure, and this is a new term.

**And a launch closure is all of the information necessary to launch your application:**

- So what **dylibs** it uses,
- what the **offsets** in them are for various **symbols**,
- where their **code signatures** are.

## Improving App Startup Time

And with that, let's go into the main body of the talk.

### Do less

So last year I said **do less**, and I'm going to say that again this year and I'm always going to say that because the less you do, the faster we can launch. And no matter how much we speed things up, if we have less work, it's going to go faster.

And the advice is basically the same.

- You should use fewer dylibs, if you can, you should **embed fewer dylibs**. System ones are better in certain ways from a time perspective, and we'll go into that.
- You should **declare fewer classes and methods**
- and you should **run fewer initializers**.

### Use more Swift

Finally, I'm going to tell you you can do a little bit more of something. You can **use more Swift**, and the reason is Swift is designed in such a way that **it avoids a lot of pitfalls that C, C++ and Objective-C allow you to do**.

- **Swift does not have initializers**.
- **Swift does not allow certain types of misaligned data structures** that cost us time in launch. So, in general, moving to Swift will make it easier for you to get very responsive app startup.
- So also, there are the **Swift size improvements** and smaller is better, so please move to this new Swift that we've shipped this year with the size improvements and that's going to help you out.

## Instruments: Static Initializer Tracing

So now let me talk about some new tooling we have.

So new in **iOS 11** and **macOS High Sierra**, we've added **Static Initializer Tracing** to **Instruments**.

So, yes, this is pretty exciting stuff because **initializers are code that have to run before main to set up objects** for you, and you haven't had much visibility into what happens before main.

So they're available through Instruments and they **provide precise timing for each static initializer**.

### Demo: Static Initializer Usage

So with that, I'd like to go to a demo right now. So over here I have an application, and as most applications at WWDC are, it's a way of sharing cute pictures of animals.

So here, let me launch it.

And, you know, it's taking a little while here, but it's still taking a while and it gets up and we can see some chinchillas and some cats.

And so let's take a look at why it took that time.

So I'm going to go and I'm going to rerun it under **Instruments**. So we'll stop the execution of the current one and run it.

And now if we go in, I'm going to start with a **blank template** and we can add the new **Static Initializer** tool, which is right there. And while we're at it, I'm also going to add a **Time Profiler** because it's always kind of nice to see what's going on.

There we go.

Okay. So now that we have those, let's start running our application.

So we're getting in our trace data, and it's still not up, but it just came up and as you can see in the background there, we had something fill in there.

So I'm going to just zoom in so you can get a look, and I have a function there called `waitForNetworkDebugger`, and that's right, because I was loading these off of an adjacent feed that we had up on our site. I was trying to debug that.

So let's go and -- I just want to actually take a quick look here in the **CPU Usage** tool. So you can see that that **initializer's roughly the same length as my CPU usage**.

So if I go down there, I can actually drill down into **dyld**, and if I do that, we're actually going to see what was taking all that time, and that time is 9.5 seconds, 9.5 seconds into the initializer. It's pretty deep.

You don't usually have to do this, but I want to show you what's going on. And down in here I can finally see `waitForNetworkDebugger`, which is what we saw up in the initializer call, but now it's very easy for you to find that.

So now that we've done that, I'm going to go back over into *Xcode* and, oh, yeah, that's the `waitForNetworkDebugger` call that I implemented. I implemented it in C because Swift won't even let you do something like this, which is because this is a bad idea, but I created a **constructor** there.

So if I go back to my source code -- if I go back to my source code, I can just **delete that function** because it was just for debugging anyway. If I run it, my app's going to come up almost instantly.

So we just saw how to quickly find what stack initializers are causing you slowdowns.

This will work across multiple dylibs, including system dylibs that may be taking a long time because of inputs you've given them, such as **complicated nibs**.

It depends on new infrastructure in **High Sierra and iOS 11's kernel and dyld**, so you need to be running the new builds to see this.

And it catches most initializers now and there's some **edge cases** we're still working on adding, but we think this is going to allow you to quickly find out what is taking time during your app launch so that you get quicker, more responsive application launches that will make your users happy. Thank you.

## A brief history of dyld

Okay. So now I said we'd do **a brief history of dyld**. So **Dynamic Linking** Through the Ages.

So originally we shipped the **first dyld** -- these didn't have version numbers, but retroactively we're giving them them.

### dyld 1

And this was **dyld 1** and it shipped as part of **NeXTStep 3.3** back in **1996**.

**Before that, NeXT used static binaries.**

And it's worth noting this predates the **POSIX dlopen** calls being standardized.

Now, `dlopen` did exist on some Unix. They were proprietary extensions that later people adopted.

And **NeXTStep** had different proprietary extensions, so people wrote **third-party wrappers** on the early versions of **macOS 10** to support standard Unix software.

**The problem was they didn't quite support the same semantics.**

- There were some weird edge cases where it didn't work, and ultimately they were kind of slow.
- It also was **written before most systems used large C++ dynamic libraries**, and this is important.

**C++** has a number of features, such as how its **initializer** ordering works. **And one definition rule, they work well in a static environment, but are actually fairly hard to do, at least with good performance, in a dynamic environment**.

So large C++ code bases cause the dynamic linker to have to do a lot of work and it was quite slow.

#### Prebinding

We also added one other feature before we shipped **macOS 10.0 Cheetah**, and that's called **prebinding**.

And for those of you in the audience who know what prebinding is, I know it was kind of painful;

and for the rest of you,

- **prebinding was a technology where we would try to find fixed addresses for every dylib in the system and for your application**,
- and **the dynamic loader would try to load everything at those addresses and if it succeeded, it would edit all of those binaries to have those precalculated addresses in it**,
- and **then the next time when it put them in the same addresses, it didn't have to do any additional work**.

And that sped up launch a lot, **but it meant that we were editing your binaries on every launch**, and that's not great for all sorts of reasons, **not the least of which is security**.

### dyld 2.0

So then came **dyld 2**, and we shipped that as part of **macOS Tiger**.

And **dyld 2** was a complete rewrite of dyld.

- **It had correct support for C++ initializer semantics**, so we slightly extended the **mach-o format** and we updated **dyld** so that we could get efficient C++ library support.

- It also has a full native **dlopen** and **dlsym** implementation with correct semantics, at which point we deprecated the Legacy API's. They are still on macOS. They have never shipped on any of our other platforms.

#### Security issues: limited sanity checking

**It was designed for speed and because it was designed for speed, it had limited sanity checking**. We did not have the malware environment we have today.

It also has **security issues** because of that, that we had to go back and **retrofit** in a number of features to make it safer on today's platforms.

#### Reduce the amount of prebinding

Finally, because it was so much faster we could reduce the amount of prebinding.

**Rather than editing your applications, we just edited the system libraries and we could do that just at software update times.**

And if you've ever seen **the phrase optimizing system performance appear in your software update**, that was added to the installer to be displayed during the time we were **updating prebinding**.

Nowadays it is used for all the optimizations, but that was the impetus.

### dyld 2.x

So we shipped dyld 2 back then and we've done a number of improvements over the years, significant improvements.

#### More architectures and platforms

First off, we've added a ton of more architectures and platforms.

- Since dyld 2 shipped on **PowerPC**, we've added **x86, x86_64, arm, arm64**, and a number of subvariants of those.
- We've also shipped **iOS, tvOS, and watchOS**, all of which required significant new work in dyld.

#### Improved security

We've improved **security** in a number of ways.

- We added **codesigning** support,
- we added some for **ASLR**, which is a technology **Address Space Layout Randomization**, which means that every time you loaded the libraries it may be at a different address. If you want more details on that, last year's talk where Nick went into extreme detail on how we launch an app, goes into that.
- And finally, we added a significant **bounds checking** to a number of things in the **mach-o header** so that you couldn't do certain types of attach with malformed binaries.

#### Improved performance

Finally, we improved performance, and because we improved performance, we could **get rid of prebinding and replace it with something called the shared cache**.

#### Shared Cache

So what is the **shared cache**?

Well, it was introduced in **iOS 3.1** and **macOS Snow Leopard**, and **it completely replaced prebinding**.

**It's a single file containing most of the system dylibs.**

And because we merged them into a single file, we can do certain types of optimizations.

##### Rearranges binaries to improve load speed

We can **rearrange** all of their **text segments** and all of their **data segments** and **rewrite** their entire **symbol tables** to **reduce the size** and to make it so we need to **mount fewer regions in each process**.

##### Pre-links dylibs

- It also allows us to **pack binary segments** and save a lot of RAM. It effectively is a **prelinker** for the dylibs.
  - And while I'm not going to go into any particular optimizations here, the RAM savings are substantial.
  - On an average iOS system, this is the difference in about **500 megs to a gigabyte of RAM at runtime**.

##### Pre-builds data structures used by dyld and ObjC

It also **prebuilds data structures that dyld and ObjC are going to use at runtime so that we don't have to do it on launch. And again, that saves more RAM and a lot of time.**

- It's **built locally on macOS**, so when you see *optimizing system performance*, we are running *update dyld shared cache*, among things that happen,
- **but on all of our other platforms we actually build it at Apple and ship it to you**.

So now that I've talked about the shared cache, I want to move into dyld 3.

### dyld 3

**dyld 3** is a brand-new dynamic linker, and we're announcing it today.

It's a complete rethink of how we do dynamic linking and

- it's going to be on by default for most macOS system apps in this week's seed,
- and **it will be on by default for all system apps on 2017 Apple OS platforms**.
- **We will completely replace dyld 2 in future Apple OS platforms for all third-party apps as well.**

**So why did we rewrite the dynamic linker again?**

#### Performance

Well, first off, **performance**. In case that's not a recurring theme, we want every ounce of launch speed we can get.

Additionally, we thought what is the **minimum**, what is the theoretical minimum that we could do to get an app up and running and how could we achieve that.

#### Security

**Security**. So as I said, we retrofitted a number of security features into dyld 2, but it's really hard to add that kind of stuff after the fact.

I think we've done a good job with it in recent years, but it's really, really difficult to do that. And so can we have **more aggressive security checking and be designed for security up front**?

#### Testability and Reliability

Finally, **testability and reliability**. Can we make dyld easier to test?

So Apple ships a ton of great testing frameworks, like **XCTest**, that you should be using, and we should be using, but they depend on low level features of dynamic linker to insert those libraries into processes, so they fundamentally cannot be used for testing the existing dyld code, and that also makes it harder for us to test security and performance features.

And so how did we do that?

Well, we've **moved most of dyld out of process**. **It's now mostly just a regular daemon** and we can test that just like everybody else does with standard testing tools, which is going to allow us to move even faster in the future in improving this.

It also **lets the bit of dyld that stays in process be as small as possible and that reduces the attack surface in your applications**.

**It also speeds up launch because the fastest code is code you never write, followed closely by code you almost never execute.**

## How `dyld 2` launches an app

So to tell you how we did this I'm going to briefly show how dyld 2 launches an app.

And again, we went into this in much more detail in last year's talk, **Optimizing App Startup Time**, so if you want to pause, if you're watching this on video, and go watch that, that might be a good idea. Or if you just want to follow along here, I'm going to go through it briefly.

So first off we have **dyld 2** and your app starts launching.

### Parse Mach-O Headers & Find Dependencies

So we have to parse your **mach-o**, and as we **parse your mach-o** we find what libraries you need, and then they may have other libraries that they need, and we do that **recursively** until we have a complete graph of all your **dylibs**, and for an average graph of application on iOS that's **between 300 to 600 dylibs**, so it's a lot of them and a lot of work.

### Map Mach-O files

We then map in all the mach-o files so we get them into your address space.

### Perform symbol lookups

We then perform **symbol lookups**, so we actually look and say if your application uses **printf**, we go and look and see that printf is in **lib system**, and we **find the address of it and we basically copy that into a function pointer in your application**.

### Bind and rebase

Then we do what's called **binding and rebasing**, which is where we **copy those pointers in** and we also -- because you're at a random address all of your pointers have to **have that base address added to them**.

### Run initializers

And then **finally**, we can **run all of your initializers**, which is what I showed the tooling for earlier, and **at that point we're ready to call your main in launch**, and that's a lot of work.

---

So how can we make this faster and how can we move it out of process?

Well, first off we identify the security sensitive components.

And from our perspective the **biggest ones** of those are **parsing mach-o headers and finding dependencies**,

because malformed **mach-o headers allow people to do certain attacks** and **your applications may use @rpaths, which are search paths, and by malforming those or inserting libraries in the right places, people can subvert applications**.

**So we do all of that out of process in the daemon, and then we identify the expensive parts of it, which are cache-able, and those are the symbol lookups.**

**Because in a given library, unless you perform the software update or change the library on disk, the symbols will always be at the same offset in that library.**

So we've identified these. Let me show you how they look in dyld 3.

## `dyld 3` is three components

**So we moved those all up front, at which point we write a closure to disk.** So as I said earlier, **a launch closure is everything you need to launch the app**. And then we move it -- we can use that in process later. So **dyld 3 is three components**.

- It's **an out-of-process mach-o parser and compiler**.
- It's **an in-process engine that runs launch closures**,
- and **it's a launch closure caching service**.

**Most launches use the cache and never have to invoke the out-of-process mach-o parser or compiler.**

And **launch closures are much simpler than mach-o**. **They are memory map files we don't have to parse in any complicated way. We can validate them simply. They are built for speed.**

And so let's talk about each one of those parts a little bit more.

### `dyld 3` is an out-of-process mach-o parser

So dyld 3 is an out-of-process mach-o parser. So what does that do?

- It **resolves** all the **search paths**, all the **@rpaths**, all the **environment variables** that can affect your launch.
- Then it **parses the mach-o binaries**
- and it performs all of those **symbol lookups**.
- Finally, it **creates the closure with the results**,
- and it's that **normal daemon** so that we can get that improved testing infrastructure.

### `dyld 3` is a small in-process engine

**dyld 3 is a small in-process engine** as well, and **this is the part that will be in your process and this is what you will mostly see.**

So all it does is

- it **validates that the launch closure** is correct
- and then it just **maps in the dylibs**
- and **jumps to main**.

And one of the things you may notice is **it never needs to parse a mach-o header or perform a symbol lookup**. We don't have to do those to launch your app anymore.

And since that's where we're spending most of our time, it's going to result in much faster app launches for you.

### `dyld 3` is a launch closure cache

Finally, **dyld 3 is a launch closure caching service.**

So what does that mean?

#### System app

Well, **system app closures we're just building directly into the shared cache**. We already have this tool that runs and analyzes every mach-o in the system.

We can just put them directly into the shared cache, so it's mapped in with all the dylibs to start with. We don't even need to open another file.

#### Third-party app

For third-party apps we're going to **build your closure during app install or system updates because at that point the system library has changed**.

So by default **these will all be prebuilt for you on iOS and tvOS and watchOS before you even run**.

#### macOS: side load applications

On macOS, because you can **side load applications**, the **in-process engine** can **RPC** out to the **daemon** if necessary on first launch, and then **after that it will be able to use a cached closure just like everything else**.

But like I said, that is not necessary on any of our other platforms.

## Preparing for `dyld 3`

### `dyld 3` potential issues

So now that I've talked about this dynamic linker that we'll be using for system apps this year and for your apps in the future, I want to talk to you about some potential issues you might see with it so that you can start updating your apps for it now.

#### Fully compatible with dyld 2.x

So first off, **it is fully compatible with dyld 2.x**.

- So **some existing API's will cause you to run slower or use fallback modes** in dyld 3 though, so we'd like you to avoid those, and we'll go into those in a second.
- Also, **some existing optimizations that you are doing may not be necessary anymore**, so you don't have to rip them out but, you know, it may not be worth putting in a lot of effort.

#### Stricter linking semantics

The other thing I want to talk about is that we're going to have **stricter linking semantics**.

So what do I mean by that?

Well, there's a lot of things that maybe work most of the time, but aren't actually correct even today and so we've identified a lot of those.

As we've been putting the new dynamic linker in, that tends to find all these **edge cases**.

- So what we've been doing is we've been **putting in workarounds for old binaries**, but we do not intend to carry those forward.
- We will do linked on or after checks to see what SDK you were built with and we **will disable those workarounds for new binaries** so that you move to these improved -- you fix these issues.

**So new binaries will cause linker issues.**

### Unaligned pointers

So, first off, I want to talk about **unaligned pointers** in your **data segments**.

So what do I mean by this?

Well, when you have **a global structure that points to a function or another global structure, that's a pointer that we have to fix up before you launch, and pointers must be naturally aligned on our system for best performance**.

And **fixing up unaligned pointers is much more complex**.

**They can span multiple pages, which can cause more page faults and other issues, and they can have atomicity issues related to multiprocessors.**

**The static linker already emits a warning for this, ld warning, pointer not aligned at address, and that's an address, often your data segments.**

And if you're fixing all warnings, you should -- hopefully you've already taken care of this. The seeds that we have out this week have some issues with **Swift keypaths**, but they will be fixed so you can ignore those, but other than that, please go and fix these issues.

#### Demo: Unaligned pointers

So for those of you who are asking how would you get something like this, I'm going to just show you real quick.

If you don't know how, it takes a lot of work. **You can't do it in Swift. So again, use more Swift.** This code here will do it, so let me show you what's going on.

---

First off, I have **attributes forcing specific alignment**. So by default the compiler's going to align it correctly for you. But sometimes you may need special alignments and this case I've said **change whatever the default alignment rules are to one**, and I've done that in two different ways just to be really, really bad, so you have to fix both of these.

Then I constructed a **global variable**. That global variable sets a pointer in with the structures and that's going to force the dynamic linker to fix up that pointer on launch.

So if you see code like this, you can just remove the alignments. You could rearrange the structure so that the pointer goes first, because that's a better alignment thing.

And there's plenty of guides online about **C structure alignment** if you want to get into the **nitty-gritty**, but hopefully you don't have to deal with this, and if you write Swift, you definitely don't have to.

### Eager symbol resolution

So next off, **eager symbol resolution**.

So what do I mean by this?

So **dyld 2 performs what we call lazy symbol resolution**.

So I said up front that dyld has to load all those symbols and that's something expensive that we want to cache. It's actually too expensive to run up front on existing applications. It would take too long.

So instead, we use a mechanism we call **lazy symbol resolution**, where, by default, the function pointer in your binary for, let's say, **printf**, doesn't point to printf. **By default it points to a function in dyld that returns a function pointer to printf.** And so when we launch, you'll call printf, it goes into dyld, we return what we call the printf and call it on your behalf the first time and then on the second time you go straight to printf.

**But since we are caching and calculating all these symbols up front now, there's no additional cost at app launch time to find them all up front**, so we are going to do that.

Now, having said that, missing symbols behave differently when you do this.

- **On existing lazy systems**, if you are missing a symbol, the first call -- **you'll launch correctly and the first time you call that symbol, you'll crash**.
- **With eager symbols you'll crash up front**.

So we do have a compatibility mode for this, and the way we're going to do that is that we are going to **just have a symbol inside dyld 3 that automatically crashes, and if we can't find your symbol, we will bind that symbol, so on first call you will crash.**

But again, that's how it works on today's SDK. If future SDK's we are going to force all symbol resolution to be up front. So if you are missing a symbol, you will crash, and that should hopefully result in you discovering crashes during development instead of your users discovering them at runtime.

#### Linker Flag: _bind_at_load

And you can simulate that behavior now today. There's a special **linker flag** which is dyld **_bind_at_load**, so if you add this to your debug build, as I said, **it's much slower, so please only put it in your debug builds**, but add this to your debug builds and you'll get more reliable behavior today and it will get you ready for what we're going to be doing in dyld 3.

Again, only use that in your test builds.

### dlopen, dlsym and dladdr

**dlopen, dlsym and dladdr**. So last year I got up here and said please **don't use them unless you have to**, but we understand that you may have to, and that's the same thing I'm saying this year.

**They have some problematic semantics**, but they are still necessary in some cases.

**Particularly, symbols found with dlsym, we need to find it at runtime**. We don't know ahead of time what they are. We can't do all that prefetching and presearching.

**So as soon as you use dlopen or dlsym, we're going and we're reading in all the pages for your symbol table that we didn't have to touch before.** So it's going to be a lot more expensive.

**Additionally**, we might have to **RPC out to the daemon**（Remote Procedure Call？远程过程调用？）, depending on how complicated it is.

So we're working on some better alternatives. We don't have those yet. But we also need to hear about your use cases to make sure we're designing something that will work for you. So please, again, they're not going away, but they will be slower and we want your feedback.

### dlclose

I want to take a second to talk about **dlclose** specifically. And so **dlclose is a bit of a misnomer**. It's a **Unix API**, so that's the name, but on our system, **if we had been writing it, it probably would be called dlrelease because it doesn't actually close the dylib. It decrements a refcount and if the refcount hits zero, it closes it.**

And why is that important?

Well, it's not appropriate for resource management. If you have a library that attaches to a piece of hardware, you shouldn't shut down the hardware in response to a dlclose because some other code in your app may have opened it behind your back and so now your hardware's not shutting down. You should have explicit resource management.

There are also a number of features on our platforms that prevent dylibs from unloading, and I'd like to go through a few of those because maybe you do them.

- You can have Objective-C classes in your dylib. That will make it not unloadable.
- You could have Swift classes. That will also make it not unloadable.
- And you can have **C under bar thread** or **C++ thread local variables**, all of which make it impossible to unload a dylib.

So on **macOS**, where there's a number of existing **Unix apps**, obviously we will keep this working, but because **almost every dylib on all of our other platforms does one of these things**, effectively it hasn't really worked on any of them ever.

So we are considering **making it just a straight up no-op, that will not do anything on any of those platforms**. If there's a reason why that's a problem, please, we want to hear about it.

### all_image_infos

Finally, I want to talk about the dyld **all_image_infos**. So this is **an interface for introspecting dylibs in a process, and it comes from the original dyld 1**.

But **it's just a struct in memory, it's not an API**, and **that was okay when we had five, ten dylibs. But as we've gotten to 300, 400, 500 dylibs, the way it's designed wastes a lot of memory**, and we want that memory back. We always want our performance and we always want our memory.

So we're going to take it away in future releases, but we will be providing replacement API's for it. And so it's very rarely used, but if you're using it, again, I want to know why you're using it and how you're using it and make sure we design API's that fit your use case.

There are a number of bits of it that are **vestigial** and don't quite do what you expect or work anyway today, so if you aren't using those, they may just go away and we need to hear about that. So please let us know how you use it.

## Best practices

So finally, let's talk about best practices.

First, make sure you launch your app with **_bind_at_load** in your **LD FLAGS**（Other linker flags） for **debug builds only**.

Fix any unaligned pointers in your data segments. Again, this warning is there. You should try to be fixing all of your warnings.

If you see it with the new Swift keypath feature, you can ignore that because we'll fix that.

You can make sure you are not depending on any terminators running when you call dlclose.

And **we want you to let us know why you're using dlopen, dlsym, dladdr, and the all_image_infos structures** to make sure that our replacements are going to suit your needs.

In the case of the ones that are part of **POSIX**, they will stay around, **they will just be lower performance**. In the case of **all_image_infos**, it is going to go away to save that memory.

Please file bug reports using **DYLD USAGE** in their titles so that they get to us so that we can find out all of your usage cases that we need to support.

And for more information, you can go to this URL. Related sessions. So last year we did **Optimizing App Startup Time**, so you may want to go and watch that for a refresher on how **traditional dynamic linking** works. It goes into much more detail than I did here since I was trying to discuss all the new stuff we're doing.

So thank you everybody for coming.
