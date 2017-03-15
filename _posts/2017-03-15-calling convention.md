偶然看见的关于calling convention的万字雄文，不过写于2009年，现在情况可能有所变化了，比如Qt中的函数有的时候会被编译成`__stdcall`方式，而不是C++默认的`__cdecl`。但还是一篇价值很高的文章。

关于calling convention在Qt中的使用，我得到的建议是：除了`extern C`以外不要用。因为它改变了类型系统。把这个问题交给编译器去处理。

作者：[Thiago Macieira](http://blog.qt.io/blog/author/thiago/Thiago).

原帖地址： [Some thoughts on calling convention](http://blog.qt.io/blog/2009/08/15/some-thoughts-on-calling-convention/)

Yes, it’s me again. And it’s not yet the blog series I promised you. No, I decided I would expand some more on the whole binary compatibility / ABI thing, by exploring an even more detailed concept.

This is a brain-dump again. So expect no conclusions. This text may range from “cool stuff, everything I ever wanted to know but never had the courage to ask” to “somewhat entertaining” to “you lost me at ‘it’s me again'”. You’ve been warned 🙂

But before I go into it, let me address something from my previous blog: the lack of conclusion. The reason the previous blog has no conclusion is because it’s a brain-dump, not an essay. But more than that, it’s because I had written a large chunk of text that needed editing before publishing. And I thought, “I can do that before I go to bed” — two hours later, I hadn’t finished, had no conclusion and was way past bedtime. So I clicked “Publish”.

If you want a conclusion, you’ve got a conclusion: binary compatibility is hard. We do it for you. You don’t have to learn all of this. Unless you’re writing a library too…

Well, I mean that binary compatibility is a trial-and-error process. It’s an art we’ve fine-tuned for several years now: Qt has managed to maintain a C++ binary-compatible library for years, while developing it, extending it and improving it in many ways. The widespread use of d-pointers helps, as well as having the rules spelled out of what you can or can’t do. We also have some automatic tests (thanks to Harald) to catch those issues. And recently I evaluated a binary-compatibility checker tool for Linux against Qt.

Like I said, this is trial and error. We fix the problems when we see them. For example, in Qt 4.4.0, we accidentally broke compatibility in QByteArray with Microsoft compilers due to removing “const” from the return type in “char QByteArray::at(int) const”. We fixed it for 4.4.1, restoring compatibility (actually, we had the fix for 4.4.0, but due to a typo it wasn’t applied in the released packages; worse, I had rebuilt the packages with the fix, but I forgot to upload them to FTP). In Qt 4.5.0, we introduced another accidental break, but this time we decided we’re going to live with it. It’s the very issue that prompted me to add a new rule to the Techbase page, write the examples, read some more information and ended up writing these blogs.

Anyway, on to calling convention.

What are you talking about?

As I noted earlier, I had a much longer text for for the previous blog. It contained some of the text I’m placing here. And another reason for doing this now is because during the month of July, I spent some time trying to get QtWebKit to compile on Solaris / UltraSPARC / Sun Studio (CC 5.9), as well as AIX / POWER6 / xlC 7.0. Long story short: it compiles on both, with a large set of patches; it doesn’t link on AIX because the Out-Of-Memory killer kills the linker. But it works fine on Solaris, except for one crash, which I tried to debug.

And trying to debug, I had to learn a bit about SPARC assembly and its calling convention.

So what’s calling convention? When you make a function call in any language, the parameters you pass to the callee have to be placed somewhere that the callee can find them, in the proper order. Also, the callee may return information, so that is also defined. Moreover, there are some CPU resources that the callee must preserve, while others it may modify freely.

The calling convention is just one part of the ABI. It’s the most important part in C, next to the sizes and alignments of types. In C++, like my previous blog showed, ABIs have to go much further.

How function calls are accomplished

Let’s start with another history lesson: in the early days of computing, memory was at premium and transistor technology was still expensive. That meant processors had few registers, of limited size, and very little memory. The Intel 4004 for example had 16 4-bit registers, 3 stack levels and could address 4096 4-bit words (i.e., max 2 kB of memory).

As time progressed, things got better: we had more RAM, wider registers and the call stack limitation was lifted. This last change was quite important: I’ll get to the significance below.

By the time we got to the 16-bit Intel 8086, the basics of calling convention that we use today were set. That platform had 8 general-purpose 16-bit-wide registers, even though one had very specific use: the stack pointer (SP). The other registers had weird limitations in what ways they could be used and some instructions can work only on a specific one.

As the name says, the Stack Pointer is a pointer into a stack. That is, we have a Last-In First-Out implementation accomplished by having a pointer to the last entry. When we push something onto the stack, we move the pointer to the next available entry. When we pop something from it, we move the pointer back. (Interestingly, the stack is implemented in all architectures I know with the strategy “grow downwards” — that is, pushing something onto the stack makes the stack pointer be decreased)

The Stack Pointer was used to remove the limitation of how many calls you could nest. The CALL instruction, besides transferring control to the address specified, also pushes the return address (i.e., the instruction after the call) onto the stack. Symmetrically, the RET instruction pops an address from the stack and jumps over there.

Now, the stack is used for many things, most importantly for automatic variables in a function (those with the “auto” keyword in C or C++, which is a long-forgotten keyword). If you have a recursive function (calling itself), it’s easy to see that the ABI must find a way of placing those variables in a different place from the outer instance.

Now, in the early DOS days, like I said in my previous blog, there was only static linking. The entirety of the libraries were provided by the compiler, so the compiler could choose how to best allocate the resources at hand (registers and stack) for passing parameters. But even with that flexibility, the compiler had to be deterministic: that is, when making a function call, it needs to know what it decided in the callee function where the parameters would be.

The simplest solution that basically every single compiler does on x86 is to push the values onto the stack. Then there’s no ambiguity. Well… that’s what you’d think. First of all, do we push from left to right, or from right to left? And who cleans up: the caller or the callee? The need for variable-length function calls in C dictated that we push from right to left and the caller cleans up. But other languages (like Pascal) chose the exact opposite. In the days of DOS, you could choose your calling convention: \__cdecl (C declaration), \__pascal or \__fortran.

But even in those days where assembly tables came with the number of cycles each instruction would take to execute and you could count on it, memory access came at a cost. To read or write from memory, you had to add 1 cycle to the instruction time. So the next step in calling conventions would be to use some registers to pass data, since their access time was 0. This was never really standardised, creating a myriad of different calling conventions that go by names like “fastcall”, “stdcall”, “syscall” and differ from compiler to compiler. The reason for that is clear: there were just not enough registers.

(The near and far pointer distinction exists for an entirely different reason. To access its full 1 MB of addressable ROM and RAM, the Intel 8086 had segmented memory, with the extra 4 bits of addressing coming from one of the segment registers, which added to the 16 bits of the general-purpose registers, gave 20 bits. Programs written for simpler 16-bit systems had 16-bit pointers, but the x86 required more bits, so an option was given to the developer which pointer size he wanted)

Today

With the introduction of the Intel 80386, the x86 platform went from 16- to 32-bit. Not much changed in the register set, besides the general registers being extended to be 32-bits wide. So the calling convention on 32-bit x86 today remains largely the same as it was in 1980: arguments are passed on the stack, pushed from right to left, and the caller cleans up.

The one extra thing is to have a few registers denominated “scratch registers” and others “preserved registers”. The scratch ones, also known as “caller-save” registers, are those that the callee can use at will and does not have to preserve any values: if the caller has anything there that it wants preserved, it must save the value. The preserved ones are the opposite: the callee must save the value if it needs the register and later restore it — hence the other name “callee-save”.

This also has some incompatibilities: on some operating systems, a given set of registers is saved; in another, the set is different.

With the AMD 64 and the x86-64 architecture, the number of general-purpose registers was finally increased. From the original 8, we passed to 16, called r8, r9, r10, r11, r12, r13, r14 and r15. I just wish they also had taken the opportunity to rename the older 8 to r0-r7. With some more registers available, the operating systems running on x86-64 decided to make it the default to pass arguments on registers.

But they obviously don’t agree on which ones. I mean, at least they tried: the www.x86-64.org has an ABI specification, which is used by GCC (and, therefore, the operating systems where GCC is the default compiler). That ABI says that the first argument is passed in register RDI, the second in RSI, the third in RDX, then RCX, R8 and R9, as long as the parameters are integers and fit into the registers (if they had used numeric register numbering, that would be r7, r6, r2, r1, r8, r9; the reason for that bizarre order is that the stack pointer is r4, whereas r3 and r5 are preserved registers [rbx and rbp], and that the new registers require more bytes in the instruction to be accessed).

On Windows, the registers are used differently. And there are many other registers in the architecture, like floating-point registers, SSE registers and other vector registers.

Other architectures

SPARC

Getting to the point of why I started thinking of calling conventions, I come to the SPARC architecture. The name “SPARC” means Scalable Processor Architecture and it gets that name due to is register stack windows. The SPARC architecture has 32 general-purpose registers, that are 64-bit wide in the UltraSPARC processors. Those 32 registers are divided in 4 groups: global, incoming, local, outgoing.

The global registers are the normal kind of registers from other architectures: their values, when set, remain there. Calling functions and returning from them doesn’t change their values.

The other three groups are part of a rotating set. The processor usually has several banks of 8 registers, but at any time only 3 can be accessed by a program. There are two special instructions in the SPARC assembly that modify the set: save and restore. When you run “save”, the set rotates by 2: 1 of the 3 original banks remains (now at a different position), plus 2 new sets appear. When you run “restore”, it rotates by 2 in the inverse direction: 2 banks disappear, the third one moves and the 2 original banks of registers are restored.

This means every function shares 8 registers with its callees, 8 registers with its callers and 8 registers are not shared with anyone. Hence the distribution in outgoing, incoming and local, respectively.

It’s very simple, therefore, to decide what’s callee-save and what’s caller-save: the instruction set takes care of it (of course, you still need to decide on the fate of the global registers). The Stack Pointer is chosen as register %o6, allowing for passing parameters that don’t fit in the registers or more than we have available. By the trick of the engine, the Stack Pointer is automatically saved on function entry as register %i6, then gets restored on exit. (The saved stack pointer is usually called Frame Pointer)

The problem with the SPARC design (I mean, besides the jump slot — who thought that executing one instruction more after a call or jump gets taken is a good idea?) is that it’s wasteful. Each and every function has a fixed set of 15 64-bit scratch registers (the locals and the incoming, minus the frame pointer as it’s special-purpose). Many functions don’t need that many registers, yet they’re assigned them anyway. And the register engine doesn’t work by magic: when it runs out of register banks, the kernel must save the registers to memory. That means a deep call stack requires the kernel intervention many times.

ARM

ARM is an architecture I’m still learning. It’s a very good architecture: there are 16 32-bit-wide general-purpose registers, each instruction is exactly 32-bits wide and instructions can be postfixed with a flag to disable execution (a.k.a. predication). Then it gets slightly messier when you consider the Thumb instruction set, which is basically ARM with half of the registers and half of the instruction size.

Like any modern RISC architecture, ARM passes arguments in registers: r0, r1, r2, r3, r4, etc. The interesting thing is that it also has a register-saving engine. I don’t know exactly how it works, but on function entry, you declare which registers you’re going to use which are not part of your incoming argument list. On exit, you restore it.

An interesting thing that ARM does is that a call does not push the return address onto the stack. Instead, the return address is saved in a special register, which is accessed by the instruction to return. Since that’s a clobbered register, if a function makes calls, it has to save the return register — usually, in another register.

Itanium (IA-64)

Now this is a platform that blows completely any comparison. It’s unfortunate it got so little attention and uptake, because it has the potential to be much better than anything else I’ve ever seen.

While the x86 has 8 general-purpose registers, the x86-64 has 16, ARM has 16 and SPARC has 32, the Itanium has 128 general-purpose registers (with a few more on reserve). Of those registers, 32 are fixed (like SPARC’s global registers) and 96 are rotating. But, unlike SPARC, the register windows have variable size.

Well, not only are there 128 general-purpose registers available, there are also 128 82-bit floating point registers available, also distributed in 32 fixed / 96 rotating (though they rotate here for a different reason). Plus, there are another 128 specific-purpose registers called the “application registers” (actually, they are ar0 to ar127, but less than 128 are implemented). Plus, there are 64 1-bit predicate registers, which can be used to disable any instruction from running, allowing you, like on ARM, to execute code conditionally without having to take jumps. Plus, there are also 8 branch registers.

The weird thing about this architecture is the instruction size: 41 bits.

Yeah, 41. Actually, what it does is pack 3 instructions into 16 bytes (123 bits total) and use the extra 5 bits as instruction modifiers.

The calling convention on Itanium makes use of the rotating general-purpose registers. On function entry, the registers starting at r32 are your incoming registers. If your function takes 3 int parameters, they’ll be in r32, r33, r34. On function entry, you declare the size of the “local set” (incoming + local, like SPARC) and the size of the “outgoing set”. When you make a call, the register stack engine automatically shifts your local set out, moving the first outgoing register into r32.

Of the 32 fixed registers, there are a couple of rules on what’s scratch and what’s save. Unlike SPARC, the stack pointer is kept in a global register (r12). Then, we have r0 which is a special register: it’s both /dev/null and /dev/zero. Reads from r0 result in zero, while writes are lost (SPARC’s %g0 is actually the same). The register r1 is reserved for the “global pointer”: it’s a value set by the caller that is used by the callee to implement Position-Independent Code. r2 and r3 are scratch registers, r4 to r7 are preserved, r8 to r11 are scratch, r12 is the stack pointer, r13 is the “thread pointer”, r14 to r31 are scratch.

The number of scratch registers is very big (2+4+18) so that functions that don’t make calls can easily store temporary values in registers, without forcing the Register Stack Engine to do anything. Functions that need preserved registers can just use the local part of the rotating set (i.e., registers starting at r32 up to the first outgoing register).

Like ARM, a function call doesn’t push the return value to the stack: it’s simply saved in one of the branch registers (by convention, b0). That actually means it’s possible on Itanium to make several nested calls and not touch the stack even once — neither the normal stack nor the stack used by the Register Stack Engine when it needs to rotate and there are no free registers.

Someone asked in my previous blog why it wasn’t enough for Intel to just publish the instruction set. Well, for one thing, this is a very complex system and people needed guidance on what the best practice is. For another, with Intel publishing the Software Conventions & Runtime Architecture Guide, it practically forced everyone to agree on one calling sequence. Unlike the situation today on x86-64, on Itanium the roles of caller and callee are very well defined. (In C, at least)

The extra two documents they prepared (the psABI and the C++ ABI) were meant to further standardise across platforms. Microsoft simply ignored those two documents — the psABI is meant for System V-style operating systems anyway — but they were followed by the others. When it migrated its operating system from PA-RISC to Itanium, HP adopted the instructions of the psABI and chose ELF for the executable format and the C++ ABI.

(Everyone ignored Intel’s suggestion on where to place libraries, though)

Conclusion

Anyway, a conclusion now. Let’s see…

How does this relate to C++? Well, C++ function calls are usually just normal C calls, despite the Sun compiler complaining about anachronism. When you call a member function, an extra parameter is inserted at the first position (the this pointer) and the rest is the same.

Knowing the calling sequence is useful if you need to do low-level debugging. Alone, it won’t get you very far, but it’s a start. You’ll need to know the layout of objects in memory, for example, to be able to debug further, in case you don’t have debugging symbols handy (for example, if you’re debugging one certain Qt module that is so big that linkers run out of memory linking it with debugging symbols). That’s what I was doing on Solaris with SPARC, until I realised that the 64-bit linker could link with debugging symbols.

But it’s also useful to know how the compiler will pass your parameters when you decide how to implement a function. One example is the QLatin1String case I approached in my String Theory blog last year.

You can say that this is micro-optimisation — it may very well be. But even today, L1 cache access takes about 3 CPU cycles, while registers are free. If your function is called very often, it may make a difference.

Or, you can brag about your knowledge at office parties. (You even get prizes if you can remember the order of the x86-64 registers after 3 beers)
