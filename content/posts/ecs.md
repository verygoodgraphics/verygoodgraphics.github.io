---
title: "ECS Architecture for GUI applications"
date: 2022-09-14T21:28:17+08:00
lastmod: 2022-09-14T21:28:17+08:00
draft: false
keywords: []
description: ""
tags: []
categories: ["VGG Internals"]
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

ECS, short for Entity-Component-System, is an architectural design pattern originally found in the video game programming. It is a powerful technique that allows easing the programming headaches as well as boosting the performance of a game containing huge amount of interactive objects.

That's why the VGG engine takes ECS as the fundamental architecture, because VGG targets for high performance in the very beginning. However, VGG is more about making interactive applications, particularly the GUI applications. <a name="initial-problem"></a>**Would the ECS architecture still be suitable for such a task?** Let's dive a little deeper.

<!--more-->

## The Essence of ECS

Let's first talk about ECS. There are already many descriptions of ECS around the internet so we just skip the rigorous but to give a one-sentence definition.

> ECS is about entities[^entity] *composed* of difference types of component whose data are manipulated by one or more systems.

[^entity]: An entity can be as simple as an ID, or as complex as a universal container. It doesn't matter, as long as it has the semantics of putting components together, where real data live in.

And the essence of ECS is *composition*, with obvious advantages over the *inheritance* in Object-Oriented-Programming.

In OOP, inheritance is one of three principles, the other two being *encapulation* and *polymorphism*. Inheritance is regarded as a technique for reusing data and functions, for example, as mountain bike is a kind of bike, the `MountainBike` class shall inherit all the data of the `Bike` class as follows
```cpp
struct Bike : public Vehicle {
  Frame  frame;
  Brake  brake;
  Saddle saddle;
  Wheel  wheels[2];
  ...
};
struct MountainBike : public Bike {
  Gear frontGear;
  Gear rearGear;
  ...
};
```

And when an instance is created out of `MountainBike` type, this instance will have all the data in `Bike` type plus all that of its ancestors. In this way, the data together with member functions can be reused in derived types or instances.

However, this leads to many headaches, just to name a few
- __Overhead of virtual functions__: extra runtime cost of virtual table and virtual pointers, let alone some unexpected overriding pitfalls
- __Diamond problem in multiple inheritance__: ambiguous overridden data and functions from multiple ancestors
- __Dynamic type castings__: performance bottleneck of `dynamic_cast` for casting base pointer to derived pointer
- __Complex inheritance chains__: in order to model for real-world problems, complex inheritance relationships has to be designed which would look daunting for new commers

