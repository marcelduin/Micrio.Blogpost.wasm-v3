# Going From JavaScript To WebAssembly In Three Steps

### My journey of taking the Micrio client to the next level

## Introduction

Hi! I'm Marcel, web developer at [Q42](https://www.q42.com/) and creator of the [Micrio storytelling platform](https://micr.io).

In 2015, I started developing a JavaScript viewer for ultra high resolution 2D and 360&deg; images with added markers, tours, audio, and more. Since then, I've been pushing to find the best balance between hardware performance, minimal CPU and bandwidth use, and compatibility for older browsers to deliver a sharp and high quality viewing experience.

For Micrio, it is vital that the performance on the client's browser is as good as possible. The reason for this is very simple: when you are being told a story, or watching a movie, even *a single frameskip* immediately takes you out of your experience.

Since Micrio is being used for an [ever growing list](https://micr.io/showcases) of awesome projects, the most important thing is that for whoever visits a Micrio project, it must work, and *work well*: delivering fast load times, and a butter smooth interactive experience.

[WebAssembly](https://webassembly.org/) (Wasm) is the ability for your browser to run *compiled* code at (near-) native speeds. It is now recognised by the W3C as the [4th official web programming language](https://www.w3.org/2019/12/pressrelease-wasm-rec.html.en), after HTML, CSS and JavaScript.

With it, you can run compiled code written in a variety of programming languages (C/C++, Rust, Go, AssemblyScript, [and many more](https://github.com/appcypher/awesome-wasm-langs)) in your browser, without any need for plugins.

Finding out about Wasm in late 2019 really made me want to try it out. Could I use this tech to make the Micrio client run smoother than the current version does? Would I need to rewrite everything in C++, and if so, how would that work? Would the effort be worth the gains it would give me? And not in the least.. *how does it work!?*

This article describes my journey from upgrading the Micrio **JavaScript-only client to use WebAssembly**, with the hopes of improving performance, and taking my code to the next level.



### The current version

To give a rough idea about the tech stack of the current latest JS-only revision of Micrio ([version 2.9](https://b.micr.io/micrio-2.9.min.js)): This library as a single JS file works on all semi-modern browsers, including even Internet Explorer 10 for 2D, and IE 11 for 360&deg; images.

It uses Canvas2D for the rendering of 2D images, and [three.js](https://threejs.org/)/WebGL rendering for 360&deg; images. Written in [ES6 JavaScript](https://caniuse.com/#search=es6) (or ECMAScript 6, the latest version of JavaScript), it still [compiles](https://developers.google.com/closure/compiler) to ES5 to support Internet Explorer 11.

As you can imagine, displaying a [231.250 x 193.750 pixel image](https://micr.io/i/xCSYV/) in your browser in a matter of milliseconds, allowing the user to freely zoom in and navigate, requires a little bit of processing power.

Now, Micrio 2.9 *isn't bad*. It runs pretty smoothly on all devices. But with WebAssembly around the corner, allowing all calculations to be done at native CPU speeds, this could potentially make a big difference in making Micrio's performance even better, and could improve the code architecture a lot.

And, perhaps, this could also mark the setup for a new major version, where I will draw a clear line and drop all compatibility and polyfills for older browsers: **Micrio 3.0**.




## First Rewrite: C++ and emscripten

As a first step into the world of WebAssembly, getting to know the ecosystem, I started to play around with [emscripten](https://emscripten.org/). With it, you can take almost any project made in C or C++, and compile it to a binary `.wasm` file that your browser can natively run. 

At this point, I didn't really have a clear image of where WebAssembly starts and ends, and how autonomously it could run inside your browser. So I started a new project from scratch to see if I could make a C++-implementation of the basic Micrio logic: a virtual *zoomable* and *pannable* image consisting of a lot of separate tiles, using a virtual camera for displaying only the tiles necessary for what the user is viewing inside your screen.

It turns out, emscripten already had great compatibility for [libsdl](https://www.libsdl.org/): a low-level audio, keyboard/mouse input, and OpenGL library. Which is awesome, because I could write my code using this very well documented library, even including mouse and key inputs and WebGL rendering. Since I was also working with downloading images, I also used the [stb_image.h](https://github.com/nothings/stb) image library.

![Setting up SDL and OpenGL](https://raw.githubusercontent.com/marcelduin/Micrio.Blogpost.wasm-v3/master/img/cpp.png "C++ in the 21st century")


The largest struggle of this was picking up C++ again, never having used it outside of hobby scope many years ago. But after a few days of cursing and second guessing myself, I had a working first version with all of the most important features written with help of the SDL library:

* A virtual camera and all necessary viewing logic;
* Image tiles downloading;
* Rendering using WebGL(/OpenGL) using a simple shader;
* Mouse event handling for panning and zooming the image;
* Resize event handling to fit Micrio to the desired `<canvas>` HTML element

You can see this version running here: https://b.micr.io/_test/wasm/index.html :

[![Micrio in native C++](https://raw.githubusercontent.com/marcelduin/Micrio.Blogpost.wasm-v3/master/img/emscripten.png)](https://b.micr.io/_test/wasm/index.html)


### First Results

As incredibly awesome it was to see Micrio in C++ running smoothly in my browser, and even handing all the user's input, there were a few reservations, which left me with an unsatisfied feeling.


#### 1. Coding C++ felt old-fashioned
Writing C++ felt like going back in time. Incredibly powerful and fully proven, but also archaic, especially for me as a web developer. I spent more time fiddling with making an optimized `Makefile` than I care to admit.

![The emscripten C++ Makefile](https://raw.githubusercontent.com/marcelduin/Micrio.Blogpost.wasm-v3/master/img/makefile.png "( ͡° ͜ʖ ͡°)")

#### 2. The compiled `.wasm` binary was very large
As great as the help of `libsdl` and `stb_image.h` were to let me use OpenGL and JPG image functions, as much did they add to the final compiled binary file. Even with all `emcc` compiler optimizations (which can even use the awesome `closure` JS compiler), the resulting WebAssembly binary file was 760KB. Compared to the JavaScript version of Micrio being around 240KB, this was a major setback. These libraries packed a lot of functionalities that were not necessary for Micrio, but were still included in the compiled version.

#### 3. TIL: A *glue* file
This is the part where I learnt where the limits of WebAssembly start and finish. **WebAssembly is not a magical self-contained binary that lets you run full applications out of the box**. It actually needs to be *bound* to the browser using JavaScript.

![glue clipart](https://raw.githubusercontent.com/marcelduin/Micrio.Blogpost.wasm-v3/master/img/glued.png)
[Image source](http://www.clker.com/clipart-13445.html)

Where I thought that all the SDL OpenGL code in C++ would automagically be recognised by the browser: *wrong*. What `emscripten` does, is to take all OpenGL operations from C++, and _convert them_ to WebGL operations your browser can understand.

Same with the `libsdl` mouse and keyboard-inputs: these were **glued** to the browser using an extra JavaScript file that would set event listeners for the specific cases, and send them to the WebAssembly binary. This separate JavaScript file was generated by the emscripten compiler, and had to be included in the HTML alongside the compiled binary `.wasm` file.

Everything added together, the new total of the *base engine* of Micrio was a whopping **791KB**; a bit too much for my liking.



## Second Rewrite: AssemblyScript

Fast forward a few months, to just after having attended the awesome [WebAssembly Summit](https://webassembly-summit.org/) in Mountain View in February 2020. With a bundle of fresh energy and inspiration, I decided to see if I could use WebAssembly to improve the Micrio JavaScript client a second time.

During the WebAssembly conference, I was very impressed by a [synth demo](https://www.youtube.com/watch?v=C8j_ieOm4vE) written in **[AssemblyScript](https://www.assemblyscript.org/)**, a language created specifically for WebAssembly, using the TypeScript syntax. Basically you can write (near) TypeScript, which compiles to a `.wasm`-binary. So anyone familiar with either TypeScript or JavaScript ES6 will not have a lot of difficulties using it.

And the great thing-- it's all installed using `npm`, so [getting it up and running](https://www.assemblyscript.org/quick-start.html) and compiling your program is super easy!

There are a few basic [`types` added in AssemblyScript](https://www.assemblyscript.org/types.html), which are required for compile-time optimizations:

* `f64` / `f32` : For 64 or 32-bit floats;
* `i8` / `i16` / `i32` / `i64` : For signed `int`s, ranging in precision
* `u8` .. `u64` : For unsigned `int`s
* [And a few more](https://www.assemblyscript.org/types.html)


### Going atomic

This time, I wanted to see if it was possible to only let a small part of Micrio run inside WebAssembly, and still use most of the JavaScript that was already inside the client. *How small can we get it?* I decided to focus on a subset of camera functions, such as translating screen coordinates to image coordinates and vice versa. So this time no rendering, event handling, or writing shaders.

![Simple camera functions in AssemblyScript](https://raw.githubusercontent.com/marcelduin/Micrio.Blogpost.wasm-v3/master/img/assemblyscript.png "My First AssemblyScript")
*Simple camera functions in AssemblyScript*

The result: a 3KB binary containing some basic math functions, that take an input and return an output. AssemblyScript offers some *glue-tooling* by providing its own [Loader](https://www.assemblyscript.org/loader.html), which will deal with importing the binary file and being able to call these functions.

However, this is optional and I ended up using the JavaScript [WebAssembly API](https://developer.mozilla.org/en-US/docs/WebAssembly/Using_the_JavaScript_API), *neat*. And it turns out, this is super easy: simply use the `fetch` API to load your compiled `.wasm` file, cast it as an `ArrayBuffer`, and use the `WebAssembly.instantiate()` function to get it up and running.

![Loading a wasm file](https://raw.githubusercontent.com/marcelduin/Micrio.Blogpost.wasm-v3/master/img/instantiate.png "Gluing it yourself")

The compiled binary will then offer an `exports` object, containing the functions that you have exported in the AssemblyScript file, which you can immediately call from JavaScript as if they were normal functions.

Wait.. "*which you can immediately call from JavaScript as if they were normal functions*"...

**WebAssembly is running synchronously to JavaScript!** :exploding_head:

Having worked with WebWorkers before, I honestly thought that WebAssembly would run inside its own CPU thread, and that any function calls would be `async`. Nope, the Wasm-functions you call will return immediately!

[*This is, like, powerful stuff*!](https://www.assemblyscript.org/exports-and-imports.html#exports)

### Bundling the compiled Wasm inside the JS file

Since I now had some extra performing hands on deck for Micrio that was very easy to integrate, I decided to include this minimal WebAssembly binary in the then-stable release of Micrio (2.9).

However, I didn't want an extra HTTP request for the Wasm binary every time someone loaded the Micrio JS. So I included a `base64` encoded version of the Wasm-file *inside* the Micrio JS, and for browsers that support it, auto-loaded that. As a fallback, I still had the original JS-only functions in place.

![Wasm as base64 embedded in JS](https://raw.githubusercontent.com/marcelduin/Micrio.Blogpost.wasm-v3/master/img/b64.png "Somehow this feels like cheating")
*Somehow this feels like cheating*

This approach worked wonderfully. Zero weird bugs and errors, and (marginal) better performance. The Micrio 2.9-release has been running Wasm for a while already!


### The Realization

Okay, so, mission succeeded! The simplest math functions inside the Micrio JS were now handled by WebAssembly. But all schizoid rendering logic (Canvas2D for flat images and three.js/WebGL for 360&deg; images) was still 100% JavaScript.

I felt like I could get more out of this, but I didn't immediately know *how*. In the next 2 weeks, going to bed and waking up with the question, I came up with the idea:

**What if I moved the entire rendering logic to AssemblyScript, using WebGL for the graphical output?**

So kind of like the C++ emscripten implementation, but this time using the lean WebAssembly approach: only replacing parts of my JavaScript, maintaining Micrio's own event handlers and module (markers, tours, audio) logic.



## Third Rewrite: AssemblyScript + WebGL

*Third time's a charm.*

This iteration, I used what I'd learned in my previous two iterations:

1. **It's possible to have the entire rendering logic inside WebAssembly**
2. **It's possible to combine JS and WebAssembly fluently**

Back to the drawing board. What I was going to make was a single WebGL renderer, used for both the 2D and 360&deg; Micrio images, so I could do away with the Canvas2D and three.js implementations, not only (hopefully) improving performance, but being able to throw away *a lot* of JS code and do away with the three.js dependency.

### Connecting WebAssembly's memory to WebGL

WebAssembly runs within its own sandbox, using its own requested amount of memory. For instance, a WebAssembly program can request 16MB of memory to work with. This memory is assigned by the browser, and can be fully used by your WebAssembly program.

As the developer, you are **100%** in control of this memory. Micrio is an excellent case where the amount of memory needed to handle the zoomable image logic can be precalculated. So it's entirely possible to have a WebAssembly program producing *zero* garbage memory, ie. runs without any external optimizations.


![180 bytes of memory for a single 2D tile](https://raw.githubusercontent.com/marcelduin/Micrio.Blogpost.wasm-v3/master/img/singletile.png "A single 2D tile takes 180 bytes of memory")


The cool thing is: this memory buffer is fully available from JavaScript as an `ArrayBuffer` object.

What is WebGL under the hood? To put it oversimplified: it takes a bunch of coordinate `Float32Arrays` and textures, and draws them on your screen using [shaders](https://en.wikipedia.org/wiki/Shader). These coordinate arrays would be single abstract representations of the zoomable images with its individual tiles.

So... if WebAssembly can create an array of vertices in 3D space, and JavaScript can have a *casted view* of those as a `Float32Array` (not cloned, simply a pointer to the shared memory space), these can be passed directly to WebGL for its geometry and texture coordinate buffers!

That means that the output of WebAssembly is **directly connected** to WebGL's input by JavaScript *just once*, at initialization.


![JavaScript directly connecting WebAssembly output to WebGL input](https://raw.githubusercontent.com/marcelduin/Micrio.Blogpost.wasm-v3/master/img/connecting.png "Leaving the hard computational work to the parts made for it")
[Image source](http://www.clker.com/clipart-14374.html)



### Moving the image tile logic to AssemblyScript

*Doing the hard work*

Now the basic setup was known, this is where it became more difficult. I had to *actually* take all the JS code for rendering 2D images using Canvas2D and 360&deg; images using three.js/WebGL (let's call this the *twin engine*), and rewrite it in AssemblyScript, using WebGL as graphical output.

This required a few steps, which I will not fully document here since it's out of scope (*next blogpost: WebGL?*). But the most important steps are laid out below.

![A schema I found googling "WebGL pipeline"](https://raw.githubusercontent.com/marcelduin/Micrio.Blogpost.wasm-v3/master/img/pipeline.svg)*The German WebGL rendering pipeline -- you're welcome! ([wiki](https://commons.wikimedia.org/wiki/File:Graphics_pipeline_de.svg))*

The input for AssemblyScript are only the Micrio image parameters: a unique ID, and the image width and height. The output must be WebGL-ready *vertex* and *texture coordinate* array buffers, containing all coordinates of all tiles and their individual texture mappings.

WebGL in its raw form gives you only low level functions to work with. Where drawing a tile in Canvas2D was simply using `context.drawImage(...)` with some relative coordinates, now all tile coordinates should be stored in a single vertex array buffer having static positions, which WebGL will draw relative to the virtual camera's 3D Matrix and the user's browser viewport.

It was my job now to create those array buffers from scratch. Since Micrio supports both 2D and 360&deg; images, and their technique is quite different, I had to create different array creation methods for both separately.

After a while, and a lot of cursing for not paying better attention in school because of the maths involved, I got it right: AssemblyScript's output were a few `ArrayBuffers` containing all tile and texture information, that could be directly linked to WebGL.

![A WebAssembly vertex buffer as Float32Array](https://raw.githubusercontent.com/marcelduin/Micrio.Blogpost.wasm-v3/master/img/vertexbuffer.png "The Float32Array vertex buffer generated by WebAssembly in the JS console")
*The Float32Array vertex buffer generated by WebAssembly in the JS console*


### Connecting it to JavaScript

*`rm -rf js/camera`*

Since I was replacing modules inside the Micrio JavaScript client instead of working from scratch, I had to first remove all old JS-based rendering code (the *twin engine*), and then make the *glue* discussed earlier, linking the WebAssembly program as transparently as possible to the existing internal Micrio JS APIs.

Not only was this a fun thing to do, it was also a great sanity check of the entire Micrio JS architecture, seeing if there was rendering logic in places where it wasn't supposed to be.

After removing all last tidbits and placing the code full of `// TODO: FIX ME FOR WASM` comments, it was time to implement the newly created `Micrio.Wasm` JavaScript module, which exposed all of the previous render functions to the rest of the client, this time handled by WebAssembly.

This module acts as a 2-way street between JS and Wasm and takes care of a few things:

* Loading the Wasm binary and setting up the shared memory buffer;
* Acting as a transparent hub between JS modules and Wasm;
* Sending all required user events (mouse, key, touch) to Wasm;
* Downloading the requested tile images and linking them to WebGL as textures;
* Controlling the main rendering loop for both Wasm and JS (for correctly placing/animating HTML elements like image markers).

Bit by bit, over the course of a few weeks, this engine was made as a perfect fit to work together with the rest of the JS client, saving some hardly used and exotic implementations (but still used by 1% of the Micrio projects) for last.


### Rendering the lot

This is what's so cool about WebGL: you can tell it to render certain *parts* of your pre-generated geometry buffer. All you need is to know the individual tiles' buffer start index, and the number of coordinates the tile uses in 3d space, and those are the only parameters to pass to WebGL to draw this tile (alongside the correct texture reference-- disregarded here).

The functions to decide what those tiles are, are quite different between 2D and 360&deg; images, the latter using a lot of 3D Matrix calculations, of which I will spare you the details here.

So now we are at the point where everything is in place to draw a frame in Micrio. To do this, AssemblyScript needs to know:

* The dimensions of your browser window, which it gets from JavaScript
* How the virtual camera is positioned and zoomed in, based on all prior user input received from JS

And then returns a bunch of `start` and `length` indices to WebGL, to draw the tiles:

![The WebGL tile drawing function](https://raw.githubusercontent.com/marcelduin/Micrio.Blogpost.wasm-v3/master/img/drawtile.png "The literal tile drawing function")

This JavaScript function, the actual WebGL rendering function, is called from *inside* AssemblyScript. Yes, [it is totally possible to call JavaScript-functions from inside your running WebAssembly functions](https://www.assemblyscript.org/exports-and-imports.html#imports)!

This makes JavaScript a mere **puppet** of WebAssembly. Which is friggin' awesome.

After all said and done, and not quite as straightforward as described here (360&deg; only came after the 2D renderer was finished), I was left with a Micrio client that was good enough to start testing and benchmarking with!



## Putting it to the test

So, after the entire ordeal of the previous chapter, we are now left with a *testable* Micrio JS/Wasm/WebGL client. Not yet ready for production, but slowly going from first steps to the full performance potential.

The new client already *felt* a lot smoother in my browser. Zooming, panning and animating clearly went more smoothly than the previous JS-only version.

This was also not a very big surprise, just looking at the difference in uncompiled code of both versions:

* Micrio 2.9 has a total of **444KB** JavaScript code
* Micrio 3.0 now has 315KB JavaScript and 56KB AssemblyScript, totalling **371KB** of code

That means that I had **73KB** less Micrio-code after the migration, for exactly the same behavior! Where did that all go!?

Not to mention that for 360&deg; images, I could now do away with the three.js dependency, which was an extra **607KB** of JavaScript.

![1051KB of 2.9 vs 371KB of 3.0](https://raw.githubusercontent.com/marcelduin/Micrio.Blogpost.wasm-v3/master/img/comparison-uncompiled.svg)

*I still feel smug about this.*

In the next chapter ([Going to production](#8-going-to-production)) I will write more about making the new library as compact as possible.

But first, I want to *prove* that the last few months were not spent in vain, and actually have some measurable results of the differences between Micrio 2.9 and 3.0.


### Benchmark till you drop

*Disclaimer*: Software testing and benchmarking is an art form to which there is no end of detail. Where a professional tester would take an actual Windows 3.1 PC to see how Micrio would perform on there, I have been a little more lazy in my approach.

As I mostly wanted to test the *difference* between the performance of Micrio 2.9 and 3.0, I decided that a **single-device** (my medium-end Windows laptop), **single-browser** (Chrome) benchmark run would be sufficient.

The Chrome devtools have great benchmarking tools, which I used. To be sure that there was as little background noise as possible, I ran all tests after a fresh reboot, making sure no unnecessary background processes, such as Windows Update, were running.

The test itself was taking [this hi-res Micrio image](https://micr.io/i/dzzLm/the-fight-between-carnival-and-lent-pieter-bruegel-the-elder "The Fight Between Carnival and Lent by Pieter Bruegel the Elder"), removing all HTML-related rendering modules such as the markers, so we're left with just the image renderer, and running the **Benchmark** video tour, which is a 2-minute animated camera path through the image.

All tests were run with the browser in fullscreen mode, on a 1440p screen (2560 x 1440px).

**If you want to try it yourself, try the tour in both versions: [Micrio 3.0](https://micr.io/i/dzzLm/) vs [Micrio 2.9](https://micr.io/i/dzzLm/the-fight-between-carnival-and-lent-pieter-bruegel-the-elder?v=2.9)** (open *Video Tours* from the menu and select *Benchmark*).


### First results and subsequent runs

First of all, the benchmarks were *not only* testing the JavaScript vs WebAssembly performance. A lot more had changed under the hood, for instance going from Canvas2D rendering to WebGL. These tests are by no means good comparisons for barebone JS versus Wasm, but rather the performance of Micrio as a whole.

After the first few trial runs, while the test *looked* much smoother on my screen using Micrio 3.0, the measured results were not that impressive. Over the 2 minute measured timespan, there was only *14% less* CPU usage than with 2.9, tested over a number of trials.

![First benchmarks were underwhelming](https://raw.githubusercontent.com/marcelduin/Micrio.Blogpost.wasm-v3/master/img/bench-1.png "Meh.")
*Meh.*

There seemed to be overall less scripting, but *way* more rendering and painting going on. Also, the red dots in the timeline at the top indicate blocked frames, or frame skips. There were actually *more* than before now.

After a long sigh, this result yielded a week's worth of code optimizations, tweaking, and small rewrites, to see if I could get more out of it than this.



### Optimizing

As always, the best way to optimize things is to examine the benchmarking test results:

1. *Which step is taking too much time, and how can we minimize that?*
2. Backtracking that code until it worked better: turning off individual functions until it barely works, until the performance is acceptable;
3. Finding the next thing to optimize, and `GOTO 1` *ad nauseam*.

I kept measuring every step to gain a better understanding of what's *actually* happening inside the browser, which also teaches some great insights. For instance, how a single overlayed HTML element can make a big difference in the `Painting` phase.

This is by far the best way to optimize code and it really helped reduce the rendering times. A few other things that helped a lot:


#### WebWorkers
I used WebWorkers to offload the downloading of the tile textures. While a good idea on paper, it actually had a big performance impact, and simply using the main JS thread for this helped a lot. Lesson learned: the browser is already *heavily* optimized by itself;


#### `requestAnimationFrame`
Using [`requestAnimationFrame`](https://developer.mozilla.org/en-US/docs/Web/API/window/requestAnimationFrame) for drawing at 60fps, as I only want to draw a next frame if it's necessary: when not all tiles are downloaded yet, or there is an animation running.

Before, I would first draw the image on the screen, and based on that, request the next frame or not. This caused a lot of janking and frame skips: because the drawing itself took a few milliseconds, it was sometimes already too late to still request the *next* frame!

So in WebAssembly, I made a `.shouldRequest()` function, which did all the pre-calculations required for the rendering, returning a `true` or `false` for whether JavaScript should ask for a next frame or not.

After *that*, all the real drawing was done.

This fix didn't change CPU usage a lot, but it *did* remove almost all of the skipped frames.


#### General point

> "Don't think too much!" -- *my old German teacher*

Your browser is already the result of 25+ years of the biggest minds on the web working together. It's now 2020, and you can place a lot more trust into its inner workings than, say, 10 years ago. Some of my performance problems were due to my overthinking, and the resulting *overengineering*.

**Tip 5: Only fix problems that are real problems; don't waste your time by making assumptions that will fix a non-problem.**


### Be pedantic

As a final optimization step, I decided to see how much I could optimize the [AssemblyScript compiler](https://www.assemblyscript.org/compiler.html#command-line-options). As it turns out, there were quite a few things I could influence; like turning off the not-needed bundled runtime, and the C++-like `-O3z` to get the smallest binary possible.

One option, however, stood out to me:

```
--pedantic        Make yourself sad for no good reason.
```

This option will show all the nitty gritty compiler warnings, going from 0 to 100+ after turning this on. I first decided to ignore this, but instead of hiding my sadness, I took it upon myself to face it head-on.

Two things I learned from this:

1. If you're using arrays with indexed access (like `array[5]`) inside AssemblyScript, there will be bounds checking to see if there are no overflows, decreasing performance. Using [`unchecked(array[5])`](https://www.assemblyscript.org/memory.html#notes) will fix this problem, and makes the compiler trust the indices you provide. The code will look really weird, but it works;

2. AssemblyScript doesn't handle circular references optimally. For instance, it makes sense to have a `Micrio.Camera` object, where the `Camera` also has a `Camera.Micrio` object, so it can refer to its own main image. Pedantic AssemblyScript warns about this, as it cannot compile optimally (which makes sense). So, for all functions requiring another class instance like `Micrio`, I just pass it as the first function argument. It's a whole different way of thinking, but keeps your classes fundamentally separated.


### The final test results are in

After the optimizations, of which none could be singled out to be *the thing that fixed it all*, the numbers were looking quite differently:


![Last benchmarks were overwhelming](https://raw.githubusercontent.com/marcelduin/Micrio.Blogpost.wasm-v3/master/img/bench-2.png "Time for celebration!")

Or for a better comparison:

![65% less CPU used in 3.0](https://raw.githubusercontent.com/marcelduin/Micrio.Blogpost.wasm-v3/master/img/bench-results.png "Numbers don't lie")

:tada: **65% less CPU used than in 2.9!** :metal:

Above that, all frame skips were gone, and there was less memory used. These results were also the same over multiple runs.

This proved to me that the whole operation was worth it. I was ecstatic, and tired of the early mornings and nights that the optimizing took me.



## Going to production

Now that we have our new Micrio running great in the garage, we need to make it street worthy.

There are a few important constraints for the production version of the Micrio JS file:

* It needs to be **a single download**: The downloaded JS file should be the *entire runtime*; there shouldn't be an extra HTTP request for the Wasm binary. This would be a lot more hassle for developers, as it immediately loses its stand-alone properties;

* It needs to be **lightweight**: Micrio is a JS library often included in external web projects, and should not affect any download times;

* It needs to **load and work on *all* browsers**: It cannot be the responsibility of the developer to do browser detection and Micrio version selection based on that.



### Keeping it a single download

Older Micrio versions required the developer to include 2 files, the Micrio JavaScript and CSS (for the default layout, markers, popups, etc), separately. I never really liked that, since both had their own version number (ie. `micrio-1.9.min.js` and `micrio-1.9.min.css`), and were both a separate HTTP request.

So for version 2.0, I embedded the CSS into the JS, making it one single package containing everything it needs. This worked great.

I wanted to do the same with the Wasm binary to prevent any second HTTP request and the extra disadvantage that Micrio could not work independently without an internet connection.

So, [like before](#52-bundling-the-compiled-wasm-inside-the-js-file), I took the compiled `.wasm` file of 32KB, made it a `base64` string, and included that inside the JS file. It did blow the Wasm up to **42KB** as an ASCII string, but worked great.



### Keeping it lightweight

Since our current version uses Wasm, and since all browsers that support that also support ES6, I knew that *for the browsers able to run this Micrio version*, I didn't have to compile the JS to ES5 anymore.

That means that the [closure compiled](https://developers.google.com/closure/compiler) JS code could use arrow functions, native promises, and much more. Resulting in a much smaller compiled codebase: turning 315KB of JS code into **170KB** of minified JavaScript.

Adding to that our 42KB of base64-encoded Wasm binary, we are left with a bundled, fully working, ES6 executable Micrio JS of **212KB**.

Putting that next to the 2.9 compiled JS of 250KB, that's ([*again*](#73-first-results-and-subsequent-runs)) about a 15% decrease of file size. Also, since the static hosted Micrio JS is delivered using `gzip` compression (not `brotli` because of IE), the resulting download sizes were also improved: **73KB** for Micrio 3.0 vs **84KB** vs 2.9!


### Keeping it compatible

This is where it gets interesting.

Since one of my requirements is that the newly created Micrio 3.0 library can run in **any** (previously compatible) browser, this means that even *Internet Explorer 10* should be able to run the JS.

Which (and I really tried), *it didn't*.

![Surprised pikachu face](https://raw.githubusercontent.com/marcelduin/Micrio.Blogpost.wasm-v3/master/img/pikachu.jpg)

This could simply be resolved including a browser warning for older, non-compatible browsers. Or, instructing that developers using Micrio in their projects should stick to the 2.9 version.

Neither of which I wanted to do:

* Micrio projects are often aimed at broad audiences and should be *all inclusive*-- even your grandparents still using Internet Explorer 10 should be able to enjoy the projects;

* By telling developers to stick to 2.9 for older browsers, the adoption of 3.0 would take forever.


#### Backwards compatibility

So, I wanted to make Micrio 3.0 be executable on *all* browsers. Of course version 3.0 would never run in Internet Explorer 10. So I needed to think of a way of **auto-switching to Micrio 2.9** in the case of incompatible browsers!

Which is not that hard. All I needed was:

* A detection whether this browser is compatible (`if(!window.WebAssembly) { .. }`);
* A way to replace the 3.0 `<script>` tag with the 2.9 version, done *while executing* the main 3.0 JS.

Done in a few lines of code, this works wonderfully!

![The backwards compatibility JS](https://raw.githubusercontent.com/marcelduin/Micrio.Blogpost.wasm-v3/master/img/compatible.png)

However, when putting everything together (the ES6-compiled JavaScript, the Wasm base64 and the compatibility loader), **Internet Explorer refused to run the file**. Because it had unknown JavaScript syntax (ES6) in it further on, it refused to parse the file as a whole and wouldn't even execute the simple backwards compatibility trick at the top.

Bummer!


#### Hackity-hack

..*Did I mention before that I base64-encoded the Wasm binary, so I could bundle it inside the main JS file?*

Instead of giving in to having to compile the brand new and shiny Micrio code to ES5, I decided to **take the ES6-compiled code, base64 encode that *too*, and put *that* in the Micrio JS bundle**, so there wouldn't be any ES6 syntax inside the JS directly.

* **Yes**, it's super ugly.
* **Yes**, it's not developer-friendly.
* **Yes**, it made the file size larger by a lot.

But most importantly, *it worked*! For Internet Explorer, the 4 lines of code to switch the 3.0 JS to 2.9 would run, and for all others, the ES6 code was decoded, and run using `new Function({the decoded ES6 inside a function string})`.

Problem solved! We now had a very ugly bundled JS file that would run Micrio 3.0 perfectly, and automatically switch to Micrio 2.9 for older browsers!

One disadvantage of `base64`-ing all compiled ES6-JS code (170KB), was that it resulted in a **226KB** blob. Adding the Wasm base64, we were left with **268KB** of JS-- or 100KB gzipped. Which was a 20% increase over Micrio 2.9.

Not dramatic, but I *still* felt unsatisfied, since the smaller file size advantage was gone due to *Internet Explorer*.


#### Hackity-hack ^ 2

What I found, was that `gzip` does not work that well with base64-encoded strings. It compresses it, but not as much as the pre-encoded code. This was for both the Wasm binary, and the ES6 code.

If only I could gzip them **before** base64-encoding them..

Long story short: *I did*.

The bundling process for the Micrio JS now looks like this:

* Take the compiled Wasm binary, gzip it (32KB -> **17KB**), and base64-encode it
* Take the closure-compiled JS, gzip it (170KB -> **55KB**), and base64-encode it
* Add the ES5 backwards compatibility loader

The only thing required now was that I should be able to decompress the decoded gzipped blobs *inside the browser*. Unfortunately there is no JS API for this, but [zlib.js](https://github.com/imaya/zlib.js), a minimal footprint JS gzipping library, does the job perfectly.

Leaving me with a neat JS file:

* The ES5 backwards compatibility loader
* The zlib.js include (12KB)
* The gzipped+base64'd Wasm binary
* The gzipped+base64'd JavaScript

Totalling... **101KB**, or **60%** smaller than the 2.9 JS bundle (250KB)! :tada:

Or, gzipped, **74KB**, which was still a **12%** profit over 2.9.

I could now finally sleep peacefully.


## Concluding

*And that* is how I ended up with the weird-looking, but greatly working [micrio-3.0.min.js](https://b.micr.io/micrio-3.0.min.js)!

The client performance is 65% better than its predecessor, the JS file size is 60% smaller, and the codebase became *a lot* cleaner, with a clear separation between AssemblyScript and JavaScript responsibilities.


##\## What's not to like?

I've offloaded the rendering as much as possible to Wasm. Micrio also has a lot of other logic (HTML markers, popups, using the Audio API..), but putting those in Wasm as well would not be beneficial for a number of reasons:

* *Any HTML operation would still require a JS render function*, and the browser's HTML rendering pipeline would not change at all. It would actually be adding an extra step.

* *The Wasm program would lose its static memory size*: since now all it does is render the image, the required memory buffers are generated at runtime, and do not grow over time. When I would add markers and additional elements, which can be dynamically added and removed, the reserved memory size would potentially be expanded during runtime, adding more rebinding logic to it.


WebAssembly is not a fix-all solution: it will never replace HTML, CSS or the basics of JavaScript. It's primarily aimed to make CPU-heavy operations run better inside your browser. So if you are thinking about using it for your projects, firstly think about the benefits you would gain from it. 


### Wishlist

As for Wasm functionality, I've used only the basics since Micrio is quite simple. For more advanced features like multithreading and being able to use reference types, check out the [WebAssembly roadmap](https://webassembly.org/roadmap/), as Wasm is still under heavy active development.


However, what I *would* like to see, is a *single, binary executable download* (`.ewasm`?), which would already include the JS glue file, being able to be run by the browser. It would have several advantages:

* A single HTTP request for a single, fully autonomous package;
* I wouldn't have to do the `base64` hacks [described above](#83-keeping-it-compatible);
* It would maximally profit from `gzip` (or `brotli`) compression.

But, that's only really nice to have.


### Final thoughts

Overall, I'm extremely satisfied with how everything turned out. Not only did I migrate Micrio to WebAssembly as much as it could benefit from it, I also took a step into the future by dropping compatibility for older browsers, while still providing an automatic fallback to 2.9.

Are the performance gains of the new version fully attributable to WebAssembly? Most likely not. I believe the new architecture using WebGL, if implemented in pure JavaScript, would still create a speed profit over the previous version. But having a monolithic and compiled renderer definitely gives even more speed, control, and maintainability. Plus, working in a compilable language for the web is just *awesome*.

For WebAssembly, seeing the possibilities and applications [growing every day](https://madewithwebassembly.com/), I'm extremely excited to see what doors it will open for the future of the web.



### Acknowledgements

Thank you, Jiri, Maurice, Mathijs, Roelf-Jan &amp; Joost for your feedback and proof reading!
