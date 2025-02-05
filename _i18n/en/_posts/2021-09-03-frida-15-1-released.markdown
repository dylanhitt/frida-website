---
layout: news_item
title: 'Frida 15.1 Released'
date: 2021-09-03 12:00:00 +0200
author: hot3eed
version: 15.1
categories: [release]
---

Introducing the _brand new_ Swift bridge! Now that Swift has been
ABI-stable since version 5, this long-awaited bridge allows Frida to play nicely
with binaries written in Swift. Whether you [consider][] Swift a static or a
dynamic language, one thing is for sure, it just got a lot more dynamic with
this Frida release.

## Metadata

Probably the first thing a reverser does when they start reversing a binary is
getting to know the different data structures that the binary defines. So it
made most sense to start by building the Swift equivalent of the `ObjC.classes`
and `ObjC.protocols` APIs. But since Swift has other first-class types,
i.e. structs and enums, and since the Swift runtime doesn't offer reflection
primitives, at least not in the sense that Objective-C does, it meant we had to
dig a little deeper.

Luckily for us, the Swift compiler emits metadata for each type
defined by the binary. This metadata is bundled in a
`TargetTypeContextDescriptor` C++ struct, defined in
[include/swift/ABI/Metadata.h][] at the time of writing. This data structure
includes the type name, its fields, its methods (if applicable,) and other useful
data depending on the type at hand. These data structures are pointed to by
relative pointers (defined in [include/swift/Basic/RelativePointer.h][].) In
Mach-O binaries, these are stored in the `__swift5_types` section.

So to dump types, Frida basically iterates over these data structures and
parses them along the way, very similar to what [dsdump][] does, except that you
don't have to build the Swift compiler in order to tinker with it.

Frida also has the advantage of being able to probe into
internal Apple dylibs written in Swift, and that's because we don't need to
parse the `dyld_shared_cache` thanks to the private `getsectiondata` API, which
gives us section offsets hassle-free.

Once we have the metadata, we're able to easily create JavaScript wrappers for
object instances and values of different types.

## Conventions

To be on par with the Objective-C bridge, the Swift bridge has to support
calling Swift functions, which also proved to be not as straight forward.

Swift defines its own calling convention, `swiftcall`, which, to put it
succinctly, tries to be as efficient as possible. That means, not wasting load
and store instructions on structs that are smaller than 4 registers-worth of
size. That is, to pass those kinds of structs directly in registers. And since
that could quickly over-book our precious 8 argument registers
(on AARCH64 `x0`-`x7`), it doesn't use the first register for the `self`
argument. It also defines an `error` register where callees can store errors
which they throw.

What we just described above is termed "physical lowering" in the Swift compiler
docs, and it's implemented by the back-end, LLVM.

The process that precedes physical lowering is termed "semantic lowering," which
is the compiler front-end figuring out who "owns" a value and whether
it's direct or indirect. Some structs, even though they might be smaller than
4 registers, have to be passed indirectly, because, for example, they are
generic and thus their exact memory layout is not known at compile time, or
because they include a weak reference that has to be in-memory at all times.

We had to implement both semantic and physical lowering in order to be able
to call Swift functions. Physical lowering is implemented using JIT-compiled
adapter functions (thanks to the `Arm64Writer` API) that does the necessary
`SystemV`-`swiftcall` translation. Semantic lowering is implemented by utilizing
the type's metadata to figure out whether we should pass a value directly or
not.

The compiler [docs][] are a great resource to learn more about the calling
convention.

## Interception

Because Swift passes structs directly in registers, there isn't a 1-to-1 mapping
between registers and actual arguments.

And now that we have JavaScript wrappers for types, and are able to call Swift
functions from the JS runtime, a good next step would be extending `Interceptor`
to support Swift functions.

For functions that are not stripped, we use a simple regex to parse argument
types and names, same for return values. After parsing them we retrieve the
type metadata, figure out the type's layout, then simply construct JS wrappers
for each argument, which we pass the Swift argument value, however many
registers it occupies.

## EOF

