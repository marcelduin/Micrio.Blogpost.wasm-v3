# Micrio v3: The Road From JavaScript To WebAssembly

## Or: How I Learned to Stop Scripting and Love the Binary


## Table of Contents

1. **Introduction**
What is Micrio, what is this article about, setting constraints: client side app migrating from JS to AssemblyScript//WebAssembly.

2. **Current Situation**
Micrio 2.9, short history, techstack, browser compatibility.

3. **(WebAssembly)[#3-webassembly]**

  1. **(The Discovery)[#31-the-discovery]**
  Asm.js old demos, WASM summit at Google Feb 2020

  2. **(The Rewrite)[#32-the-rewrite]**
  The initial application of WASM to Micrio 2.9

  3. **The Realization**
  The *“But can it do more for Micrio?”* process -- how it took 4 weeks to come up with a masterplan

  4. **The Rewrite**
  4 months of back to the drawing board -- back to basics with WebGL and manually created memory buffers

  5. **The Benchmark**
  What and how to measure, what to improve

  6. **The Rewrite**
  Putting everything together in a single JS file, making it work on all browsers, reducing clutter and last minute code optimizations

  7. **The Polish**
  Finishing touches and real world results

4. **Conclusions**
The result: pros and cons. When (not) to use WASM, best practices, thoughts on the future.

5. **Afterthoughts and the future**
Compiling for the web, server microservices using WASM, freedom of choice of programming language, and how it will really change the landscape of technology, the fabric of our world, and might be the ultimate answer of life, the universe, and everything.




## 3. WebAssembly

WebAssembly (WASM) is the ability for your browser to run *compiled* code at (near-) native speeds. It is now recognised by the W3C as the [4th official web programming language](https://www.w3.org/2019/12/pressrelease-wasm-rec.html.en), after HTML, CSS and JavaScript.

Basically, this means you can run compiled code written in a variety of programming languages (`C/C++`, `Rust`, `Go`, `AssemblyScript`, [..](https://github.com/appcypher/awesome-wasm-langs "and many many more")) in your browser, without any need for plugins. In its purest form, you will need some JavaScript to get it running and to communicate with the browser. For instance if you want to have a graphical output such as a game, you will need to link your program to work with available renderers, such as WebGL.

This section will be the tale of my discovery of WebAssembly, and the journey to migrating the 2.9 Micrio version written in plain JavaScript to WebAssembly as much as possible.



### 3.1. The Discovery

Already back in 2013, a [demo was released](https://www.youtube.com/watch?v=BV32Cs_CMqo) by the Mozilla team running Unreal Engine 3 in the Firefox browser at 60fps, using a port of the engine made ready for the web in only 4 days of work.

For me, this was the first application I saw of [asm.js](https://en.wikipedia.org/wiki/Asm.js) -- allowing compiled code to run inside your browser at near-native speeds, using a super-optimized CPU-friendly subset of JavaScript. You could get this to work by compiling to [LLVM](https://en.wikipedia.org/wiki/LLVM)-compatible formats using for instance [emscripten](https://emscripten.org/) for `C/C++`-code. 

This was a definite game changer for the web and left me wanting to try it out myself for the longest time (*spoilers: I never did*). It was also picked up brilliantly by the major browser engines, each in their own optimizing way.

Fast forward to March 2017. [WebAssembly is introduced](https://hacks.mozilla.org/2017/03/why-webassembly-is-faster-than-asm-js/) as an even more powerful way to run binary code in your browser. This joint effort by all major browsers (Firefox, Chrome, Safari and Internet Explorer) was focussed on bundling all separate efforts made so far by running compiled code inside the browser.

To illustrate the importance of its potential, for me as a simple web developer thinking browsers were always competing to be the best and most used browser, I was amazed that *all these browsers have worked together on this*, putting aside their differences.

Now to be honest, in 2017 I totally missed this. I was introduced to WebAssembly two years later, in 2019, when WebAssembly was recognised by the W3C as the fourth official programming language for the web. As soon as I realised there would be a WebAssembly Summit at Google HQ early 2020, I really, *really* wanted to go there to see what's up. I found a colleague at Q42 to join me and in Febuary 2020 we found ourselves in Mountain View, surrounded by an awe-inspiring crowd.

That day was a real eye-opener for me on what WebAssembly can do, was already doing, and the amount of potential it still has to change how the web works. Not only for running compiled code in your browser, but **so much more** (briefly touched upon at the end of this post). You can watch the entire summit [here on YouTube](https://www.youtube.com/watch?v=IBZFJzGnBoU&list=PL6ed-L7Ni0yQ1pCKkw1g3QeN2BQxXvCPK).



### 3.2. The Rewrite


