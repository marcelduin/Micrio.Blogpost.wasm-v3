# Micrio v3: The Road From JavaScript To WebAssembly

## Or: How I Learned to Stop Scripting and Love the Binary


## Table of Contents

1. **Introduction**:
What is Micrio, what is this article about, setting constraints: client side app migrating from JS to AssemblyScript//WebAssembly.

2. **Current Situation**:
Micrio 2.9, short history, techstack, browser compatibility.

3. **[WebAssembly](#3-webassembly)**

	1. **[The Discovery](#31-the-discovery)**:
	Asm.js old demos, WASM summit at Google Feb 2020

	2. **[The Rewrite (1)](#32-the-rewrite-c-and-emscripten)**:
	From JS to C++

	3. **[First Results](#33-first-results)**:
	How C++ was not the perfect choice

	4. **[The Rewrite (2)](#34-the-rewrite-assemblyscript)**:
	The initial application of AssemblyScript WASM to Micrio 2.9

	5. **The Realization**:
	The *“But can it do more for Micrio?”* process -- how it took 4 weeks to come up with a masterplan

	6. **The Rewrite (3)**:
	4 months of back to the drawing board -- back to basics with WebGL and manually created memory buffers

	7. **The Benchmark**:
	What and how to measure, what to improve

	8. **The Rewrite (4)**:
	Putting everything together in a single JS file, making it work on all browsers, reducing clutter and last minute code optimizations

	9. **The Satisfaction**:
	Finishing touches and real world results

4. **Conclusions**:
The result: pros and cons. When (not) to use WASM, best practices, thoughts on the future.

5. **Afterthoughts and the future**:
Compiling for the web, server microservices using WASM, freedom of choice of programming language, and how it will really change the landscape of technology, the fabric of our world, and might be the ultimate answer of life, the universe, and everything.




## 3. WebAssembly

WebAssembly (WASM) is the ability for your browser to run *compiled* code at (near-) native speeds. It is now recognised by the W3C as the [4th official web programming language](https://www.w3.org/2019/12/pressrelease-wasm-rec.html.en), after HTML, CSS and JavaScript.

Basically, this means you can run compiled code written in a variety of programming languages (`C/C++`, `Rust`, `Go`, `AssemblyScript`, [..](https://github.com/appcypher/awesome-wasm-langs "and many many more")) in your browser, without any need for plugins. In its purest form, you will need some JavaScript to get it running and to communicate with the browser. For instance if you want to have a graphical output such as a game, you will need to link your program to work with available renderers, such as WebGL.

This section will be the tale of my discovery of WebAssembly, and the journey to migrating the 2.9 Micrio version written in plain JavaScript to WebAssembly as much as possible.



### 3.1. The Discovery

Already back in 2013, a [demo was released](https://www.youtube.com/watch?v=BV32Cs_CMqo) by the Mozilla team running Unreal Engine 3 in the Firefox browser at 60fps, using a port of the engine made ready for the web in only 4 days of work.

For me, this was the first application I saw of [asm.js](https://en.wikipedia.org/wiki/Asm.js) -- allowing compiled code to run inside your browser at near-native speeds, using a super-optimized CPU-friendly subset of JavaScript. You could get this to work by compiling to [LLVM](https://en.wikipedia.org/wiki/LLVM)-compatible formats using for instance [emscripten](https://emscripten.org/) for `C/C++`-code. 

This was a definite game changer for the web and left me wanting to try it out myself for the longest time (*spoilers: it took 6 years*). It was also picked up brilliantly by the major browser engines, each in their own optimizing way.

Fast forward to March 2017. [WebAssembly is introduced](https://hacks.mozilla.org/2017/03/why-webassembly-is-faster-than-asm-js/) as an even more powerful way to run binary code in your browser. This joint effort by all major browsers (Firefox, Chrome, Safari and Internet Explorer) was focussed on bundling all separate efforts made so far by running compiled code inside the browser.

To illustrate the importance of its potential, for me as a simple web developer thinking browsers were always competing to be the best and most used browser, I was amazed that *all these browsers have worked together on this*, putting aside their differences.

To be honest, in 2017 I totally missed this. I was introduced to WebAssembly two years later in 2019, when WebAssembly was recognised by the W3C as the fourth official programming language for the web. As soon as I realised there would be a WebAssembly Summit at Google HQ early 2020, I really, *really* wanted to go there to see what's up. I found a colleague at Q42 to join me and in Febuary 2020 we found ourselves in Mountain View, surrounded by an awe-inspiring crowd.

That day was a real eye-opener for me on what WebAssembly can do, was already doing, and the amount of potential it still has to change how the web works. Not only for running compiled code in your browser, but **so much more** (briefly touched upon at the end of this post). You can watch the entire summit [here on YouTube](https://www.youtube.com/watch?v=IBZFJzGnBoU&list=PL6ed-L7Ni0yQ1pCKkw1g3QeN2BQxXvCPK).



### 3.2. The Rewrite: C++ and emscripten

Prior to the WebAssembly Summit, and to get to know the ecosystem, I finally followed up on my mental note from 2013 to play around with [emscripten](https://emscripten.org/). Basically you can take almost any project made in `C` or `C++`, and compile it to a binary `.wasm`-file, that your browser can natively run. 

At this point, I still didn't really have a clear image of where WebAssembly starts and stops, and how autonomously could run inside your browser, so I started a new project from scratch to see if I could make a `C++`-implementation of the basic Micrio logic: a virtual *zoomable* and *pannable* image consisting of a lot of separate tiles, using a virtual camera downloading and displaying only the tiles necessary for what you are viewing inside your screen.

This sounds like a lot of work, but the basic model comes down to simple maths; I had already created a serverside `C#` image renderer earlier; so *how hard could it be* to simply port that to `C++`?

It turns out, emscripten had already a great compatibility for [libsdl](https://www.libsdl.org/): a low-level audio, keyboard/mouse input, and OpenGL library. Which is awesome, because I could write my code using this very well documented library, already *including mouse and key* inputs and WebGL rendering. Since I was also working with downloading images, I also used the [stb_image.h](https://github.com/nothings/stb) image library.

The largest struggle of this was picking up `C++` again, never having used it for real-world projects. But after a few days of cursing and second guessing myself, I had a working first version with all of the most important features written with help of the SDL library:

* A virtual camera and all necessary viewing logic;
* Image tiles downloading;
* Image using WebGL(/OpenGL) using a simple shader;
* Mouse event handling for panning and zooming the image;
* Resize event handling to fit Micrio to the desired `<canvas>` HTML element

You can see this version running here: https://b.micr.io/_test/wasm/index.html



### 3.3. First Results

As incredibly awesome it was to see Micrio in `C++` running smoothly in my browser, and even handing all the user's input, there were a few reservations, which left me with an unsatisfied feeling.


#### 1. Coding C++ felt old-fashioned
Writing C++ felt like going back in time a little. Incredibly powerful and fully proven, but also archaic, especially as a web developer. I spent more time fiddling with making an optimized `Makefile` than I care to admit.

#### 2. The compiled `.wasm` binary was very large
As great as the help of `libsdl` and `stb_image.h` were to let me use OpenGL and JPG image functions, as much did they add to the final compiled binary file. Even with all `emcc` `Makefile` optimizations, the resulting WebAssembly binary file was `760KB`; compared to the JavaScript version of Micrio being around `240KB`, this was a major setback. These libraries packed a lot of functionalities that were not necessary for Micrio, but were still included in the compiled version.

#### 3. TIL: A `glue` file
This was the largest take-away for me. This is the part where I learnt where the limits of WebAssembly start and finish. **WebAssembly is not a magical self-contained binary that lets you run full applications out of the box**. It actually needs to be *bound* to the browser using JavaScript.

So, where I thought that all the SDL OpenGL code in `C++` would automagically be recognised by the browser: **`false`**. What `emscripten` does, is take all these OpenGL operations, and _convert them_ to WebGL operations your browser can understand.

Same with the `libsdl` mouse and keyboard-inputs: these were **glued** to the browser, using an extra JavaScript file that would set `EventListeners` for the specific cases, and send them to the WebAssembly binary. This separate JavaScript file was generated by the emscripten compiler, and had to be included in the HTML alongside the compiled binary `.wasm`-file.

Everything added together, the new total of the *base engine* of Micrio was a whopping **`791KB`**; a bit too much for my likes.

However, it's extremely neat that there is a `C++`-port of Micrio that would run natively on Linux and MacOS-systems. And I learned a lot from this.


### 3.4. The Rewrite: AssemblyScript

Fast forward a few months, to just after the WebAssembly Summit in Mountain View in Febuary 2020. With a bundle of fresh energy and inspiration, I decided to see if I could use WebAssembly to improve the Micrio JavaScript client. I knew at this point that WebAssembly is really powerful for CPU-intense jobs. Micrio is doing quite some computations like deciding which image tiles are inside your screen, so this would be a good starting point.

