---
layout: post
sidebar : true
blog: true
title:  "Streaming data from C++ to Node.js"
date:   2016-07-20 15:40:56
permalink: /streaming-data-from-c-to-node-js/
name: streaming-data-from-c-to-node-js
disqus_id : silvrback-scottfrees-26006
disqus_shortname: scottfrees
---
If you've worked with C++ addons for Node.js before, you know there's a lot of work that goes into making them play well with the rest of your JavaScript program.  Once you learn how to keep up with V8 changes using [NAN](https://github.com/nodejs/nan) and how to deal with [C++/JavaScript data type conversions](http://blog.scottfrees.com/type-conversions-from-javascript-to-c-in-v8), you also need to deal with asynchronous addon functions.  The asynchronous pattern - [which I've posted about before](https://blog.scottfrees.com/c-processing-from-node-js-part-4-asynchronous-addons) - is tricky, to say the least.  Using the asynchronous pattern lets Node.js code call into C++ and receive data through a callback - allowing the C++ addon to do it's work inside a separate thread.  This allows the addon code to run on the CPU for long stretches without tying up Node.js's event loop - a big win.
<!--more-->
Once you've mastered all this though, you'll want to start thinking about how you can support *higher level* abstractions for moving data - like streaming data between Node.js and C++.  This post covers part one of the topic - **streaming data from C++ to Node.js**, and I'll follow this up with another post covering streaming the other way soon.  

In both posts, I'll show you how to create event-based and streaming interfaces to addons using the [`streaming-worker`](https://www.npmjs.com/package/streaming-worker) and [`streaming-worker-sdk`](https://www.npmjs.com/package/streaming-worker-sdk) modules - which I've adapted from the Streaming chapter in my ebook:  [C++ and JavaScript Integration](https://scottfrees.com/ebooks/nodecpp/).  In the book, I go over the details of how `streaming-worker` and `streaming-worker-sdk` actually works internally.  In these posts I'm just going to show you how to use them in your own addons. 

## Example - sensor data
Believe it or not, performance isn't the most common reason we create C++ addons for Node.js.  While there are certainly times where C++ code runs much more efficiently than JavaScript, usually that performance gain is eaten up by the overhead of marshaling data between V8 memory and C++ proper.  While workarounds exist (Buffers), performance isn't the main reason C++ addons are useful.  Perhaps the most common reason addons are used is their ability to *leverage existing C++* code.  This can be especially critical when interacting with devices - specifically devices that only provide C/C++ API's.  

My background includes a lot of work in virtual reality, and I've found that position/orientation sensors commonly ship with only C or C++ API's.  If you want to push position/orientation data from your Oculus Rift into a Node-webkit/electron app - then an addon can be your answer.  

I'm going to keep things simple in these posts - no need to plug in your VR HMD... but the concepts will transfer to interacting with virtually any input device with a C/C++ api.  Instead of using an actual device, I'm going to have the C++ code emit noisy (random) positional data.  I'll forego orientation data (it's the same idea), and instead of building a VR app around the sensor data, I'm just going to dump the data to the screen (from Node.js of course).  The focus will be on the streaming interface between the addon and Node.js.

By the time we're done, we'll have an addon that can be listened to as an event emitter or as an input stream - sort of like this:

```js
...
const worker = require("streaming-worker");
const wobbly_sensor = worker(addon_path, {name: "Head Mounted Display"});

// Option 1 - the emitter interface
wobbly_sensor.from.on('position_sample', function(sample){
	console.log(JSON.parse(sample));
});

// Option 2 - the stream interface
var output = wobbly_sensor.from.stream();
output.pipe(process.stdout);
```

In the code above, `wobbly_sensor` is an addon that (could) interact with a C/C++ api to pull actual position data from a VR sensor - but here it's just random x/y/z coordinates.  Once instantiated, the sensor addon will start registering/sending position coordinates until it is told to stop.  The sensor addon code will be executing in a worker thread - the JavaScript above is completely asynchronous.  

## Understanding `streaming-worker`
The full source code for this example is [here](https://github.com/freezer333/streaming-worker).  I'm going to start out by setting up two directories - `/addon` and `/js`.  The C++ project and code will be in `/addon`, and we'll be building it with `node-gyp`.  If you've never worked on addons before, stop here and check out my [post on the basics](https://blog.scottfrees.com/c-processing-from-node-js) before continuing.   The `/js` directory will hold just the JavaScript program - which will have a dependency on the addon.

Before diving into the code, let me explain a bit about what `streaming-worker-sdk` and `streaming-worker` are.  In order to have an asynchronous addon that continously sends information back to Node.js through callbacks (and events/stream data), you need to manage your own thread synchronization and handle V8 Local/Persistent handles carefully.  I've posted articles explaining the ins and outs of it [using pure V8 code](https://blog.scottfrees.com/c-processing-from-node-js-part-4-asynchronous-addons) and [using NAN](https://blog.scottfrees.com/building-an-asynchronous-c-addon-for-node-js-using-nan) - but in `streaming-worker-sdk` I've abstracted away much of the details for you - it gives the addon developer a simple interface to send a continuous stream of updates to Node.js.  `streaming-worker-sdk` is the C++ header file(s) that contain all this logic. To access it, you'll inherit from a class called `StreamingWorker` included in the SDK.  That class will be run (it's `Execute` method, that is) in a seperate thread, and you'll be able to send name/value pairs back to Node.js through a provided API.

Messages that get sent from the addon back to JavaScript could be dealt with directly in your own JavaScript code - but they aren't particularly nice to work with.  Instead, your JavaScript code will utilize the `streaming-worker` module - which adapts the addon's output into both an EventEmitter-like interface and a set of streams.

## Setting up the addon project
Inside the `/addon` directory, let's start out by creating a package.json for the sensor addon.

```js
{
  "name": "sensor_sim",
  "version": "0.0.1",
  "gypfile": true,
  "dependencies": {
    "nan": "*",
    "streaming-worker-sdk": "*"
  }
}
```

Note `gypfile` is set to true - this is going to be a C++ addon, not something that is meant to be directly executed (and thus, no entry point is defined).  The dependencies include [`NAN`](https://github.com/nodejs/nan) and the actual SDK, `streaming-worker-sdk`.  While you won't necessarily need to use much of NAN when creating your addon, the SDK relies upon it.

Up next, let's create the `binding.gyp` file.  If you are foggy on addon development, the C++ code we create will be built with `node-gyp`, a cross-platform build system that uses your native compiler to build addon executables.  `binding.gyp` allows you to configure the build - setting up include directories and specifying compiler options.  If you are using a relatively new version of clang/g++/msvs/xcode, the following file will be sufficient.  If you are using an older compiler, you may need to add some additional flags to enable full C++ 11 support.

```js
{
  "targets": [
    {
      "target_name": "sensor_sim",
      "sources": [ "sensor_sim.cpp" ], 
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

Note above we're going to create the C++ addon code in `sensor_sim.cpp` (sources).  The most important part though is the `include_dirs`.  When we compile the addon, we'll be depending on NAN and the streaming SDK.  The package.json file from earlier will instruct npm to download NAN and `streaming-worker-sdk` from the npm registry.  Unlike normal modules, these aren't JS code - they are C++ headers, but they will be placed in `node_modules` nonetheless.  The `&lt;!(node -e \"require`... magic is instructing node-gyp to include the actual directories of those modules in the build path.

## C++ Sensor code 
Now let's create the `sensor_sim.cpp` file.  We'll start out by including the basic headers/libs we'll need to get off the ground.

```cpp
#include &lt;iostream&gt;
#include &lt;chrono&gt;
#include &lt;random&gt;
#include &lt;thread&gt;
#include "streaming-worker.h"
#include "json.hpp"  //https://github.com/nlohmann/json

using namespace std;
using json = nlohmann::json;
````

Most of these are self-explanatory.   I'm going to produce random positional datapoints, so I'm including `random`.  I'm also going to generate these datapoints once every few milliseconds, so I'll use `chrono` and `thread` to sleep for specific intervals.  `streaming-worker.h` is going to give me the streaming logic - specifically the `StreamingWorker` I'll inherit from.  Finally, I've included `json.hpp` - a fantastic JSON library for C++ found [here](https://github.com/nlohmann/json).  Please download that `hpp` file too, and put it directly in the `/addon` directory.  We'll end up using this to construct position samples in C++ that can be consumed as JSON in JavaScript - something like this:

```js
{ 
    position:
    { 
        x: -0.0505441173468603,
        y: -0.452091167700998,
        z: 0.49470661007039 
    },
    sensor: 'Head Mounted Display' 
}
```

All streaming addons using `streaming-worker-sdk` are implemented as a class, extending `StreamingWorker`.  `StreamingWorker` is configured by passing in three callback functions into it's constructor.  While you don't need to be concerned with where these callbacks will come from (they will be provided by the sdk itself), they represent callbacks invoked when data updates are present, errors occur, and when the addon completes.  

```cpp
class Sensor : public StreamingWorker {
  public:
    Sensor(Callback *data, Callback *complete, 
		Callback *error_callback,  v8::Local&lt;v8::Object&gt; &amp; options) 
          : StreamingWorker(data, complete, error_callback){

      // name is a member variable in Sensor
      name = "default sensor";
      // interrogate the options object to see if there is a "name" property
      if (options-&gt;IsObject() ) {
        v8::Local&lt;v8::Value&gt; name_ = options-&gt;Get(New&lt;v8::String&gt;("name").ToLocalChecked());
        if ( name_-&gt;IsString() ) {
          v8::String::Utf8Value s(name_);
          name = *s;
        }
      }
    }
```

In this example, `Sensor`'s constructor accepts an optional `option` object - which is a V8 local object handle that can contain a JavaScript object with aribitrary data.  Later we'll see how this is used in this example to name the sensor - but we're defaulting to "default sensor".

Once the addon is created, the base class (`StreamingWorker`) will queue your addon up as a worker thread, hooked into `lib_uv`'s event loop.  Your addon will be invoked within it's `Execute` function.  **Beware, this function is executed in a worker thread, not the Node.js event loop thread!**.  Synchronization when accessing any state variables (member variables) is your responsibility.  The `Execute` function must accept a `progress` object, which is defined by `nan`'s [AsyncProgressWorker](https://github.com/nodejs/nan/blob/master/doc/asyncworker.md) class.  You won't use it directly, but you'll pass it along to the sending method when you want to send messages to Node.js.

You communicate with Node.js through `Message` objects - which are simple name value pairs (values are strings, feel free to extend the implementation to accomodate other types/templates!).

```cpp
/// defined in streaming-worker.h
class Message {
public:
  string name;
  string data;
  Message(string name, string data) : name(name), data(data){}
};
```

Your addon inherits a thread-safe method for sending messages back to JavaScript, called `writeToNode`.  This call requires two parameters, the `progress` object passed into your `Execute` method, and the message to send.  

Here's our `Execute` implementation for `Sensor`, which sits in a loop and generates random position sample data every 50 milliseconds.  In a real addon of course, you'd be making a C/C++ api call to the device instead of computing random data!

```cpp

// member of Sensor
void Execute (const AsyncProgressWorker::ExecutionProgress&amp; progress) {
   std::random_device rd;
   std::uniform_real_distribution&lt;double&gt; pos_dist(-1.0, 1.0);
   while (!closed()) {
      json sample;
      sample["sensor"] = name;
      sample["position"]["x"] = pos_dist(rd);
      sample["position"]["y"] = pos_dist(rd);
      sample["position"]["z"] = pos_dist(rd);

      // serialize the json samplet to a string and send to JavaScript
      Message tosend("position_sample", sample.dump());
      writeToNode(progress, tosend);
        
      std::this_thread::sleep_for(chrono::milliseconds(50));
   }
}
```

Behind the scenes, each time we make a `writeToNode` call, an event is being queue'd to `lib_uv`.  Any output collected from the addon between events is cached in a thread-safe queue, and then sent back to Node in the event loop's thread when lib_uv processes the event.

Notice the `closed()` function call -  which we invoke at the top of each turn around that loop.  This function will return true when a signal is sent from JavaScript to close the addon - this will be automatically handled by the SDK.

Our final step is implementing afactory function declared - but not defined - in `streaming-worker.h`.   You MUST include this function, and you cannot alter the signature at all.  The base wrapper class calls this to build your particular worker.  

```cpp
StreamingWorker * create_worker(Callback *data, Callback *complete
    , Callback *error_callback, v8::Local&lt;v8::Object&gt; &amp; options) {

  return new Sensor(data, complete, error_callback, options);
}
```
Lastly, you need to initialize the module using the standard Node.js addon convention.  

```cpp
NODE_MODULE(sensor_sim, StreamWorkerWrapper::Init)
```

This line of code registers the `Init` function within the SDK as the addon's entry point.  It ultimately sets up the addon so JavaScript can instantiate instances of `Sensor` through the `create_worker` function defined above.

## Building the addon
To build the addon we need to pull down the dependencies.  Do an `npm install` from the `/addon` directory - which should download `nan` and `streaming-worker-sdk` based on the dependencies we put in `package.json`.

```
- /addon
    - binding.gyp
    - json.hpp
    - package.json
    - sensor_sim.cpp
    - node_modules/
```

After pulling down the dependencies, npm will build the addon - placing the executable in `/addon/build/Release` in a file called `sensor_sim.node`.  **This is the binary we will now interact with from JavaScript.**

## JavaScript program
Let's move over now to the `js` directory, where we'll write the JavaScript program to use the addon.  We'll start again with `package.json` to declare our dependencies.

```js
{
  "name": "sensor_sim",
  "version": "0.0.1",
  "scripts": {"start": "node sensor_sim.js"},
  "dependencies": {
    "streaming-worker": "*",
    "through": "^2.3.8"
  }
}

```

Note that I've declared `streaming-worker` (not `streaming-worker-sdk`!), which adapts the output of addons created with `streaming-worker-sdk` into event emitters and streams.  I've also included `through` to help out with dumping the output stream from `Sensor` to the screen.

In `sensor_sim.js`, we'll start by using `streaming-worker` to instantiate a sensor object.  

```js
"use strict"; 

const worker = require("streaming-worker");
const through = require('through');
const path = require("path");

const addon_path = path.join(__dirname, "../addon/build/Release/sensor_sim");
const wobbly_sensor = worker(addon_path, {name: "Head Mounted Display"});
```

The `worker` function exported by the `streaming-worker` library is a factory method to create a `StreamingWorker` - through some `ObjectWrap` trickory, it's essentially invoking the `create_worker` function we defined in `sensor_sim.cpp`.  Once instantiated, the addon's `Execute` method is automatically up and running - now it's time to access the data.

### Using the Event Emitter API
`streaming-worker` has added two `EventEmitter`-like interfaces on the object it creates (`wobbly_sensor`) - `to` and `from`.  As you might expect, `to` allows you to emit events *to* your addon - see my next post on this.  The `from` object lets you listen for events sent *from* your addon - which we already did using the `writeToNode` method.

```js
wobbly_sensor.from.on('position_sample', function(sample){
	console.log(JSON.parse(sample));
});
```

In case you forgot, here was the C++ code sending position updates:

```cpp
Message tosend("position_sample", sample.dump());
writeToNode(progress, tosend);

```

Each `Message` we send from C++ is a name/value pair - the value is a string that we created by serializing a position sample.  Now we are using `JSON.parse` to turn that string back into an object.  

If we run this JavaScript program now (`npm start`), we'll get an endless printout of position samples printing to the screen.  Let's make sure it actually stops, by calling the `addon`'s `close` method.

```cpp
setTimeout(function(){wobbly_sensor.close()}, 5000);
```

Now, after 5 seconds, the addon is closed.  Note that calling `close` here just sets a flag within the `Sensor` class such that when it's `closed` function is called (remember we call it at the top of each sensing loop?), it returns true. 

### Using the Streaming API
Ok, so that's not exactly "streaming"... the `streaming-worker` adapter also adds two methods for creating actual streams to and from your C++ code too though.  To capture the output of your C++ via a stream, you can use the `stream()` method on the `from` object:

```js
const out = wobbly_sensor.from.stream();
```

It's likely you'll want to pipe this to something, I recommend getting familiar with [`through`](https://github.com/dominictarr/through).

```js
out.pipe(through(function(data){
    // the data coming in is an array, 
    // Element 0 is the name of the event emitted by the addon (position_sample)
    // Element 1 is the data - which in this case is a JSON object 

	var sample = JSON.parse(data[1]);
	this.queue(sample.position.x 
				+ ", " + sample.position.y 
				+ ", " + sample.position.z 
				+ "\n");
        		
})).pipe(process.stdout)
```

And there you have it, a streaming interface for an addon generating data asynchronously!  The full source code for all this is at [https://github.com/freezer333/streaming-worker](https://github.com/freezer333/streaming-worker), in the `/examples/sensor_sim` directory.  You'll also find a few other examples that demonstrate other features.   The setup is pretty general - you can use this to create addons that output lots of different types of data.  

There's a ton of V8/NAN work going on behind the scenes to make `streaming-worker-sdk` work, which includes a lot of interesting work with NAN's asynchronous addon patterns and `ObjectWrap`.  If you want to learn how it all works, have a look at the contents of my ebook - [C++ and Node.js Integration](https://scottfrees.com/ebooks/nodecpp/), which can be purchased [here](https://gumroad.com/l/dTVf).  There's also a lot more examples on these topics in my [nodecpp-demo](https://github.com/freezer333/nodecpp-demo), which supports the ebook. 

In my next post, I'll detail how to use these modules for streaming data in the other direction - from Node.js into a C++ addon.
