# Introduction

So yeah, I am writing a Wayland compositor (actually, more like a library for compositors). Since I am in this rabbit hole already, I thought I might as well document my journey through it.

This series isn't meant to be a tutorial on how to write a Wayland compositor in Rust (although, if this project ends up being successful, then I would need to write one of those too). It's more of a record of my struggle with Rust and/or Wayland. Consequently, these chapters aren't going to be in a logical order, I am just writing things down as I encounter them.

## Why?

Well, there are a couple of reasons. Understanding code, even those written by one's self, is hard. Having a written record will help future me understand why certain design decisions are made.

And since I don't consider myself an expert on Rust, and _definitely_ not one on Wayland either, I hope other people reading this can point out things I missed and perhaps lead to a better design.

Lastly, it's difficult to remain motivated on a big project like this. Knowing there are other people caring about this project would help me a lot. How do I gather some attention without something concrete to show? Well, I write about it.

## OK, but why a Wayland compositor?

Because it's fun, because I want to see Wayland, _and_ Rust thrive.

Personally, I maintain [a small compositor for X11](https://github.com/yshui/picom), so when I say the X11 design is showing some age, I am speaking from experience. On the other hand, the Wayland design does look a lot better. And it's also where the most of the effort seems to be focused on - I want in on all the fun too!

And since the compositor is security critical, Rust seems to be a good candidate to write it in. Additionally I want to try out the new, shiny async/await feature of Rust as well.

## Prior art

Obviously, I am not the first person to write a Wayland compositor in Rust. People have tried/been trying to do the same thing. Learning from others is a good a idea, so I think I need to talk a bit about them.

First of all, yes, I am aware of the [way-cooler project](https://github.com/way-cooler/way-cooler), the [failed](http://way-cooler.org/blog/2020/01/09/way-cooler-post-mortem.html) attempt of writing a Wayland compositor in Rust. Many may point to that and say Rust isn't a suitable language choice. But, not to sound arogant, I think I can do better. If nothing else, at least I can learn from their experiences.

And people have indeed been able to do better! [The Smithay project](https://github.com/Smithay/smithay)! Not sure if you have heard about it, but they are a suite of Rust libraries for writing Wayland compositors and clients in Rust. In other words, their goals overlap with mine.

So why am I starting my own library? Without going in to details (as that's what the rest of this series is about), this project has, in general, a different design style to Smithay. It also a smaller scope, which I hope could lead to a clearer design.

## Expectation (of the reader)

The reader (you) is expected to have some moderate understanding of Rust. But I don't expect you to know anything about Wayland. Basically, me when I started this project.

## Expectation (of me/of this project)

So, what's this project currently at?

Well, have a look:

<video controls>
    <source src="assets/intro_demo.mp4" type="video/mp4">
</video>

Voilà! Stuff on the screen! Other than that, pretty much nothing else works. Windows are upside down, no mouse/keyboard, things aren't scaled correctly, yada yada.

There is still a long way to go. And there is no guarantee this project would succeed. I hope it does, if not, I hope at least what I wrote here is entertaining.
