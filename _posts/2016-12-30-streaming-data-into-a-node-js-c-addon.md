---
layout: post
sidebar : true
blog: true
newsletter: true
post_title:  "Streaming data into a Node.js C++ Addon"
title:  "Streaming data into a Node.js C++ Addon"
date:   2016-12-30 15:40:56
permalink: /streaming-data-into-a-node-js-c-addon/
name: streaming-data-into-a-node-js-c-addon
disqus_id : silvrback-scottfrees-28944
disqus_shortname: scottfrees
---
Earlier this year I [posted an article](/streaming-data-from-c-to-node-js) showing how we can build event-based and streaming interfaces for sending data from Node.js C++ addons to JavaScript.  This mode of interacting with addons can be a lot easier in some situations, especially when your C++ code runs [asynchronously](/c-processing-from-node-js-part-4-asynchronous-addons).  

In this post, I'm going to use the [`streaming-worker`](https://www.npmjs.com/package/streaming-worker) and [`streaming-worker-sdk`](https://www.npmjs.com/package/streaming-worker-sdk) modules - which I've adapted from the Streaming chapter in my ebook: [C++ and JavaScript Integration](/book/). In the book, I cover the details of how streaming-worker and streaming-worker-sdk actually works internally - here I'll just focus on using them. 
<!--more-->

