# WWDC16 - Session406: \<Optimizing App Startup Time\>

<https://developer.apple.com/videos/play/wwdc2016/406/>

---

- [WWDC16 - Session406: \<Optimizing App Startup Time\>](#wwdc16---session406-optimizing-app-startup-time)
  - [Mach-O](#mach-o)
    - [Segments](#segments)
    - [Sections](#sections)
    - [Universal files](#universal-files)
  - [Virtual memory](#virtual-memory)
    - [What is virtual memory](#what-is-virtual-memory)
    - [What can you do with VM](#what-can-you-do-with-vm)
    - [File backed mapping](#file-backed-mapping)
    - [Copy on write](#copy-on-write)
  - [From exec to main](#from-exec-to-main)
    - [positioned independent code](#positioned-independent-code)
  - [Practical tips](#practical-tips)
    - [warm launch & cold launch](#warm-launch--cold-launch)
    - [DYLD_PRINT_STATISTICS](#dyld_print_statistics)
    - [dylib loading](#dylib-loading)
    - [binding and rebasing](#binding-and-rebasing)
      - [C++ virtual functions](#c-virtual-functions)
      - [Migrating to Swift](#migrating-to-swift)
      - [Machine generated codes](#machine-generated-codes)
    - [initializer](#initializer)
      - [explicit initializers](#explicit-initializers)
      - [implicit initializers](#implicit-initializers)

Good morning and welcome to session 406, Optimizing App Startup Time.

My name is Nick Kledzik, and today my colleague Louis and I are going to take you on a guided tour of **how a process launches**.

Now you may be wondering, is this topic right for me. So we had our crack developing marketing team do some research, and they determined there are three groups that will benefit by listening to this talk.

- The first, is app developers that have a app that launches to slowly.
- The second group, is app developers that don't want to be in the first group .
- And lastly, is anyone who's just really curious about how the OS operates.

---

So this talk is going to be divided in **two sections**,

- the first is more **theory**
- and the second more **practical**,

I'll be doing the first theory part.

And in it I'll be walking you through all the steps that happen, all the way up to **main()**.

But in order for you to understand and appreciate all the steps I first need to give you a crash course on **Mach-O** and **Virtual Memory**.

## Mach-O

So first some **Mach-O** terminology, quickly.

**Mach-O is a bunch of file types for different run time executables.**

- So the first **executable**, that's **the main binary in an app**, it's also **the main binary in an app extension**.
- A **dylib** is a dynamic library, on other platforms meet, you may know those as **DSOs** or **DLLs**. Our platform also has another kind of thing called a **bundle**. Now **a bundle's a special kind of dylib that you cannot link against, all you can do is load it at run time by an dlopen and that's used on a Mac OS for plug-ins**.
- Last, is the term **image**. **Image refers to any of these three types**. And I'll be using that term a lot.
- And lastly, the term **framework** is very overloaded in our industry, but in this context, **a framework is a dylib with a special directory structure around it to holds files needed by that dylib**.

### Segments

So let's dive right into the **Mach-O image format**. **A Mach-O image is divided into segments, by convention all segment names are, use upper case letters.**

Now, **each segment is always a multiple of the page size**, in this example the `__TEXT` is 3 pages, the `__DATA` and `__LINKEDIT` are each one page.

Now **the page size is determined by the hardware, for arm64, the page size is 16K, everything else it's 4k.**

### Sections

Now another way to look at the thing is sections. **So sections is something the compiler omits. But sections are really just a subrange of a segment, they don't have any of the constraints of being page size, but they are non-overlapping.**

---

Now, the most common segment names are `__TEXT`, `__DATA`, `__LINKEDIT`, in fact almost every binary has exactly those three segments. You can add custom ones but it usually doesn't add any value.

So what are these used for?

Well `__TEXT` is **at the start of the file, it contains the Mach-O header, it contains any machine instructions as well as any read only constant such as C strings.**

The `__DATA` segment is **rewrite, the `__DATA` segment contains all your global variables.**

And lastly, is the `__LINKEDIT`. Now the `__LINKEDIT` doesn't contain your functions of global variables, a `__LINKEDIT` **contains information about your function of variables such as their name and address.**

### Universal files

You may have also heard of **universal files**, what are they?

Well suppose you build an iOS app, for a **64 bit**, and now you have this **Mach-O** file, so what happens the next code when you say you also want to build it for **32 bit** devices?

When you rebuild, Xcode will build **another separate Mach-O file**, this one built for **32 bits**, RB7.

And then **those two files are merged into a third file, called the Mach-O universal file**. And that has a **header** at the start, and **all the header has a list of all the architectures and what their offsets are in the file**. And that header is also **one page in size**.

## Virtual memory

Now you may be wondering,

why are the segments multiple page sizes?

Why is the header a page sizes, and it's wasting a lot of space.

Well the reason everything is page based has to do with our next topic which is virtual memory.

### What is virtual memory

So what is virtual memory?

Some of you may know **the adage in software engineering that every problem can be solved by adding a level of indirection**.

So the problem with, that virtual memory solves, is how do you manage all your physical RAM when you have all these processes?

So they added a little of indirection. **Every process is a logical address space which gets mapped to some physical page of RAM.**

Now **this mapping does not have to be one to one, you could have logical addresses that go to no physical RAM and you can have multiple logical addresses that go to the same physical RAM**. This offered lots of opportunities here.

### What can you do with VM

So what can you do with VM?

Well first, **if you have a logical address that does not map to any physical RAM, when you access that address in your process, a page fault happens**. At that point the kernel stops that thread and tries to figure out what needs to happen.

The next thing is **if you have two processes, with different logical addresses, mapping to the same physical page, those two processes are now sharing the same bit of RAM. You now have sharing between processes.**

### File backed mapping

Another interesting feature is **file backed mapping**.

Rather than actually read an entire file into RAM you can tell the VM system through the **mmap** call, the I want this slice of this file mapped to this address range in my process.

So why would you do that?

Well rather than having to read the entire file, by having that mapping set up, as you first access those different addresses, as if you had read it in memory, each time you access an address that hasn't been accessed before it will cause a page fault, the kernel will read just that one page. And that gives you lazy reading of your file.

Now we can put all these features together, and what I told you about Mach-O you now realize that the `__TEXT` segment of any of that dylib or image can be mapped into multiple processes, it will be read lazily, and all those pages can be shared between those processes.

### Copy on write

What about the `__DATA` segment?

The `__DATA` segment **is read, write**, so for that we have trick called **copy on write**, it's kind of similar to the, cloning that seen in the Apple file system.

What copy and write does is it optimistically shares the `__DATA` page between all the processes.

What happens when one process, as long as they're **only reading from the global variables** that sharing works.

But **as soon as one process actually tries to write** to its `__DATA` page, the **copy and write happens**.

**The copy and write causes the kernel to make a copy of that page into another physical RAM and redirect the mapping to go to that. So that one process now has its own copy of that page.**

Which brings us to **clean versus dirty pages**.

So that copy is considered a dirty page.

- A dirty page is something that contains process specific information.
- A clean page is something that the kernel could regenerate later if needed such as rereading from disc.

So dirty pages are much more expensive than clean pages.

---

And the last thing is the permission boundaries are on page boundaries.

By that I mean the permissions are you can mark a page readable, writable, or executable, or any combination of those.

So let's put this all together, I talked about the Mach-O format, something about virtual memory, let's see how they play together.

Now I'm going to skip ahead and talk a little, how the dyld operates and in a few moments I'll actually walk you through this but for now, I just want to show you how this maps between Mach-O and virtual memory.

So we have a dylib file here, and **rather than reading it in memory we've mapped it in memory**. So, in memory this dylib would have taken eight pages.

The savings, why it's different is these **ZeroFills**. So it turns out most global variables are zero initially. So the static makes an optimization that moves all the zero global variables to the end, and then takes up no disc space. And instead, we use the VM feature to tell the VM the first time this page is accessed, fill it with zero's. So it requires no reading.

---

So the first thing **dyld** has to do is it has to look at the **Mach-O header**, in memory, in this process. So it'll be looking at the top box in memory, when that happens, there's nothing there, there's no mapping to a physical page so a page fault happens. At that point the kernel realizes this is mapped to a file, so it'll read the first page of the file, place it into physical RAM, set the mapping to it.

Now dyld can actually start reading through the **Mach-O header**. It reads through the **Mach-O header**, the **Mach-O header** says oh, there's some information in the `__LINKEDIT` segment you need to look at. So again, dyld drops down what's in the bottom box in process one. Which again causes a **page fault**.

Kernel services it by reading into another physical page of RAM, the `__LINKEDIT`. Now dyld can expect a `__LINKEDIT`.

Now in process, the `__LINKEDIT` will tell dyld, you need to make some fix ups to this `__DATA` page to make this dylib runable.

So, the same thing happens, dyld is now, reads some data from the `__DATA` page, but there's something different here. dyld is actually going to write something back, it's actually going to change that `__DATA` page and at this point, a **copy on write happens. And this page becomes dirty**.

So what would have been 8 pages of dirty RAM if I just malloced eight pages and then then read the stuff into it I would have eight pages of dirty RAM. But now I only have one page of dirty RAM and two clean pages.

---

So what's going to happen when the second process loads the same dylib. So in the second process dyld goes through the same steps.

First it looks at the **Mach-O header**, but this time the kernel says, ah, I already have that page in RAM somewhere so it simply redirects the mapping to reuse that page no IO was done.

The same think with `__LINKEDIT`, it's much faster. Now we get to the `__DATA` page, at this point the kernel has to look to see if the `__DATA` page, the clean copy already still exists in RAM somewhere, and if it does it can reuse it, if not, it has to reread it.

And now in this process, dyld will dirty the RAM.

---

Now the last step is the `__LINKEDIT` is only needed while dyld is doing its operations. **So it can hint to the kernel, once it's done, that it doesn't really need these __LINKEDIT pages anymore, you can reclaim them when someone else needs RAM.**

So the result is now we have two processes sharing these dylibs, each one would have been eight pages, or a total of 16 dirty pages, but now we only have two dirty pages and one clean, shared page.

---

Two other minor things I want to go over is that how security effects dyld, these two big security things that have impacted dyld.

So one is **ASLR, address space layout randomization**, this is a decade or two old technology, where basically you **randomize the load address**.

The second is **code signing**, it has to, many of you have had to deal with code signing, in Xcode, and you think of code signing as, you **run a cryptographic hash over the entire file, and then sign it with your signature**. Well, in order to validate that run time, that means the entire file would have to be re-read.

So instead what actually happens at build time, is **every single page of your Mach-O file gets its own individual cryptographic hash. And all those hashes are stored in the __LINKEDIT**.

This allows each page to be validated that it hasn't been **tampered** with and was owned by you at page in time.

## From exec to main

Okay, so we finished the crash course, now I'm going to walk you from **exec()** to **main()**.

So what is **exec()**?

`exec()` is a **system call**.

When you trap into the kernel, you basically say I want to replace this process with this new program. The kernel wipes the entire address space and maps in that executable you specified.

Now for **ASLR** it maps it in at a random address. The next thing it does is **from that random, back down to zero, it marks that whole region inaccessible**, and by that I mean it's marked **not readable, not writeable, not executable**.

The size of that region is

- **at least 4KB to 32 bit processes** and
- **at least 4GB for 64 bit processes**.

This catches any NULL pointer references and also foresees more bits, it catches any, pointer truncations.

Now, life was easy for the first couple decades, of Unix because all I do is map a program, set the **PC** into it, and start running it.

And then shared libraries were invented.

---

**So who loads dylibs?**

They quickly realize that they got really complicated fast and the kernel people didn't want the kernel to do it, so instead **a helper program** was created. In our platform it's called **dyld**.

On other Unix's you may know it as `LD.SO`.

**So when the kernel's done mapping a process it now maps another Mach-O called dyld into that process at another random address.**

**Sets the PC into dyld and let's dyld finish launching the process.**

So **now dyld's running in process and its job is to load all the dylibs that you depend on and get everything prepared and running**.

---

So let's walk through those steps. This is a whole bunch of steps and it has sort of a timeline along the bottom here, as we walk through these we'll walk through the timeline.

So first thing, is dyld has to **map all the dependent dylibs**.

Well what are the dependent dylibs?

To find those **it first reads the header of the main executable that the kernel already mapped in** that header is a list of all the dependent libraries. So it's got to parse that out. Then it has to find each dylib.

And once it's found each dylib it has to open and run the start of each file, it needs to make sure that it is a Mach-O file, **validate it, find its code signature, register that code signature to the kernel**. And then it can actually **call mmap at each segment in that dylib**.

Okay, so that's pretty simple. Your app knows about the kernel dyld, dyld then says oh this app depends on A and B dylib, load the two of those, we're done. Well, it gets more complicated, because A.dylib and B.dylib themselves could depend upon the dylibs.

So dyld has to do the same thing over again for each of those dylibs, and each of the dylibs may depend on something that's already loaded or something new so it has to determine whether it's already been loaded or not, and if not, it needs to load it. So, this continues on and on.

And eventually it has everything loaded. Now if you look at a process, the average process in our system, loads anywhere between 1 to 400 dylibs, so that's a lot of dylibs to be loaded.

Luckily most of those are OS dylibs, and we do a lot of work when building the OS to **pre-calculate** and **pre-cache** a lot of the work that dyld has to do to load these things.

So OS dylibs load very, very quickly.

---

So now we've loaded all the dylibs, but they're all sitting in their floating independent of each other, and now we actually have to bind them together.

That's called **fix-ups**.

But one thing about fix-ups is we've learned, **because of code signing we can't actually alter instructions**.

So how does one dylib call into another dylib if you can't change the instructions of how it calls?

### positioned independent code

Well, we call back our old friend, and we add a lot of old indirection. So our **code-gen**, is called **dynamic PIC**. It's **positioned independent code**, meaning the code can be loaded into the address and is dynamic, meaning things are, addressed indirectly.

What that means is to call for one thing to another, the co-gen actually creates a pointer in the `__DATA` segment and that pointer points to what you want to call. The code loads that pointer and jumps to the pointer. So all dyld is doing is fixing up pointers and data.

---

Now there's two main categories of fix-ups, **rebasing and binding**, so what's the difference?

- So **rebasing** is if you have **a pointer that's pointing to within your image**, and any adjustments needed by that,
- the second is **binding**. Binding is if you're **pointing something outside your image**.

And they each need to be fixed up differently, so I'll go through the steps.

But first, if you're curious, there's a command, dyld info with a bunch of options on it. You can run this on any binary and you'll see all the fix-ups that dyld will have to be doing for that binary to prepare it.

---

So rebasing.

Well in the old age you could specify a preferred load address for each dylib, and that preferred load address was the **static linker** and dyld work together such that, if you load, it to that preferred load address, all the pointers and data that was supposed to code internally, were correct and dyld wouldn't have to do any fix-ups.

But these days, with **ASLR**, your **dylib is loaded to a random address**. It's slid to some other address, which means all those pointers and data are now still pointed to the old address. So in order to fix those up, we need to calculate the slide, which is how much has it moved, and for each of those interior pointers, to basically add the slide value to them.

**So rebasing means going through all your data pointers, that are internal, and basically adding a slide to them**.

So the concept is very simple, read, add, write, read, add, write.

But where are those data pointers?

Where those pointers are in your segment, are encoded in the `__LINKEDIT` segment.

Now, at this point, all we've had is everything mapped in, so when we start doing rebasing, we're actually causing page faults to page in all the `__DATA` pages. And **then we causing copy and writes as we're changing them**.

So rebasing can sometimes be expensive because of all the IO.

But one trick we do is we do it sequentially and from the kernel's point of view, it sees data faults happen sequentially. And when it sees that, the kernel, is reading ahead for us which makes the IO less costly.

---

So next is **binding**, binding is for **pointers that point outside your dylib**.

**They're actually bound by name, they're actually is the string**, in this case, malloc stored in the `__LINKEDIT`, that says **this data pointer needs to point to malloc**.

So at run time, dyld needs to actually **find the implementation of that symbol**, which requires a lot of computation, **looking through symbol tables**.

**Once it's found, that values that's stored in that data pointer.**

So this is way more computationally complex than rebasing is. But there's very little IO because rebasing has done most of the IO already.

Next, so ObjC has a bunch of `__DATA` structures, class `__DATA` structure which is a pointer to its methods and a pointer to a super gloss and so forth.

Almost all those are fixed up, via rebasing or binding. But there's a few extra things that ObjC runtime requires.

The first is **ObjC** is dynamic language and you can request a class become substantiated by name. So that means **the ObjC runtime has to maintain a table of all names of which class that they map to**. So every time you load something, it defines a class, its name needs to be registered with a global table.

Next, in **C++** you may have heard of the fragile ivar problem, sorry. **Fragile base class problem**.

We don't have that problem with ObjC because one of the fix-ups we do is we change the offsets of all the ivars dynamically, at load time.

Next, in **ObjC** you can define **categories** which change the methods of another class. Sometimes those are in classes that are not in your image on another dylib, that, those method fix-ups have to be applied at this point.

And lastly, **ObjC** is based on **selectors being unique** so we need unique selectors.

---

So now the work that we've done all the `__DATA` fix-ups, now we can do all the `__DATA` fix-ups that can be basically described statically.

So now's our chance to do dynamic `__DATA` fix ups.

So in **C++**, you can have an initializer, you can say equals whatever expression you want. That arbitrary expression, at this time needs to be run and it's run at this point now.

So the **C++** compiler generates, initiliazers for these arbitrary `__DATA` initialization.

In **ObjC**, there's something called the `+load` method. Now the `+load` method is **deprecated, we recommend that you don't use it**. We recommend you use a `+initialize`. But if you have one, it's run at this point.

---

So, now I have this big graph, we have your **main executable** top, **all the dylibs** depend on, this huge graph, we have to run **initializers**.

What order do we want them in?

Well, we run them bottom up. And the reason is, when an initialize is run it may need to call up some dylib and you want to make sure that dylibs already ready to be called.

So by running the initializers from the bottom all the way up the app class you're safe to call into something you depend on.

So once all initiliazers are done, now we actually finally get to call the `main()` dyld program.

So you survived this theory part, you now all are experts on how processes start, you now know that **dyld is a helper program**, it loads all dependent libraries, fixing up all the `__DATA` pages, runs initializers and then jumps to `main()`.

So now to put all this theory you've learned to use, I'd like to hand it over to Louis, who will be giving you some practical tips.

## Practical tips

Thanks, Nick.

We've all had that experience where we pull our phone out of our pocket, press the home button, and then tap on an application we want to run. And then tap, and tap, and tap again on some button because it's not responding.

When that happens to me, it's really frustrating, and I want to delete the app.

I'm Louis Gerbarg I work on dyld and today, we're going to discuss how to make your app launch instantly, so your users are delighted.

---

- So first off, let's discuss what we're going to go through in this part of the talk.
- We're going to discuss **how fast you actually need to launch** so that your users are going to have a good experience.
- **How to measure that launch time**. Because it can be very difficult. **The standard ways you measure your application don't apply before your code can run.**
- We're going to go through a list of the common reasons why your code, or sorry we're going to go through a list of, why, the **common reasons your launch can be slow**.
- And finally, we're going to go through, **a way to fix all the slow downs**. So I'm going to give you a little spoiler for the rest of my talk.

---

You need to do less stuff .

Now, I don't mean your app should have less features, I'm saying that your **app has to do less things before it's running**. We want you to figure out how to defer some of your launch behaviors in order to initialize them just before execution.

So, let's discuss the goals, how fast we want to launch.

Well, the launch time for various platforms are different. But, a good, a good rule of thumb, is **400 milliseconds** is a good launch time.

Now, the reason for that is that we have launch animations on the phone to give a sense of continuity between the home screen and your application, when you see it execute. And those animations take time, and those animations, give you a chance to hide your launch times.

Obviously that may be different, in different context your app extensions are also applications that have to launch, they launch in different amounts of time.

And a phone and TV, and a watch are different things, but 400 milliseconds is a good target.

You can **never take longer than 20 seconds to launch**. If you take longer than 20 seconds, the OS will kill your app, assuming it's going through an infinite loop, and we've all had that experience. Where you click an app, it comes up to a home screen, it doesn't respond, and then it just goes away, and that's usually what's happening here.

Finally, it's very important to **test on your slowest supported device**. So those timers are constant values across all supported devices on our platforms.

So, if you hit 400 milliseconds on a *iPhone 6S* that you're using for testing right now, you're probably just barely hitting it, you're probably not going to hit it on a *iPhone 5*.

So let's do a recap of Nick's part of the talk.

What do we have to do to launch,

- we have to parse images,
- map images,
- rebase images,
- bind images,
- run image initializers,
- and then call `main()`.

If that sounds like a lot, it is, I'm exhausted just saying it. And then after that, we have to call `UIApplicationMain`, you'll see that in your **ObjC** apps or in your **Swift** apps handled implicitly. That does some other things, including running the framework initializers and loading your nibs.

And then finally you'll get a call back in your **application delegate**.

I'm mentioning these last two because **those are counted in those 400 milliseconds times** that I just mentioned. But we're not going to discuss them in this talk.

If you want a better view of what goes on there, there's **a talk from 2012, iOS app performance responsiveness** (<https://developer.apple.com/videos/play/wwdc2012/235/>). I highly recommend you go back and view the video. But that's the last we're going to speak of them right now.

### warm launch & cold launch

So, let's move on, one more thing I want to talk about, warm versus cold launches.

So when you launch an app, we talk about warm and cold launches.

And a **warm launch** is an app where the **application is already in memory, either because it's been launched and quit previously, and it's still sitting in the discache in the kernel, or because you just copied it over**.

A **cold launch** is a launch where it's not in the discache.

And a cold launch is generally the more important to measure.

The reason a cold launch is more important to measure is **that's when your user is launching an app after rebooting the phone, or for the first time in a long time**, that's when you really want it to be instant.

In order to measure those, you really need to **reboot** between measurements. Having said that, if you're working on improving your warm launches, your cold launches will tend to improve also. You can do rapid development cycles on warm launches, but then every so often, test with a cold launch.

### DYLD_PRINT_STATISTICS

So, how do we measure time before `main()`?

Well, we have a built in measurement system in dyld, you can access it through setting an environment variable， **DYLD_PRINT_STATISTICS**.

And it's been available in shipping OSes actually, but it prints out a lot of internal debugging information that's not particularly useful, it's missing some information that you probably want.

And we're fixing that today.

So it's significantly improved on the new OSes.

It's going to put out, a lot more relevant information for you that should give you actionable ways to improve your launch times. And it will be available in seed 2.

So, one other thing I want to talk about with this, is that the **debugger has to pause launch on every single dylib load in order to parse the symbols from your app and load your break points, over a USB cable that can be very time consuming**.

**But dyld knows about that and it subtracts the debugger time out from the numbers it's registering**. So you don't have to worry about it, but you notice it because dyld's going to give you much smaller numbers than you'll observe by looking at the clock on the wall. That's expected and understood, and it's everything's going correctly if you see that, but I just wanted to make note of it.

So let's move on, to setting an environment variable in Xcode, you just go to the scheme editor, and you add it like this. Once you do that you'll get the new console log into the output, console output logged.

And what does that look like?

Well this is what the output looks like, and we have a time bar on the bottom representing the different parts of it. And let's add one more thing.

Let's add an indicator for that **400 milliseconds** target, which this app I'm working on is not hitting.

So, if you look in, this is in order basically the steps that Nick discussed in order to launch an app so let's just go through them in order.

### dylib loading

So dylib loading, the big thing to understand about dylib loading and the slowdown that you'll see from it, is that **embedded dylibs can be expensive**.

So Nick said **an average app can be 100 to 400 dylibs**. But OS dylibs are fast because when we build the OS, we have ways of pre-calculating a lot of that data. But we don't have every dylib in every app when we're building the OS. We **can't pre-calculate them for the dylibs you embed with your app**, so we have to go through a much slower process as we load those.

And the solution for this is that we just need to use fewer dylibs and that can be rough. And I'm not saying you can't use any, but there are a couple of options here you can **merge existing dylibs**.

You can **use static archives and link them into both, into apps that way**. And you have an option to **lazy load**, which is to use **dlopen**, **but dlopen causes some subtle performance** and correctness issues, and it actually results in doing more work later on, but it is deferred. So, it's a viable option but you should think long and hard about it and, I would **discourage** it if at all possible.

So, I have an app here that currently has 26 dylibs, And it's taking 240 milliseconds just to load those, but if I change it and **merge** those dylibs into two dylibs, then it only takes 20 milliseconds to load the dylibs.

So I can still have dylibs, I can still use them to share, functionality between my app and my extension, but, limiting them will be very useful.

And I understand this is a tradeoff you're making between your development convenience and your application launch time for your users. Because the more dylibs that you have the easier it is to build and re-link your app in and the faster your development cycles are.

So you absolutely can and should use some, but it's good to try to target a limited number, we would, I would say off hand, **a good target's about a half a dozen**.

### binding and rebasing

So now that we've fixed up our dylib count let's move on to the next place where we're having a slowdown. Between 350 milliseconds in binding and rebasing.

So as Nick mentioned, **rebasing tends to be slower due to IO and binding tends to be computationally expensive but it's already done the IO**. So that IO is for both of them and they're **commingled**, the timing's also commingled.

So if we go in and look at that, all that is **fixing up pointers** in the `__DATA` section. **So what we have to do, is just fix up fewer pointers**.

Nick showed you a tool you can run to see what pointers are being fixed up in the `__DATA`, section, dyld info.

And it shows what **segments** and **sections** things are in, so that will give you a good idea of what's being fixed up.

For instance, if you see a **symbol** to an **ObjC class** in **ObjC section**, that's probably that you have a bunch of ObjC classes. So, one of the things you can do is you can just to **reduce the number of ObjC classes object and ivars** that you have.

So **there are a number of coding styles that are encouraging very small classes**, that maybe only have one or two functions.

And, **those particular patterns may result in gradual slowdowns of your applications as you add more and more of them**. So you should be careful about those.

Now having 100 or 1,000 classes isn't a problem, but we've seen apps with 5, 10, 15, 20,000 classes. And in those cases that can add up to 7 or 800 milliseconds to your launch time for **the kernel to page them in**.

#### C++ virtual functions

Another thing you can do is you can try to reduce your use of **C++ virtual functions**. So **virtual functions create what we call V tables**, which are the same as ObjC metadata in that in the sense that they create structures in the `__DATA` section that **have to be fixed up**. They're smaller than ObjC, they're smaller than ObjC metadata but they're still significant for some applications.

#### Migrating to Swift

You can use **Swift structs**. So **Swift tends to use less data that has pointers for fix-ups** of this sort.

And, Swift is more inlinable and can better co-gen to avoid a lot of that, so migrating to Swift is a great way to improve this.

#### Machine generated codes

And one other thing, you should be careful about **machine generated codes**, so we have instances where, you may describe some structures in terms of a **DSL** or some custom language and then **have a program that generates other code from it**. And if those generated programs have a lot of pointers in them, they can become very expensive because when you generate your code you can generate very, very large structures. We've seen cases where, this causes megabytes and megabytes of data.

But the upside is you usually have a lot of control because you can just change the code generator to **use something that's not pointers, for instance offset based, structures**. And that will be a big win. So in this case, let's look at what's going on here with my, with my load time.

And I have at least 10,000 classes, I actually have 20,000, so many it scrolled off the slide.

And if I cut it down to 1,000 classes, I just cut my launch times, my time in this part of the launch from 350 to 20 milliseconds.

### initializer

So, now, everything but the **initializer** is actually below that 400 millisecond mark, so we're doing pretty good.

So for ObjC set up, well Nick mentioned everything it had to do.

- It had to do class registration,
- it has to deal with the non-fragile ivars,
- it has to do category registration and it has to do selector uniquing.

And I'm not going to spend much time on this one at all, and the reason I'm not is, we solved all of those by fixing up the rebasing and data, and binding before. All the reductions there are going to be the same thing you want to do here.

So we just get a little bit of a free win here, it's small. It's 8 milliseconds. But we didn't do anything explicit for it.

---

And now finally, we're going to look at my initializers which are the big 10 seconds here.

So I'm going to go a little more in depth on this than Nick did.

There are **two types of initializers**,

#### explicit initializers

explicit initializers, things like `+load`. As Nick said we recommend replacing that with `+initialize`, which will cause the **ObjC** runtime to initialize your code when the classes were substantiated instead of when the file is loaded.

Or, in **C/C++** there's an **attribute** that can be **put onto functions** which will cause it to, **generate those as initializers**, so that's an **explicit initializer**, that we just rather you didn't use.

We rather you replace them with call **site initializers**.

- So by call site initializers I mean things like **dispatch once**.
- Or if you're in **cross platform code**, **pthread once**.
- Or if you're in **C++** code, **std once**.

All these functions have basically the same sort of functionality where, any code in one of these blocks will be executed the first time its hit and only that.

**Dispatch once is very, very optimized in our system. After the first execution of it, it's basically equivalent to a no op running past it**, so I highly recommend that instead of using, explicit initializers.

#### implicit initializers

So let's move on to implicit initializers. So inplicit initializers are what Nick described mostly from **C++** globals with **non-trivial initializers**, with **non-trivial constructors**.

- **And one option is you can replace those with call site initializers** like we just mentioned. There's certainly places where you can place globals with non-global structures or pointers to objects that you will initialize.
- Another option is that you don't have non-trivial initializers. So in C++ there's initializers called a **POD** a plain old data. And if you're objects are just plain old datas, the static, or the **static linker** will pre-calculate all the data for the `__DATA` section, lay it out as just data seen there, it doesn't have to be run, it doesn't have to be fixed up.

Finally, it can be really hard to find these, because they're implicit, but we have a warning in the compiler **-Wglobal-constructors** and if you do that it will give you warnings whenever you're generating one of these. So it's good to add that to the flags your compiler uses.

---

Another option is just to rewrite them in Swift. And the reason is, Swift has global variables and they'll be initialized, they're guaranteed to be initialized before you use them. But the way it does it, instead, is **instead of using an initializer**, it, behind the scenes, u**ses dispatch once** for you. It uses one of those call site initializers.

So moving to Swift will take care of this for you, so I highly encourage it that's an option.

Finally, **in your initializers please don't call dlopen**, that will be a big performance hit for a bunch of reasons. When dyld's running it's before the app has started and, we can do things like turn off our locking, because we're single threaded.

As soon as dlopens happened, in those situations, the graph of how our initializers have to run changes, we could have multiple threads, we have to turn on locking, it's just going to be a big performance mess. You also can have subtle deadlocking and undefined behaviors.

Also, please **don't start threads in your initializers**, basically for the same reason.

You can set up a mute text if you have to and mute text even have like, preferred mute texts even have, predefined static values that you can set them up with that run no code.

But actually starting a thread in your initializer is, potentially a big performance and correctness issue. So here we have some code, I have a **C++** class with a non-trivial initializer.

I'm having trouble with the connection. Please try again in a moment. Well, thank you Siri. I'm having a, I have a non-trivial initializer.

And I guess I had it in for debugging all commented out and okay, I'm down to 50 milliseconds, total. I have plenty of time to initialize my nibs and do everything else, we're in very good shape.

---

So now that we've gone through that, let's talk about what we should know if you just, this was really long and pretty dense.

The first one is please use **DYLD_PRINT_STATISTICS** to measure your times, add it to your performance or aggression suites. So you can track how your app is performing over time, so as you're actively doing something you don't find it months later and have trouble debugging it.

You can improve your app launch time by, reducing the number of dylibs you have, reducing the amount of ObjC classes you have, and eliminating your static initializers.

And you can improve in general by using more Swift because it just does the right things.

Finally, **dlopen** usage is **discouraged**, it causes subtle performance issues that are hard to diagnose.

For more information you can see the URL up on screen. There are several related sessions later in the week and again, there's the app performance session from 2012 that goes into the other parts of app launch, that highly recommend you watch, if you're interested.
