---
layout: post
title:  "Tiny Simulator: A graphical processor simulator for education"
date:   2020-08-24 11:21:09 +0100
---

Advanced computer architecture concepts can be extremely difficult to grasp for
computer science students. At least, that was my experience back when I was an
undergraduate student at university. There are certainly wonderful resources
like the legendary Hennessy and Patterson books and, of course, Wikipedia. But
these mostly provide prose descriptions of complex algorithms and concepts that
I often found difficult to follow. So I would have to pour over the
explanations for hours with pen and paper in an attempt to visualize what was
happenning inside the processor.

My undergraduate years are long past, but I still decided to try to address the
visualization problem. Therefore, I put together a processor simulator that I
call [Tiny Simulator](/tiny-simulator). It runs on a web browser and provides a
graphical interface along with simple controls to visualize the flow of data
through the processor's pipeline as instructions are executed. [Tiny
Simulator](/tiny-simulator) demonstrates computer architecture concepts like
reservation stations, register renaming, branch prediction, etc.


The simulated processor implements a simple, made up Instruction Set
Architecture with enough instructions to write basic programs. In fact, a
handful of programs, like bubblesort and factorial, are provided as
demonstrators. The idea is to keep it simple enough that anyone familiar
with basic computer architecture can easily pick it up and learn about more
advanced techniques.

As always, please let me know if you find [Tiny Simulator](/tiny-simulator)
useful, have a question or find an issue!
