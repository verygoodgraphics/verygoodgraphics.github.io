---
title: "Introducing VGG and Design-as-Code"
date: 2022-05-12T17:38:17+08:00
lastmod: 2022-05-12T17:38:17+08:00
draft: false
keywords: []
description: ""
tags: []
categories: ["VGG Products"]
author: Harry

# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: true
autoCollapseToc: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright:
reward: false
mathjax: false
---

VGG is yet another engine and framework for making interactive apps, which emphasizes the concept of _Design-as-Code_ at its core. And it may hopefully unite the worlds of design and code at last.

Meanwhile, the whole world is about using Web technologies, platform-specific frameworks, or cross-platform frameworks, to make apps, the most famous and innovative among them being [Flutter](https://flutter.dev/), which takes advantage of underlying parts of browser engine to achieve the goal of "Write Once, Run Anywhere"[^wora].

[^wora]: Sounds like Java right? But for this time, it's not about programming languages, but about GUI frameworks.

So why do we need another engine or framework? That's what we need to explain about Design-as-Code, but let's look at other choices first.

<!--more-->

## The Web Technologies

Writing HTML[^wh] [sucks](https://rogovoy.me/blog/writing-html-sucks).

[^wh]: Here writing HTML actually means writing HTML+CSS.

It is very interesting. Because HTML, short for `Hyper Text Markup Language`, was originally invented thirty years ago for making **linkable** documents, rather than apps. It is more about **markup** than a language. The majority doesn't even take writing HTML as programming, since it is not a programming language at all.

It was not until the advent of HTML5, which is commonly referred to as **H5**, together with CSS3 and modern versions of JavaScript, that people were starting to make **real** Web apps.[^webapp]

[^webapp]: Actually the trend started more earlier with the advent of Web 2.0 and AJAX technology, but those made are more of web-site than web-app, like Facebook.

And thanks to the optimization team of browser vendors, nowadays the performance of JavaScript gains the magnitude-level improvements so that the **interactive** web apps won't be bottle-necked.

{{< figure src="/images/js-perf-his.png" width="50%" >}}

Do you remember this famous picture? It's from [Lin Clark](https://hacks.mozilla.org/2017/02/a-cartoon-intro-to-webassembly/) when she tries to introduce the performance history of JavaScript. The performance started to boost from 2008 and is still not reaching the end.

The benefit is great for app end-users, however, writing HTML still sucks for coders.

When we have so-called Front-end Frameworks, such as React, Vue, Angular and etc, and is overwhelmed by the booming of front-end ecosystems[^feco], we still need to write **HTML-like** code and tweak styles carefully with CSS or one of its variants.

[^feco]: Let's take the number of NPM packages for example, and we will see.

It's slow and boring.

Tens of thousands of greatest minds are wasted on implementing those visual interfaces, using a technology stack that was not originally intended for. Those tasks are not limited to tweaking styles back-and-forth, as well as thinking about browser compatibility issues, weird CSS hacks, performance caveats and so on.

And No-Code [doesn't help](https://rogovoy.me/blog/writing-html-sucks) either. Anyway, seldom No-Code products are solving problems for coders.

We may ask ourselves, why are we still sticking to HTML and CSS, and using sophisticated tools from over-grown NPM ecosystem to create apps, which is less efficient and more irritating?

## Other Non-Web Frameworks

Because there is no other better ways.

Electron, though well-known and having a proven application like VSCode, would be first ruled out from the list, since it is actually still a Web-based solution, thus we don't even need to talk about its performance issues.

What else do we have?

- Some are platform-specific frameworks since the beginning of golden ages, like MFC for Windows, Cocoa for macOS, and GTK for UNIX/Linux. And others are modern mobile toolkits like those specifically made for iOS, Android, or other mobile operating systems.

- The cross-platform frameworks, notably the widely-adopted Qt Framework. But it is mainly used for non-mobile/non-web environments. The _cross-platform_ here means crossing desktop OS initially, while endeavour is still being put into mobile/web targets for those frameworks.

- Upcoming brand-new solutions like Flutter, which is a mobile-first cross-platform framework, and is promising for web and desktop as well.

As the ratio of web apps is increasing over the years, the platform-specific or cross-platform frameworks mentioned above are less used since they are often targeting traditional desktop applications.

And the developer experience is even worse than writing HTML because they may be obliged to write _imperative_ and _object-oriented_ code like this[^coderef][^qtdesigner],

```
var count = 0
let stack = new VStack
let text = new Text("Count: \(count)")
stack.add_child(text)
let button = new Button("Increment")
button.set_onclick(||
    count += 1
    text.set_text("Count: \(count)")
)
stack.add_child(button)
```

rather than writing **declarative** and possibly **reactive** code that programmer always dreams of, like this.[^coderef]

```
struct AppState {
    count: i32
}

VStack {
    Text("count: \(state.count)")
    Button("Increment") {
        state.count += 1
    }
}
```

[^coderef]: Code samples taken from Raph Levien's [_Towards a unified theory of reactive UI_](https://raphlinus.github.io/ui/druid/2019/11/22/reactive-ui.html)
[^qtdesigner]: But there is Qt Designer that will do the dirty work for Qt developers.

That's why Flutter seems like a silver-bullet for developing apps:

- It is declarative and reactive in nature.
- It is truly cross-platform for making all of desktop, mobile, and web apps.

Though some people don't like Flutter because it introduces another new and unfamiliar language as well as extra VM burden, which is likely the result of technical bureaucracy.

The real problem of Flutter lies in the compatibility with existing ecosystems, as people are inclined to reuse existing resources and sticking to old but mature apps. And programming language matters for the same reason.

And it also explains why the JavaScript version of Flutter was tried by some gifted and insightful people. But it failed as Flutter itself is rapidly changing its internals so that it won't catch up. Thankfully, as all work pays off, it gave birth to the [Kraken](https://github.com/openkraken/kraken) framework, which allows coders to write HTML, and uses Flutter as the basis for cross-platform rendering.

Wait... What? Writing HTML again in non-web frameworks[^rn]?

[^rn]: Similar work is done in [React Native](https://reactnative.dev/), which allows coding in React but rendering with native system widgets.

## Design-to-Code

No, no more writing HTML!

Still, we have to admit that HTML+CSS is a good combination to represent UI, as

- HTML is responsible for the structure of the content,
- and CSS is individually responsible for the style of the content.

So that structure and style are de-coupled, which is good for engineering. The premise is that the engineering is necessary, however.

In practice, the engineering of UI is sometimes meaningless and unnecessary. Suppose we already have a high-fidelity design prototype given by designers, and what coders need to do next is

1. Re-implement the design prototype using code, which is HTML+CSS stuff in 99% cases.
2. Add business logic to the UI he or she just re-implemented.

The first part is always the source of pain. It has plenty of details. It is time-consuming. It needs discussions with designers, back-and-forth. The communication cost is expensive. If the design updates, the code needs update too, and maybe another costly discussion is needed.

And not to mention that, this kind of job is usually seen as low-ended, and thus the so-called front-end programmer is usually looked down upon by other non-front-end programmers.[^fep]

[^fep]: One evidence is that front-end programmer gets lower salary standard. The FE programmers always come and go, but good FE programmer is rare and is eagerly demanded by the market, which is really weird.

Some clever people came out of the solution of **Design-to-Code** using compiler, or more specifically, the transpiler technique, which turns the whole high-fidelity design into machine-generated HTML+CSS code.

It sounds adorable, but it is catering for PMs and designers, rather than developers. The nasty intrinsics include but is not limited to

- The generated code is ugly, or at least not conformed to current coding style.
- The integration of generated code is troublesome. What if it depends another 3rd-party library? What if the generated code got updated and whole chunk of changes is thrown to the version control system?
- The design tool like Sketch or Figma, is always more visually-capable than HTML+CSS for advanced visual effects, thus sometimes the generated code could not produce exactly the same UI of design prototype, and some patches are definitely needed. What if the generated code got updated and the patches won't apply any more?

All in all, the Design-to-Code is not a good technical solution from coder's perspective. Now let's look what **Design-as-Code** is.

## Design-as-Code

In a Design-as-Code solution, no more HTML needs to be written, or generated. Even HTML itself does not exist.

Because the design just replaces the role of HTML+CSS, so that design is just code.

The obvious advantage is that the designing and developing of graphical UI needs to be done only once, because two things is actually one thing. And consequently less discussions are needed, which make both sides happier.

As for developer, he or she is able to build business logic directly on the design prototype, which is then runnable by VGG engine as a whole, like a real application. This will save a lot of duplicated work so as to increase the work efficiency not only for developer but also for the whole team.

The concept of _Design-as-Code_ is simple, however, it faces many challenges, including those in compiler, programming languages and computer graphics, and the last of which is where the name of VGG came from. VGG is short for _Very Good Graphics_, because it is initially just a design rendering engine. As it develops, more and more interesting ideas and experiments are carried out in the engine, and it turns out to be a brand-new framework for developing interactive applications.

## Conclusion

In this post, we have discussed about the Web technologies, platform-specific frameworks, cross-platform frameworks, the Design-to-Code solution as well as the Design-as-Code concept.

We proposed Design-as-Code concept for escaping from the HTML world, and introduced VGG as a brand-new engine and framework of D-a-C for developing interactive applications. But VGG is not finished yet, as many technical challenges needs to be overcome first.

This post is just an introduction, and more details about VGG engine will be discussed in later posts. If you have interests, please keep an eye our official [blog](https://blog.verygoodgraphics.com/) and our open-sourced version of VGG [engine](https://github.com/verygoodgraphics/vgg_runtime).

## Contact us

- Discord: https://discord.gg/g3HJuKP54D
  <img width="595" alt="Group-Eng 16 59 55" src="https://user-images.githubusercontent.com/111478642/196389211-64cfa786-dd66-4ae0-b7ec-b81e04ac274f.png">