Note that the bridge is still very early in development, and so:
  - Currently supports Darwin arm64(e) only.
  - Performance is not yet in tip-top shape, some edge case might not be handled
    properly and some bugs are to be expected.
  - There's a chance the API might change in breaking ways in the
    short-to-medium term.
  - PRs and issues are very welcome!


Refer to the [documentation][] for an up-to-date resource on the current API.

Enjoy!


### Changes in 15.1.0

- Implement the Swift bridge, which allows Frida to:
  - Explore Swift modules along with types implemented in them, i.e. classes,
    structs, enums and protocols.
  - Create JavaScript wrappers for object instances and values.
  - Invoke functions that use the `swiftcall` calling convention from the
    JavaScript runtime.
  - Intercept Swift functions and automatically parse their arguments and return
    values.
- Fix i/macOS regression where changes related to iOS 15 support ended up
  breaking support for attaching to Apple system daemons.
- gadget: Add interaction.parameters in connect mode. These parameters are then
  "reflected" into app's info under `parameters.config`. Thanks [@mrmacete][]!

### Changes in 15.1.1

- gumjs: Fix Swift lifetime logic in the V8 runtime.

### Changes in 15.1.2

- control-service: Fix signal wiring, so signals such as Device.output get
  emitted correctly by a remote frida-server. Thanks [@mrmacete][]!
- gadget: Fix the “runtime” option, which was forgotten during the refactorings
  leading up to Frida 15.
- relocator: Optimize handling of x86 RIP relative code, by simply adjusting the
  offset where possible.
- gumjs: Add ESM support, so tools like frida-compile can output better code.
- gumjs: Throw away source code after parsing it.
- gumjs: Plug leak when compiling to QuickJS bytecode.
- java: Expose JNIEnv->GetDirectBufferAddress. Thanks [@pandasauce][]!

### Changes in 15.1.3

- objc: Fix the Proxy respondsToSelector implementation. Thanks [@hot3eed][]!
- gumjs: Fix V8 runtime crash when module is missing.
- gumjs: Emit V8 exceptions thrown by ESM entrypoints.

### Changes in 15.1.4

- gumjs: Fix double-free related to weak ref callbacks in the QuickJS runtime.
  Thanks [@mrmacete][]!
- gumjs: Ignore Interceptor context in unrelated NativeCallback invokes. In this
  way invalid Interceptor contexts from higher up the call stack will be safely
  ignored in favor of the minimal but correct callback context. Thanks
  [@mrmacete][]!
- gumjs: Fix ESM module name normalization logic.
- gumjs: Add Process getters for cwd, home, and tmp dirs.
- swift: Improve metadata caching performance by ~3x. Thanks [@hot3eed][]!
- node: Publish Electron prebuilds for v15 instead of v13.

### Changes in 15.1.5

- gumjs: Fix crash when QuickJS stringifies large numbers. Thanks
  [@vfsfitvnm][]!
- gumjs: Expose GError and GIConv to CModule. Thanks [@0xDC00][]!
- droidy: Support ADB server host/port environment variables. Thanks
  [@amirpeles90][]!
- swift: Improve load behavior on non-Darwin.

### Changes in 15.1.6

- swift: Fix crash during load on non-Darwin. Thanks [@hot3eed][]!

### Changes in 15.1.7

- swift: Fix CoreSymbolication integration on older OS versions. Thanks
  [@hot3eed][]!
- python: Add setup.py download logic for Android/ARM. Our CI doesn't yet build
  and upload such a binary, though.

### Changes in 15.1.8

- darwin: Add support for working in the SRD environment. Thanks [@Nessphoro][]!
- darwin: Add support for building with newer iOS SDKs.

### Changes in 15.1.9

- x86-relocator: Fix patching of RIP relative instructions. This was a
  regression introduced in 15.1.2, resulting in Stalker becoming unreliable.
- portal-service: Always remove ClusterNode sessions whenever a session ID is
  unset from PortalService. This avoids both NULL dereferences and leaks.
  Thanks [@mrmacete][]!
- frida-portal: Fix typo in --help output.

### Changes in 15.1.10

