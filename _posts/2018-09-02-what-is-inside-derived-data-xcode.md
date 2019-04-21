---
layout: post
title: What's inside the Derived Data folder?
filename: "2018-09-02-what-is-inside-derived-data-xcode.md"
---

Deleting derived data - a well know trick that comes in handy every time Xcode behaves strangely for no obvious reason. I still clearly remember when my senior told me about this basic iOS dev trick for the first time.

![Derived Data folder](/images/2018-09-02/derived-data-1.png)

 As years went by, and with more experience gained, I started to understand what kind of errors can be fixed like that. However, I never really understood **what exactly is inside the DerivedData folder**. I decided to change that and here are my findings.

<!-- more -->

> *Note:* DerivedData content varies with the Xcode version. I used Xcode 10.0 beta 6 in this post.

I completely deleted existing DeriveData folder. Then I created a sample project called `DDExample` from the single view app template and opened it in Xcode.

Xcode immediately creates a new Derived Data folder with two subfolders - one named `ModuleCache` and one with the name of the project followed by some kind of hash.

![Derived Data](/images/2018-09-02/derived-data-2.png)

---

### Module Cache

![Derived Data](/images/2018-09-02/derived-data-3.png)

As the name suggests, this is where Xcode stores precompiled module files (`.pcm`). A module is the way how reusable code is organized and shared. Modules were introduced to Clang (the compiler used by Xcode) several years ago, mainly to ensure reasonable and scalable compile times. It was common that for every single `import` in the source file, megabytes of additional headers had to be included and parsed by the compiler. Thanks to modules, the headers are parsed and compiled just once.

You can see two subfolders named `AIEKQT3S8ZS7` and `391J0EBN0O3XH` there. The number of those folders and their names will be most likely different on your machine. The name of each subfolder refers to a hash computed from the arguments passed to the compiler. The more unique compiler configurations your project uses, the more subfolders with `.pcm` files are in this folder. Each subfolder contains the same set of `.pcm` files precompiled with given arguments.

> More info about this process: [Behind the Scenes of the Xcode Build Process](https://developer.apple.com/videos/play/wwdc2018/415/).

What's worth to explicitly mention is that the `ModuleCache` folder is not project-specific and is **shared among all your projects**.

---

### Index

![Derived Data](/images/2018-09-02/derived-data-4.png)

Here is where Xcode stores the data gathered during the indexing phase. This data is used for search, quick navigation and refactoring within the project. Prior to Xcode 9, the data was stored using [SQLite in a human-readable form](http://spaceisdisorienting.com/deciphering-Xcodes-index).

With Xcode 9, Apple changed the way the index data is stored and is now using [LMDB](https://en.wikipedia.org/wiki/Lightning_Memory-Mapped_Database).

![Derived Data](/images/2018-09-02/derived-data-5.png)

That's not really big of a deal because you can still open and inspect the `mdb` files. ~~However, instead of human-readable keys, Apple is using some kind of hashes.~~ However, Apple is using binary keys and they are not human readable.

![Derived Data](/images/2018-09-02/derived-data-6.png)
> *I used [FastoNoSQL](https://fastonosql.com/) to open the `mdb` file.*

I can't tell you more about how the current format works because I couldn't find any additional info about the subject. Feel free to comment below or ping me on [Twitter](https://twitter.com/vojtastavik) if you have more info.


---

### Logs

![Derived Data](/images/2018-09-02/derived-data-7.png)

Inside this folder, Xcode stores all kinds of logs divided by domains (`Install`, `Build`, etc.). Remember that I still haven't built the project so the `Build` logs folder is empty. See what happens when I build the project:

![Derived Data](/images/2018-09-02/derived-data-8.gif)

> More info about [Test logs in Xcode](https://michele.io/test-logs-in-Xcode/).

---

### TextIndex

![Derived Data](/images/2018-09-02/derived-data-9.png)

I have no idea what this folder is for. I wasn't able to find a single piece of information that mentions this folder or file. **If you can help shed some light on this, please comment below or ping me on [Twitter](https://twitter.com/vojtastavik).**

---

### Build/Products

![Derived Data](/images/2018-09-02/derived-data-10.png)

Probably the most important folder inside Derived Data! Here is where your final `.app` file (folder) is assembled. This is the actual file that gets copied and installed to the simulator (or an iOS device).

You can see that I've built the app in the Debug configuration targeting iOS simulator. When I build it for the actual device, `Debug-iphoneos` subfolder is created.

![Derived Data](/images/2018-09-02/derived-data-11.png)

---

### Build/Intermediates.noindex

![Derived Data](/images/2018-09-02/derived-data-12.png)

And finally the last folder I'd like to mention: `Intermediates`. This is the place where Xcode's build system writes auxiliary files needed for building the app. Those files are also cached here and reused later. The build system is smart enough to not recompile files that haven't been changed.

Look what happens when I change something in the `AppDelegate.swift` and build again. You can see that only AppDelegate auxiliary files were updated.

![Derived Data](/images/2018-09-02/derived-data-13.png)

---
*EDIT (9.9.2018):*
- *Updated info about binary keys*
- *Added FastoNoSQL link*
