---
layout: post
title:  "What's So Good About RISC V?"
date:   2022-08-14 11:21:09 +0100
---

I was fresh out of university when I first heard about RISC V. The [RISC V
Foundation](https://riscv.org/about/history/) just set up shop and the big
guys, like Google and IBM,[^1] were throwing their weight behind it. There was
suddenly renewed interest on open-source processors, like
[BOOM](https://github.com/riscv-boom/riscv-boom), and many IP businesses
started emerging alongside RISC V. But "open" hardware or Instruction Set
Architectures (ISA) are clearly not novel ideas,[^2] [^3] [^4] and most
certainly, neither are Reduced Instruction Set Computers (RISC). So what is so
good about RISC V?

I pondered this question for some time and I will humbly provide my
perspective. I first worked with computer architectures while
pursuing a PhD. I then joined ARM as a CPU Architect for about a year before
joining Codasip where I currently work. ARM is an established IP business that
owns its well-known ISA while Codasip, among other
things, provides RISC V IP and is a founding member of the RISC V Foundation.
In short, I worked with both ARM, the incumbent, and RISC V, the challenger, as
an *engineer*, so hopefully my perspective provides a slightly different
insight.

I would like to stress that the opinions expressed in this post are solely my
own and do not express the views or opinions of my employer. My opinion is of
course that RISC V is a great project and here are three reasons why...

# 1. It's Free!

And again, **it's FREE!** Well, actually RISC V is *sort of* free. The ISA is
free and "open" in the sense that anyone can download the manuals and use them
however they like. For example, an enterprising group of engineers can
develop and commercialize RISC V processors without paying a penny in licenses
or royalties for the ISA. Anyone can configure or customize RISC V however they
see fit and use it for research, business or any other purpose. Even better,
anyone can choose to ignore RISC V compliance and *modify* it however they see
fit to make a new ISA based on RISC V -- we could call it RISC VI or RISC V++!

But there is a (tiny) catch! You need to be a member of the RISC V
International Association to really be part of RISC V.  RISC V International
oversees and fosters the RISC V community including its strategy, technical
committees and workgroups, technical deliveries, etc. You must be a member if
you want to participate in discussions, simply ask a technical question or have
any kind of influence or voice in RISC V.[^5] Of course, becoming a member of
RISC V International costs $$$!

Therefore, RISC V is more of an *open standard*, like Bluetooth[^6] or DDR[^7],
as opposed to a regular open-source project. The ISA specifications are
available online and you can submit patches via GitHub,[^8] but anything
significant, like a new instruction, will probably be dismissed if you are not
a member and the ideas were not previously discussed in members-only forums.

**NOTE:** The RISC V International Association is a non-profit organization
incorporated in Switzerland back in 2020 to "manage strategic risk for [the]
community investing in RISC V for the next 50+ years".[^9] Feel free to read
that in whatever geopolitical context you like!

The good news is that RISC V International memberships are relatively cheap,
that is, when compared to buying an architecture license. Also,
you do not have to ask anyone's permission to use RISC V. You just need to get
on with the work! In contrast, you cannot touch proprietary IP with a ten-foot
pole for ages while your organization negotiates the purchase of an
architecture license with, say, ARM. And most importantly, RISC V gives you the
freedom to configure, customize or modify the ISA *however you want* without
having to ask anyone for permission, and this brings us to the second point...

# 2. Innovation

Imagine that you are designing a new processor optimized for machine learning
applications, so you decide that it needs a matrix coprocessor. Your new
processor will implement a proprietary ISA, but there is a problem! Your legal
department is saying, after the requisite few weeks of delays, that you are
forbidden from extending the proprietary ISA with custom instructions to easily
interface with your matrix coprocessor even though you purchased an
architecture license. Perhaps the restriction is in the license agreement!
What do you do?

Situations like this happened before. Apple wanted a matrix coprocessor, known
as Apple Matrix coprocessor (AMX), in their M1 chip which implements the
(proprietary) ARMv8-A ISA.[^11] However, ARM "has long resisted adding custom
instructions to their ISA" as Erik Engheim pointed out.[^10] Apple did not end
up extending the ISA in the usual way; AMX instructions are posted to the
coprocessor via the processor's store unit instead.[^11] Also, Apple did not
document AMX and we are only aware of it thanks to Dougall Johnson's reverse
engineering efforts![^11] The idea of matrix coprocessors caught on and ARM
eventually introduced a proper Scalable Matrix Extension (SME) for their
ARMv9-A ISA although this took years and the M1 was in production by then.[^12]

The point of this story is that closed, proprietary ISAs restrict innovation.
You cannot do whatever you want with those ISAs without asking permission, and
permission may not always be granted![^13] This is not a problem with
an open ISA like RISC V. You do not have to ask permission to extend
the ISA if the current specification does not suit your needs. Extending RISC V
is even encouraged and the designers helpfully point out free opcodes for
custom instructions in the ISA specification![^14] There is also a huge amount
of freedom to tune the core components of the ISA in an implementation, like
the memory model.

The freedom to extend the ISA allows engineers to create innovative and unique
RISC V processors. For example, you can be sure that two processors
implementing, say, the proprietary ARMv8-M ISA only differ in their
microarchitecture (e.g. slightly different pipeline, perhaps a coprocessor,
etc). In contrast, two RISC V processors could have a different
microarchitecture *and* implement different (custom) ISA extensions such that
one processor is great for machine learning while the other is great for
real-time applications. The freedom to innovate creating these
*domain-specific* ISA extensions, not just the microarchitecture, is specially
important as we reach the end of Moore's law and Denard scaling.

Extending the ISA is so appealing that a few years ago ARM opened up the
possibility for licensees to build custom instructions.[^15]

Additionally, the freedom to innovate with RISC V enables businesses to come up
with products that have clear differentiator features, like custom instructions
and coprocessors, and even entirely novel products or services. RISC V also
opens up great opportunities for research. And this brings us to my final
point...

# 3. Community

An ISA would not be complete if it does not have a great community around it.
RISC V absolutely nailed this point! A great open-source, research and business
community sprung up to create a rich software (compilers, operating systems,
tools, etc) and hardware (processors, coprocessors, etc) ecosystem and
offer IP or services around the ISA.

My employer, [Codasip](https://codasip.com/), is a great example of a business
in the RISC V community.[^16] Codasip's flagship product, Codasip Studio, is an
innovative EDA toolset that allows engineers to quickly design and customize
processors *and* ISAs. Of course, you can use Studio to customize any ISA,
but it would be a hard sell if an open ISA, like RISC V, was not available and
customers could only rely on proprietary ISAs that restrict innovation. Similar
to Codasip, there are many other businesses and projects who depend on an open
ISA, like RISC V, to exist.[^16]

# Conclusion

So what's so good about RISC V? It is free, enables innovation and has a great
community around it! Those are, in my opinion, the three key points that
make RISC V a compelling ISA today. Personally, this is also why I decided to
join the RISC V community!

# References

[^1]: [Here](https://riscv.org/members/) is a list of members in the RISC V Foundation.
[^2]: [OpenRISC](https://openrisc.io/)
[^3]: [OpenPOWER](https://openpowerfoundation.org/)
[^4]: [OpenSPARC](https://www.oracle.com/servers/technologies/opensparc-overview.html)
[^5]: [RISC V Technical Forums](https://riscv.org/technical/technical-forums/)
[^6]: [Bluetooth SIG develops and maintains the Bluetooth standards](https://www.bluetooth.com/about-us/)
[^7]: [JEDEC develops and maintains the DDR (and many other) standards](https://www.jedec.org/about-jedec)
[^8]: [RISC V's GitHub](https://github.com/riscv)
[^9]: [RISC V history](https://riscv.org/about/history/)
[^10]: Erik Engheim, The Secret Apple M1 Coprocessor, see [here](https://medium.com/swlh/apples-m1-secret-coprocessor-6599492fc1e1)
[^11]: The Apple Matrix coprocessor (AMX) is largely undocumented. Developer Dougall Johnson has reversed engineered a lot of it. See [here](https://gist.github.com/dougallj/7a75a3be1ec69ca550e7c36dc75e0d6f)
[^12]: ARM, Introducing the Scalable Matrix Extension for the Armv9-A Architecture, see [here](https://community.arm.com/arm-community-blogs/b/architectures-and-processors-blog/posts/scalable-matrix-extension-armv9-a-architecture)
[^13]: There certainly are advantages for tightly controlled ISAs. For example, the ISA does not "fragment" in the sense that processors implement the same core set of instructions, so it is easier to have software that is written once and runs everywhere. I personally still favor innovation! Solutions to mitigate problems like fragmentation exist anyway.
[^14]: See Chapter 26 (Extending RISC V) from [The RISC V Instruction Set Manual Volume I: Unprivileged ISA.](https://riscv.org/technical/specifications/)
[^15]: Kevin Krewell, Arm Responds to RISC V, and More, see [here](https://www.eetimes.com/arm-responds-to-risc-v-and-more/#)
[^16]: Many of these businesses are also RISC V members. See [here](https://riscv.org/members/)
[^17]: Kevin Krewell, Arm Responds to RISC V, and More. See [here](https://www.eetimes.com/arm-responds-to-risc-v-and-more/#)