- p2p: Gracefully handle unsupported ICE candidates.
- p2p: Disable ICE-TCP for now.
- p2p: Rework PeerSocket to fix synchronization issues.
- socket: Fix WebService teardown.
- x86-writer: Add UD2 instructions after/in indirect branches.
- x86-writer: Emit larger NOPs when possible.
- stalker: Improve performance of the x86/64 backend.
- objc-api-resolver: Guard against disposed ObjC classes. Thanks [@mrmacete][]!
- gumjs: Fix V8 Interceptor.{replace,revert}() regression. Thanks [@3vilWind][]!
- gumjs: Expand Instruction API w/ regsAccessed and operand.access. Thanks
  [@3vilWind][]!

### Changes in 15.1.11

- x86-writer: Add put_sahf() and put_lahf().
- x86-relocator: Fix handling of out-of-range Jcc branch targets. Thanks
  [@0xDC00][]!
- stalker: Optimize target address retrieval.
- stalker: Avoid expensive XCHG instructions.
- stalker: Optimize IC prolog to use SAHF/LAHF.
- memory: Improve scan() to support regex patterns. Thanks [@hot3eed][]!
- kernel: Get base from all_image_info where supported. Thanks [@mrmacete][]!
- windows: Improve dbghelp backtracer reliability. Thanks [@HonicRoku][]!

### Changes in 15.1.12

- agent: Fix hang on unload w/ NO_REPLY_EXPECTED calls, where we would wait for
  a reply to be sent. Due to the server-side DBus code being generated by the
  Vala compiler, and it previously (in Frida <= 15.1.9) ignoring
  NO_REPLY_EXPECTED, this bug went unnoticed.
- android: Move to NDK r24 Beta 1.

### Changes in 15.1.13

- linux: Improve module resolution on glibc systems.
- fruity: Fix spawn on dyld v4 case (jailed iOS 15.x). Thanks [@mrmacete][]!
- objc-api-resolver: Protect against objc_disposeClassPair() via mutex. Thanks
  [@mrmacete][]!
- gumjs: Update Kernel.scan\*() to match Memory.scan\*(). Thanks [@hot3eed][]!
- stalker: Fix emitted branch op-code on x86.
- stalker: Fix handling of Thumb IT AL.
- stalker: Handle excluded Linux calls through PLT on x86.
- stalker: Fix Linux exception handling on x86.
- java: Add Java.backtrace(), without any API stability guarantees for now.

### Changes in 15.1.14

- backtracer: Improve fuzzy backtracers to also include the immediate caller,
  and avoid walking past stack end when known.
- windows: Implement Thread.try_get_ranges().
- linux: Implement Thread.try_get_ranges().
- ios: Check in with launchd on SRD systems.
- stalker: Fix accidental clobbery in call depth code on x86.
- node: Publish Node.js prebuilds for v17, too.


[consider]: https://youtu.be/0rHG_Pa86oA?t=36
[include/swift/ABI/Metadata.h]: https://github.com/apple/swift/blob/52e852a7a9758e6edcb872761ab997b552eec565/include/swift/ABI/Metadata.h
[dsdump]: https://github.com/DerekSelander/dsdump
[include/swift/Basic/RelativePointer.h]: https://github.com/apple/swift/blob/52e852a7a9758e6edcb872761ab997b552eec565/include/swift/Basic/RelativePointer.h
[docs]: https://github.com/apple/swift/blob/52e852a7a9758e6edcb872761ab997b552eec565/docs/ABI/CallingConvention.rst
[documentation]: https://github.com/frida/frida-swift-bridge/blob/master/docs/api.md
[@mrmacete]: https://twitter.com/bezjaje
[@pandasauce]: https://github.com/pandasauce
[@hot3eed]: https://github.com/hot3eed
[@vfsfitvnm]: https://github.com/vfsfitvnm
[@0xDC00]: https://github.com/0xDC00
[@amirpeles90]: https://github.com/amirpeles90
[@Nessphoro]: https://github.com/Nessphoro
[@3vilWind]: https://github.com/3vilWind
[@HonicRoku]: https://github.com/HonicRoku
