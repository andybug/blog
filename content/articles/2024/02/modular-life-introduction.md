---
title: 'Modular Life: Introduction'
date: 2024-02-24
params:
  author: Andrew Fields
series:
 - modular-life
tags:
 - wasm
summary: |
    Introduces WebAssembly outside of the browser and explains some of its
    benefits as an embedded language.
---

_This article is part of the [Modular Life](/series/modular-life) series._

In this series of articles we will discover how to embed WebAssembly into a Rust project, ultimately implementing a [cellular automaton](https://en.wikipedia.org/wiki/Cellular_automaton) system.
We will build up to that point step-by-step, starting with the absolute basics.
The target audience for this series are developers with at least some understanding of basic Rust concepts but no familiarity with WebAssembly is required.

Cellular automata (the plural form) are programs with usually simple rules that generate incredible varieties of outputs, the most famous of which is [Conway's Game of Life](https://en.wikipedia.org/wiki/Conway%27s_Game_of_Life).
I think this will be an interesting topic for this series as we can architect a solution that really demonstrates a lot of the features that WebAssembly provides and also produces an interesting visual output.

{{% centerImageAndDescription src="https://blog-content.andybug.com/images%2F2024%2F02%2Fmodular-life-introduction%2Fwikipedia-gospers-glider-gun.gif" alt="An example of a complex system arising fom simple rules" title="Gosper's Glider Gun" %}}
_A "glider" from Conway's Game of Life. Source: [Wikipedia](https://en.wikipedia.org/wiki/File:Gospers_glider_gun.gif)_
{{% /centerImageAndDescription %}}

This article will give some background on WebAssembly, extending a program's functionality with plugins, and outline a rough plan for the coming articles.

## What is WASM?

[WebAssembly](https://webassembly.org/) (WASM) is an intermediate language that [many different](https://github.com/appcypher/awesome-wasm-langs) programming languages can use as a compilation target.
This intermediate format ([WASM Binary Files](https://webassembly.github.io/spec/core/binary/index.html)) can be executed by a virtual machine directly or further compiled to native machine code and then executed.
While WASM was originally intended for execution inside of a web browser, in these articles we are going to run it directly on the operating system like any other process.

When running outside the web browser, typically [WebAssembly Systems Interface](https://wasi.dev/) (WASI) is used as this provides an interface to interact with the system just like a normal process.
You can think of WASI like [POSIX](https://en.wikipedia.org/wiki/POSIX), which it partially implements, in that it provides facilities for reading and writing files, opening sockets, etc.

{{% centerImageAndDescription src="https://blog-content.andybug.com/images%2F2024%2F02%2Fmodular-life-introduction%2Fwasm-build-chain.excalidraw.png" alt="" title="WASM build chain" %}}
_The WASM build and execution chain._
{{% /centerImageAndDescription %}}

To execute WASM outside the browser, you need a runtime such as [Wasmtime](https://wasmtime.dev/).
The runtime takes your compiled WASM binary files and translates them to machine code, providing interfaces for your code to call (WASI, for example), then actually executes the code.
Embedding WASM into your own program essentially involves compiling the runtime into your project, configuring the runtime, setting up which interfaces (think functions) to provide, then having the runtime execute code.

For our cellular automaton implementation, we could define an interface for computing the state of a cell given the surrounding cells.
Then we can implement the logic in WASM modules that adhere to that interface.
The host (or even another WASM module) can simply call the data processing modules for each cell during a simulation step.
This would allow us to rapidly iterate on the logic or even swap between different rule sets by changing which modules are loaded.

{{% centerImageAndDescription src="https://blog-content.andybug.com/images%2F2024%2F02%2Fmodular-life-introduction%2Fca-wasm-architectures.excalidraw.png" alt="" title="Degree of WASM embedding" %}}
_The above diagram outlines three different levels of WASM use within a simplified cellular automaton simulation._
{{% /centerImageAndDescription %}}

## Why Use WASM for Embedding?

Embedding scripting languages into large programs has been done for a long time.
Game engines are a notable example, where the engine (usually written in something fast and low-level like C++) handles the things that require high optimization (e.g. graphics and audio) while the game logic is pushed out to scripts.
The main advantage of this approach is that the logic for a game, which is something that needs to be tweaked frequently, can be changed without recompiling the entire engine.
In addition to that, it allows end-users to change the functionality of the program to their needs (think game mods) and share those changes with other users.
[Lua](https://www.lua.org/) is a popular embedded language used by many applications for these purposes.

So why use WASM if languages like Lua already exist and are popular?

Firstly, WASM is a compiled language.
This means that it should, on average, be [significantly faster](https://programming-language-benchmarks.vercel.app/lua-vs-wasm) than a scripting language like Lua.
And since it is also an [intermediate language](https://en.wikipedia.org/wiki/Intermediate_representation), many different programming languages can target WASM.
It is quite easy to have code running in the same WASM program from multiple different source programming languages.

WASM achieves the above by being the intermediate language that translates source-language code to a common set of primitive types and memory model.
Taking this even further, the [Component Model](https://github.com/WebAssembly/component-model/blob/main/design/mvp/Explainer.md) provides a convenient typed interfacing mechanism between modules themselves and the host.
In the Component Model, each module (now a component) publishes the interfaces that it provides or consumes.
When loading the component, a linker ties together the various imports and exports from other components to ensure all of the necessary interfaces are present.
The Component Model should serve as a solid foundation for building modular applications as it standardizes all inter-module communication.

{{% centerImageAndDescription src="https://blog-content.andybug.com/images%2F2024%2F02%2Fmodular-life-introduction%2Fcomponent-model-example.excalidraw.png" alt="" title="Component Model example" %}}
_The above shows how modules import and export interfaces in the Component Model, allowing typed communication between them and also the host._
{{% /centerImageAndDescription %}}

Additionally, WASM provides a strong [sandboxing model](https://webassembly.org/docs/security/) so we can be confident that WASM code coming from external sources is safe to execute with a properly-configured runtime.
The runtime enforces stricter memory safety than a native C or C++ program, enabling it to eliminate issues like code injection.
In the WASI context, we can control all of the functions mapped in to our WASM modules, including the low-level OS functions.
For example, we can provide an implementation for Linux's `open()` function that has additional constraints, such as locking it down to a certain directory.
With a hardened runtime you can run code provided by users directly on your servers - imagine how that could change the design of your web applications.

{{% centerImageAndDescription src="https://blog-content.andybug.com/images%2F2024%2F02%2Fmodular-life-introduction%2Fexample-sandbox-open.excalidraw.png" alt="" title="Custom implementation of system functions" %}}
_The diagram above shows how the host process can provide its own implementation for low-level system functions._
{{% /centerImageAndDescription %}}

Finally, you may just want to learn WASM.
It is an exciting new technology that seems poised to play an important part in the web and perhaps on the server too.

## What's Coming Up Next

Now that we've covered why you might want to use WASM, let me outline the goals for this series of articles.

- Get some Rust code to compile into WASM and run some basic benchmarks on it
- Create a Rust host program that loads and executes WASM modules
- Introduce the Component Model to make interfacing between the modules and host easier
- Implement memory sharing between host and modules
- Securing the WASM runtime

Next up, let's compile some WASM and run some benchmarks!