And the developer needs to know the [Object-Model](https://www.oreilly.com/library/view/inside-the-c/0201834545/) well to write memory-safe and performant C++ code, and sometimes he needs to resort to exotic programming idoms such as CRTP[^crtp] or SFINAE[^sfinae] for even better performance.

[^crtp]: CRTP stands for Curiously-Recurring-Template-Pattern
[^sfinae]: SFINAE stands for Substitution-Failure-Is-Not-An-Error

The complexity disappears when we use composition with ECS architecture. If we want to create a new class for `MountainBike`, we needn't to inherit from `Bike`, but only needs to describe the unique data fileds a mountain bike will have. When we need an entity for a mountain bike, we could push both instances of `BikeComponent` and `MountainBikeComponent` into the entity, which are renamed from `Bike` and `MountainBike` respectively without inheritance relationship.
```cpp
make_entity()
  .push(BikeComponent{
    .frame = Frame{},
    .brake = Brake{},
    .saddle = Saddle{},
    .wheels = { Wheel{}, Wheel{} },
    ...
  })
  .push(MountainBikeComponent{
    .frontGear = Gear{},
    .rearGear = Gear{},
    ...
  });
```

And even more components can be pushed into an entity, each of them describes a specific feature that could be freely added to or removed from an entity. As long as proper memory layout is configured for those components for the sake of cache miss ratio reduction, the performance can be quite good.[^ecs-perf]

[^ecs-perf]: As a matter of fact, most ECS-based systems do careful memory layout configuration to improve performance, like [this](https://github.com/amzeratul/halley/blob/develop/src/engine/entity/include/halley/entity/entity.h#L154)

As to systems, which can be as simple as a function, would manipulate those data accordingly. It is exactly what Niklaus Wirth proposed,

> Programs = Data Structures + Algorithm

where components represents data structures and systems represents algorithm. The programming suddenly becomes easier and clearer since there are no more traps hiding bebind various advanced concepts, to be more specific, no more extra runtime cost due to how the compiler implements inheritance, and no more ambiguity issue due to multiple inheritance.

As a matter of fact, ECS is a bit like [Trait](https://en.wikipedia.org/wiki/Trait_(computer_programming)), since trait contains a set of functions, and both of them emphasize the concept of composition. However, a trait class cannot have member data, whereas a component could be more than a POD[^pod] type. All in all, ECS is an architectural design pattern, having no hard-constraints on programming, so
- A component type is free to have inheritance relationships with another component type;
- A component type is also free to have both data and member functions[^sysfunc].

[^pod]: POD stands for Plain-Old-Data-structure
[^sysfunc]: Usually systems account for such functions, but it doesn't matter if a component type has some small or inline functions.

## The Essence of GUI

We already have a rough impression of ECS, and now it's time for discussing GUI. But let's start from a game. A game may consist of many entities, including rivers, mountains, players, NPCs, monsters, etc. These entities usually have no relationships, or more specifically, no hierarchical relationships. In contrast, a GUI is essentially a tree of entities, where hierarchy must be imposed.

{{< figure src="/images/simple-dialog.png" width="50%" >}}

As you can see, the simple dialog above can be divided into three parts: title, content and the button group. The button group contains another two buttons. So the tree looks like this
```
Dialog
├── Title
├── Content
└── Button Group
    ├── Cancel Button
    └── OK Button
```

where the title and content are instances of text widget[^widget], button group is of layout widget, and buttons are of button widget. If we see each widget an entity, these widgets comprise a hierarchical entity tree.

[^widget]: In Web programming, specifically React-based web programming, we have another *component* concept, which actually means reusable GUI widgets. To distinguish from our *component* concept in ECS, we just call this reusable GUI component __widget__.

So the [initial problem](#initial-problem) becomes, would the ECS architecture be suitable for modeling such a hierarchical system?

And from another aspect of view, would it be an overkill to use ECS architecture for GUI applications? Since ECS architecture could process up to millions of entities, do we really need to process such a magnitude of entities in even most complex GUI like the following example?

{{< figure src="/images/maya.png" width="100%" >}}

We'll figure out the answers in the next chapter.

## ECS for GUI

As we have mentioned in the first chapter, the data are defined by the components and the systems will update the data constantly during the whole execution lifetime. This leds to an interesting question, how does the system find the appropriate data? The answer is through, in our definition, queries.

Different ECS libraries have huge discrepancy with respect to the implementation details, like how the components are organized in memory and so as with the query implementation. But this won't stop up dividing the queries into two different semantic types, horizontal query and vertical query.

### Horizontal Query

Let's have a look at the adapted code snippets from the moust famous C++ ECS framework [`EnTT`](https://github.com/skypjack/entt)
```cpp
void update(entt::registry &registry) {
    auto view = registry.view<position, const velocity>();

    // use a callback
    view.each([](auto &pos, const auto &vel) { /* ... */ });

    // use a range-for
    for(auto [entity, pos, vel]: view.each()) {
        // ...
    }

    // use forward iterators and get only the components of interest
    for(auto entity: view) {
        const auto &vel = view.get<velocity>(entity);
        // ...
    }
}
```

In the above code, the `update` is a simple system, which developer could utilize to update the entity position based on its current velocity. The system at first queries all entities that have both component of `position` and `velocity`, save the result in a view, and then iterate over each entity for the updating.

{{< figure src="/images/ecs-query.png" width="100%" >}}

If we visualize this process, we will find out that the query is **horizontal**. It doesn't matter whether there is a hierachy or not because of possibly flattened storage and indirection of the relationship.

Horizontal queries are common in video game programming, but also exist in GUI applications. Let's have a look at the following example.
```html
<div id="entity-a" class="entity">
  <div class="position-component"></div>
  <div class="velocity-component"></div>
  <div id="entity-b" class="entity">
    <div class="position-component"></div>
    <div class="velocity-component"></div>
  </div>
</div>
```

If we use CSS selector `.position-component`, we will get an array of `div`s whose class is `position-component`. The embedded hierarchy won't affect the final result.

Say, if we want to mimic the CSS selector behavior during GUI programming, we can just take advantage of the horizontal query ability of ECS, which means ECS at least suits this particular use case.

### Vertical Query

Compared to horizontal query, vertical query focus on querying *entities*, rather than *components*. So vertical query cares about a specific *named* entity, looking into the internals and finding all the components of it.

Let's still take the dialog example for discussion. In order to start iterating through the entity tree, we need to find the root entity, namely the `dialog entity` in this example. After that, we search for what components this entity has, or which children it holds, then possibly recurring into each of children for further investigations.

Here comes an interesting question. How do we model the hierarchy relationship between entities?

There are mainly two methods,

- We could hardcode the relationship into the entity data structure; or
- We could implement a relationship component which is hot-pluggable into an entity.

Each of the two methods works, but most mature ECS libraries, including [EnTT](https://skypjack.github.io/2019-06-25-ecs-baf-part-4/) and [flecs](https://ajmmertens.medium.com/building-games-in-ecs-with-entity-relationships-657275ba2c6c), prefer the latter, so does VGG engine.

The most important advantage is the dynamic ability to add or remove a relationship, compared to the hardcoded method. Because sometimes we need more than one tree hierarchy during the whole lifetime. Take chromium engine for [instance](https://developer.chrome.com/articles/renderingng-architecture/#architecture-components), during the rasterization pipeline, we need at least three trees, including DOM tree, layout tree and the property tree, which could be peeked in the following figure from their offical blog.

{{< figure src="/images/chrome-pipeline.avif" width="80%" >}}

A single node entity could be reused across multiple trees, as long as multiple relationship component is added into the entity, which is exactly what ECS excels at.

### Event Mechanism

In *interactive* GUI applications, how the architecture handles interactions is quite important, which is what we call the event mechanism. The *signal* concept, taken equally as the *event* concept, would be just called event in our context.

Strictly speaking, the event mechanism should not be part of ECS architecture, but as it plays an important role in interactive apps and games so good support of it is common in ECS frameworks. For example, the EnTT library proposed [four distinct concepts](https://skypjack.github.io/entt/md_docs_md_signal.html) for the entire event mechanism, namely the delegate, the signal, the dispatcher and the emitter.

We won't bother those abstract concepts but just describe what we expected in such a framework

- The ability to define various types of events, including
  - The inner events, which are system-generated events like the creation and destruction of an entity, as well as the addition and the deletion of a component from an entity
  - The outer events, which are user-generated events like mice and keyboards events, plus user-defined events which could be freely sent from one entity to another
- The ability to send events to multiple entities, like broadcasting or multicasting messages in a network, and possibly across different threads or processes.
- The ability to respond to those events, using pre-registered callbacks for each type of event, which itself could be dynamically added or removed
	- The responding process, a.k.a. invoking callbacks, should be asynchronous, which means it won't make GUI unresponsible. Note this is orthgonal to multi-threading since even single thread is capable of parellel computing using technique like coroutine.
- The ability to queue, delay, filter, even compose events for more complicated behaviors

More descriptions are welcome but the above abilities should be sufficient for a simple GUI framework.

Let's look into a famous non-ECS GUI framework, Qt. Qt uses a [signal-slot mechanism](https://doc.qt.io/qt-6/signalsandslots.html) as its core event machanism, which could be summarized as follows

- All object should be inherited from `QObject` in order to use signal-slot mechanism
- A meta-compiler is needed to transform signal-slot related code into normal C++ code
- Signal and slot is of many-to-many relationship, which are connected by `connect` function. The executing of slot function could happen in another thread depending on [connection type](https://doc.qt.io/qt-6/qt.html#ConnectionType-enum)
- The exchanged data are defined by the arguments of signal and slot functions, which are type-checked by the meta-object system

We can find out that except for advanced ability like event composing, Qt's signal-slot mechanism loosely conforms to our expectations.

As to ECS-based GUI framework, we could claim that those expectations should be satisfied more easily due to its flexibility, for example, we do not need a meta-compiler and singular inheritance. The data could be exchanged with a POD-style component across entities as well. We will leave it to implementation details or possibly another blog post in the future.

### Summary of ECS for GUI

We have already discussed several use cases and come to an conclusion that ECS could be even a better solution than traditional OOP-based solution for the GUI programming problems.

And also please remember, VGG is utilizing designs as the base of a GUI. A common design would have up to 10-40k layers, even 250k layers.[^hn-penpot] This places a huge performance challenge so we think ECS is just right here to rescue.

[^hn-penpot]: From this [discussion](https://news.ycombinator.com/item?id=33013123) on Hacker News

## Conclusion

In this article, we have discussed the essence of ECS architecture and GUI, and elaborated several use cases of ECS in GUI programming. This is a very big and deep topic so it is impossible to cover everything in a single post, for instance, we haven't mentioned concrete implementation details of such an ECS architecture specialized for GUI programming.

There are many good ECS implementations around the internet, for example
- [EnTT](https://github.com/skypjack/entt) is implemented with a custom [sparse-set](https://github.com/skypjack/entt/blob/master/src/entt/entity/sparse_set.hpp) data structure.
- [flecs](https://github.com/SanderMertens/flecs) is implemented with a traditional [archtype](https://ajmmertens.medium.com/building-an-ecs-2-archetypes-and-vectorization-fe21690805f9) method, where the addition and deletion of component could be a theoretical performance bottleneck in common scenarios.
- [halley](https://github.com/amzeratul/halley) engine, which is a game engine successfully shipped a game called [Wargroove](https://store.steampowered.com/app/607050/_/), implements a [custom](https://github.com/amzeratul/halley/tree/develop/src/engine/entity/include/halley/entity) ECS architecture

VGG engine does not use a baked ECS framework like EnTT or flecs but to choose to implement one on our own, just like what halley has done. This is because we believe a tailered version better suits for our use cases, particularly for programming GUI applications. Currently VGG implements a simple ECS architecture which still lacks some features, but the inner iteration is happening. If you like the concept of [Design-as-Code](/posts/intro-vgg/) as well as the ECS architecture, you can keep an eye on our [open-sourced](https://github.com/verygoodgraphics/vgg_runtime) version.