## Working with `streaming-worker`
The full source code for this example is [here](https://github.com/freezer333/streaming-worker).  Let's start out by setting up two directories - `/addon` and `/js`.  The C++ project and code will be in `/addon`.  If you've never worked on addons before, stop here and check out my [post on the basics](/c-processing-from-node-js) before continuing.   The `/js` directory will hold just the JavaScript program - which will have a dependency on the addon.

I've already covered how `streaming-worker-sdk` works [here](/streaming-data-from-c-to-node-js), but as a refresher - `streaming-worker` and `streaming-worker-sdk` are two halves of a library I've created to facilitate event-based and streaming interfaces between addons.  For streaming data to JavaScript from C++, the `streaming-worker-sdk` C++ headers abstract away a lot of the details of working with asynchronous addons.  The `streaming-worker` JavaScript module creates an API for communicating with these addons.   In this post, we'll create an addon that sits in a loop within a worker threads and reads data off a queue abstraction provided by `streaming-worker-sdk`.  Data will be sent to our asynchronous addon using `streaming-worker`'s JavaScript API.

## The example
If you've read this far, you probably have your own use case for emitting data to C++ in mind - so I'm going to keep this example really short - we'll build a simple accumulator.  Before getting into how to build it, I think it's helpful to see how the end-product will be used in JavaScript.

First, the event-emitter API:

``` javascript
"use strict"; 

const worker = require("streaming-worker");
const path = require("path");

// we will build this...
var addon_path = path.join(__dirname, "build/Release/accumulate");
const acc = worker(addon_path);

acc.to.emit("value", 3);
acc.to.emit("value", 16);
acc.to.emit("value", 42);
acc.to.emit("value", -1);

acc.from.on('sum', function(value){
    console.log("Accumulated Sum:  " + value);
});
```

And alternatively, we'll be able to use a streaming API:

``` javascript
"use strict"; 

const worker = require("streaming-worker");
const path = require("path");
const streamify = require('stream-array');

var addon_path = path.join(__dirname, "build/Release/accumulate");
const acc = worker(addon_path);

// create an input stream, with handler for the close event
const input = acc.to.stream("value",
    function () {
        acc.to.emit('value', -1);
    });

streamify([1, 2, 3, 4, 5, 6]).pipe(input);

acc.from.on('sum', function(value){
    console.log("Accumulated Sum:  " + value);
});
```

In both cases, our C++ addon will collect data sent from JavaScript and when a sentinel value or close event is received it will emit a `sum` event with the sum of all numbers sent to it.  

## Setting up the addon project
Inside the `/addon` directory, let's start out by creating a package.json for the accumulator addon.

```js
{
  "name": "accumulator",
  "version": "0.0.1",
  "gypfile": true,
  "dependencies": {
    "nan": "*",
    "streaming-worker-sdk": "*"
  }
}
```

Note that the dependencies include [`NAN`](https://github.com/nodejs/nan) and the actual SDK, `streaming-worker-sdk`.  Next we need to create the `binding.gyp` file.   If you are using a relatively new version of clang/g++/msvs/xcode, the following file will be sufficient.  If you are using an older compiler, you may need to add some additional flags to enable full C++ 11 support.

```js
{
  "targets": [
    {
      "target_name": "accumulator",
      "sources": [ "accumulator.cpp" ], 
      "cflags": ["-Wall", "-std=c++11"],
      "cflags!": [ '-fno-exceptions' ],
      "cflags_cc!": [ '-fno-exceptions' ],
      "include_dirs" : [
		"&lt;!(node -e \"require('nan')\")", 
		"&lt;!(node -e \"require('streaming-worker-sdk')\")"
		]
    }
  ]
}
``` 

Note above we're going to create the C++ addon code in `accumulator.cpp` (sources).  The most important part though is the `include_dirs`.  When we compile the addon, we'll be depending on NAN and the streaming SDK.  The package.json file from earlier will instruct npm to download NAN and `streaming-worker-sdk` from the npm registry.  Unlike normal modules, these aren't JS code - they are C++ headers, but they will be placed in `node_modules` nonetheless.  The `&lt;!(node -e \"require`... magic is instructing node-gyp to include the actual directories of those modules in the build path.



##  Creating the Addon

Inside `accumulator.cpp` we now need to create a class that extends `StreamingWorker` from the `streaming-worker-sdk`.  In it's constructor, we pass along the callbacks that will be created by the other half of the application (JavaScript) and initialize a `sum` variable where we'll accumulate our data.

``` cpp
#include &lt;nan.h&gt;
#include &lt;string&gt;
#include &lt;algorithm&gt;
#include &lt;iostream&gt;
#include &lt;chrono&gt;
#include &lt;thread&gt;
#include "streaming-worker.h"

using namespace std;
class Accumulate : public StreamingWorker {
  public:
    Accumulate(Callback *data, Callback *complete, 
        Callback *error_callback, 
        v8::Local&lt;v8::Object&gt; &amp; options) 
        : StreamingWorker(data, complete, error_callback){

        sum = 0;
    }
    ~Accumulate(){}

    void Execute (const AsyncProgressWorker::ExecutionProgress&amp; progress) {
      int value ;
      do {
        Message m = fromNode.read();
        value = std::stoi(m.data);
        if ( value &gt; 0 ){
            sum += value;
        }
        else {
            Message tosend("sum", std::to_string(sum));
            writeToNode(progress, tosend);
        }
      } while (value &gt; 0);
    }
  private:
    int sum;
};
```
The `StreamingWorker` class will be instantiated in Node's event loop thread when an instance of the addon is created.  It itself extends  `Nan`'s [AsyncProgressWorker](https://github.com/nodejs/nan/blob/master/doc/asyncworker.md) class, and this functionality is used to queue up a new worker thread to run the `Execute` function.  `Execute` just sits in a loop and reads individual messages from the `fromNode` queue, which is an inherited member from the `StreamingWorker`.  This queue holds incoming data sent to the addon from JavaScript.   The queue handles all synchronization issues related to bridging Node's event loop thread and the worker thread `Execute` is running in.  The `read` method on the `fromNode` queue is blocking.

You communicate with Node.js through `Message` objects - which are simple name value pairs (values are strings, feel free to extend the implementation to handle other types using templates!).

```cpp
/// defined in streaming-worker.h
class Message {
public:
  string name;
  string data;
  Message(string name, string data) : name(name), data(data){}
};
```

Each `Message` object read contains string data, which in this case is just converted back to an integer.  The Accumulator treats a negative number as a signal to stop, and return the accumulated sum to JavaScript via `StreamingWorker`'s  `writeToNode` function - which was introduced in the [first post](/streaming-data-from-c-to-node-js).

At the bottom of  `accumulator.cpp` we must also include two functions to setup the addon properly when required from JavaScript.

```cpp
StreamingWorker * create_worker(Callback *data
    , Callback *complete
    , Callback *error_callback, v8::Local&lt;v8::Object&gt; &amp; options) {
 return new Accumulate(data, complete, error_callback, options);
}

NODE_MODULE(accumulate, StreamWorkerWrapper::Init)
```

This is essentially boilerplate code to allow the `streaming-worker` module to package `Accumulator` into a proper Node.js Addon.  If you are interesting in seeing how it's all done, check out the [full source code](https://github.com/freezer333/nodecpp-demo/tree/master/streaming), and [my book](/book/).

A simple `npm install` will build the addon.

## Back to JavaScript

Now let's take a closer look at the JavaScript program shown earlier.  Notice the first thing that is required is the `streaming-worker` module, this module wraps C++ addons created with the `streaming-worker-sdk` to provide event emitter and streaming interfaces.  We instantiate the addon indirectly, by calling the `worker` factory function with the supplied path to the addon executable.

``` javascript
"use strict"; 

const worker = require("streaming-worker");
const path = require("path");

// we will build this...
var addon_path = path.join(__dirname, "build/Release/accumulate");
const acc = worker(addon_path);

acc.to.emit("value", 3);
acc.to.emit("value", 16);
acc.to.emit("value", 42);
acc.to.emit("value", -1);

acc.from.on('sum', function(value){
    console.log("Accumulated Sum:  " + value);
});
```
Once instantiated, the addon is adorned with a `to` event emitter interface, we can emit messages to the addon - in this case a "value" event with an associated integer.  The addon will read this off it's queue to process the data.

Once we emit `-1`, the `Accumulator` addon was written to calculate the sum, and emit the answer back.  We capture that by registering a handler on the `sum` event when fired by the associated `from` emitter connecting the [C++ to JavaScript](/streaming-data-from-c-to-node-js).

When run, you'll get the expected answer of 61.

## Streaming input to C++

While the `streaming-worker` automatically creates event emitters, it does not create a streaming interface unless told to do so.  Each addon is likely to want to be notified that the stream has closed differently (for example, the accumulator detects -1 as a sentinel but other addons could use a 'close' message, or a different sentinel value). To allow for this flexibility, `streaming-worker` accepts a parameterized callback into a `stream` function which creates the input stream.  When the input stream is closed, the callback is invoked - which in this case will just send the -1 sentinel to the accumulator. 

``` javascript
"use strict"; 

const worker = require("streaming-worker");
const path = require("path");
const streamify = require('stream-array');

var addon_path = path.join(__dirname, "build/Release/accumulate");
const acc = worker(addon_path);

const input = acc.to.stream("value",
    function () {
        acc.to.emit('value', -1);
    });

streamify([1, 2, 3, 4, 5, 6]).pipe(input);

acc.from.on('sum', function(value){
    console.log("Accumulated Sum:  " + value);
});
```

## Conclusion

This post was a followup to my initial post using `streaming-worker` to send events and stream data from C++ into JavaScript.  While this is a really simple example, hopefully this post can help you get started using `streaming-worker` for sending data *into* your C++ addons.   The full source code for all this is at https://github.com/freezer333/streaming-worker. You'll also find a few other examples that demonstrate other features. The setup is pretty general - you can use this to create addons that output lots of different types of data.

There's a ton of V8/NAN work going on behind the scenes to make streaming-worker-sdk work, which includes a lot of interesting work with NAN's asynchronous addon patterns and ObjectWrap. If you want to learn how it all works, have a look at the contents of my [ebook - C++ and Node.js Integration](/book/), which can be [purchased here](https://gumroad.com/l/dTVf). 
