---
layout: post
sidebar : true
blog: true
newsletter: true
post_title:  "Building an Asynchronous C++ Addon for Node.js using Nan"
title:  "Building an Asynchronous C++ Addon for Node.js using Nan"
date:   2015-12-31 15:40:56
permalink: /building-an-asynchronous-c-addon-for-node-js-using-nan/
name: building-an-asynchronous-c-addon-for-node-js-using-nan
disqus_id : silvrback-scottfrees-20931
disqus_shortname: scottfrees
---
This post is the fourth and final in a series dedicated to showing you how to get your C++ application onto the web by integrating with Node.js.  In the [first post](/getting-your-c-to-the-web-with-node-js), I outlined three general options:
<!--more-->
1. **[Automation](/automating-a-c-program-from-a-node-js-web-app)** - call your C++ as a standalone app in a child process.
2. **[Shared library](/calling-native-c-dlls-from-a-node-js-web-app)** - pack your C++ routines in a shared library (dll) and call those routines from Node.js directly.
3. **Node.js Addon** - compile your C++ code as a native Node.js module/addon.

Each of these options have their advantages and disadvantages, they primarily differ in the degree in which you need to modify your C++, the performance hit you are willing to take when calling C++, and your familiarity / comfort in dealing with Node.js and the V8 API.

**If you haven't read the [first post](/getting-your-c-to-the-web-with-node-js), you might want to check that out first, before going forward.**  The [second post](/automating-a-c-program-from-a-node-js-web-app) covered automation in detail, and also introduced the C++ code that I'm focused on calling - a prime number implementation found [here](https://gist.github.com/freezer333/ee7c9880c26d3bf83b8e).  I covered the shared library route in the [third post](/calling-native-c-dlls-from-a-node-js-web-app).

This is about building native addon modules for Node.js.  The material builds on a lot of the topics I've described [previously](/c-processing-from-node-js), especially [asynchronous addons](/c-processing-from-node-js-part-4-asynchronous-addons).  I'll use [Nan](https://github.com/nodejs/nan) to simplify the development of asynchronous addons and also shield us from API-breaking changes that tend to crop up with [V8](https://developers.google.com/v8/?hl=en) - the engine that Node.js relies upon.   

## Why develop an addon?
Well... let's think first about "why not?".  If you don't have access to the source code of your legacy C++ application, then [automation](/automating-a-c-program-from-a-node-js-web-app) is your best option - you won't be able to create the type of Node.js addon I'll describe here.  Of course, if your legacy code is *not C or C+++*, then automation might very well be your best bet as well (although there are indeed bindings from Node to other languages as well).  If your C/C++ code is already in a dll or shared library, then of course it likely makes the most sense to use FFI - as described [here](/calling-native-c-dlls-from-a-node-js-web-app).

For situations where you have complete access (and are comfortable editing) the C/C++ you are targeting though, creating a native addon is likely to be the most powerful approach.  First, if your code is already reasonably well organized (clearly defined entry and exit/return points), it won't be too difficult to create the addon itself - especially using Nan.  Second, addons are quite flexible - they can be blocking/synchronous or asynchronous, and support most use cases (i.e. passing/returning objects, arrays, etc.).  Finally, when you create a Node.js addon, your JavaScript code is cleaner than when using the automation or shared library approaches - as you'll see by comparing JavaScript code in this post with the other posts in the series.  You end up calling your C++ as if it is a completely normal JavaScript module in Node.

## Getting the tutorial code
All of the code for this series is available on [github](https://github.com/freezer333/cppwebify-tutorial):

```
> git clone https://github.com/freezer333/cppwebify-tutorial.git
```

For this particular post, checkout the **addon** tag

```
> git checkout addon
```

As described in the earlier posts, I'm using a C implementation of the Prime number sieve as an example.  I've been integrating the C / C++ code into a Node.js web application - with a dedicated route/url for invoking each example technique.

Once you've checked out the code, take a moment to survey the directory structure.  The `/cpp` directory has all the C++ code for primesieve.   In the third post on shared libraries, I modified the primesieve code so output could be collected by an `exchange` class, a pointer to which is passed into the library's entry point as a `void *` since it's used from C, not C++:

```c++
 int generate_primes(int under, void * exchange);
```

That code was placed in `/cpp/prime4lib`, and I'll use it when building the addon in this post.  Please checkout the [shared library post](/calling-native-c-dlls-from-a-node-js-web-app) if you haven't already, so you are up to speed.

We'll now create a Node.js Addon that uses that function and gives us our prime numbers through a simple API:

``` js
// synchronous version
var primes = primenode.getPrimes(under); 

// asynchronous version
nodeprime.getPrimes(100, function (err, primes) {
       console.log(primes);
});
```

# How addons work
If you've done any work in Node.js, then you are very familiar with the following:

```js
var lib = require('some_library_on_npm');
```

Of course, the library need not by written by someone else and/or posted on npm, you can "require" local JavaScript modules on your own machine as well.  The point is, to import a JavaScript module you simply use the `require` keyword and give it a name (or path) to the module.

What you might not know is that `some_library_on_npm` could very well be written in C/C++ instead of JavaScript, and if the module is packaged correctly, you *might never need to know*!

Node allows you to load modules written in C++ as natively compiled binaries - however these C++ modules must utilize the V8 API.  Node.js is really just a set of JavaScript modules that run on top of Google V8 JavaScript engine, which just so happens to be written in ... you guessed it - C++!  The V8 API allows you to write your own C++ modules by registering specific functions in your code with V8.  Once loaded, those functions are exposed to the JavaScript (your Node.js program) executing on top of V8.

Knowing a bit about the V8 API, along with the basics of packaging your module so it can be built with node-gyp on the target machines, you can create modules in C++ that integrate seamlessly with anyone's Node.js application.

The only catch is (1) the V8 API is tricky - due to the complexities of memory management and coordinating resources between the JavaScript executing in the Node.js event loop and your own modules and (2) the API tends to change in subtle (and not so subtle) ways every so often.  To learn all about the details of this, check out my [posts](/c-processing-from-node-js), and my [ebook](/book/).

## Nan - protection from the V8 API
Happily, the folks at [io.js Addon API Working Group](https://github.com/nodejs/nan#governance--contributing) have created and maintained an abstraction layer around the V8 API which can make these challenges a bit easier to deal with.  You can read a lot about it [here](https://nodesource.com/blog/c-add-ons-for-nodejs-v4/)

Nan is ultimately a C++ library, however you install it with npm.  First, we'll create a new project (folder) for our soon-to-be addon.  I've created it at `/cpp/nodeprime_sync`.  Now we'll need to install Nan with the following command:

```
> npm install --save nan
```

That command will download Nan and put it in the `node_modules` directory under `/cpp/nodeprime_sync)`.  In addition to the C++ API defined in V8, we now have a bunch of headers (and a namespace) for `Nan` that we can use in our C++ module.

# Synchronous Addon Code
Next up, we create our C++ addon file - `/cpp/nodeprime_sync/addon.cpp`.  We are going to create a wrapper around the `generate_primes` function found in `/cpp/prime4lib` and register it with V8 using macros defined in V8 and Nan.  First, we'll include the headers for the primesieve code, the exchange class we use to collect data from the primseieve code, and V8/Nan:

```c++
#include <nan.h>  // includes v8 too
#include <functional>
#include <iostream>

#include "exchange.h"  // class to hold values returned from primesieve
#include "prime_sieve.h"

// bring in the required namespaces
using namespace Nan;
using namespace v8;
using namespace std;
```  

Now let's create a function that will do the calculation.  Much of it is familiar from the shared library post, we'll use the `exchange` class to collect output from primesieve.  The main difference is that we'll be collecting the data into a V8 Local Array, which can be returned wholesale to the calling JavaScript code.  Before diving into the C++, here's how the function will (almost) be used in JavaScript:

```js
var primes = primenode.getPrimes(under);
// primes is now the array of all prime numbers less than under
```

So, here's the C++:

```c++
NAN_METHOD(CalculatePrimes) {
    Nan:: HandleScope scope;
    
    int under = To<int>(info[0]).FromJust();
    v8::Local<v8::Array> results = New<v8::Array>(under);

    int i = 0;
    exchange x(
        [&](void * data) {
            Nan::Set(results, i, New<v8::Number>(*((int *) data)));
            i++;
       });

    generate_primes(under, (void*)&x);

    info.GetReturnValue().Set(results);
}
```

First, the function signature is actually a call to a Nan macro.  `NAN_METHOD` decorates the `CalculatePrimes` function to have the correct parameters and return types - which are highly specific to the version of V8 you are using.

The first line in the function creates a scope, an important aspect of the V8 memory architecture (but something I'm not going to get too far into now!).  You can think of a scope object as a slab of memory - like a stack frame - where your V8 local variables are created within.  V8 local variables can be passed back to JavaScript calling code, and can expose data coming from the calling JavaScript as well.

The next line of code is where we extract the parameter this function would be called with.   Since JavaScript plays fast and loose with the number of parameters a function can be called with, Nan wraps the actual call signature in an `info` object.  It's an array of parameters, representing the parameters the JavaScript code called the function with.  

```c++
int under = To<int>(info[0]).FromJust();
```

We are simply casting this to a standard integer.  Next, we allocate a new local V8 array that will be our return value - an array of prime numbers less than "under".

```c++
// conservatively assume under is the maximum number of primes
v8::Local<v8::Array> results = New<v8::Array>(under);
```

Next is our exchange class, from the last post.  We are creating a callback, which primesieve (`generate_primes`) will call each time a prime number is found.  Here, instead of adding each prime number to a vector, we are adding it to the local V8 array we have declared.  Note that the array will be "oversized", since if  "under" is 100 there is clearly not 100 prime numbers less than 100!  Each element that is not explicitely set will be set to `undefined` when accessed through JavaScript later.  We now call the primesieve implementation, which executes and incrementally fills up the array with primes through the exchange object.

```c++
int i = 0;
exchange x(
    [&](void * data) {
        Nan::Set(results, i, New<v8::Number>(*((int *) data)));
        i++;
   });

generate_primes(under, (void*)&x);
```

All that's left to do is return the data to JavaScript - which is done through the `info` object.

```c++
info.GetReturnValue().Set(results);
```

At this point, control will be returned to our JavaScript.

Finally, we need to register this function (`CalculatePrimes`) with V8 correctly, which we do at the bottom of the file:

```c++
NAN_MODULE_INIT(Init) {
    Nan::Set(target, New<String>("getPrimes").ToLocalChecked(),
        GetFunction(New<FunctionTemplate>(CalculatePrimes)).ToLocalChecked());
}

NODE_MODULE(addon, Init)
```

The first macro associates an exported "getPrimes" function with `CalculatePrimes`.  It's an initialization function that V8 will call when the addon is loaded.  The last macro (V8, not Nan) tells V8 to call that very `Init` function.

## Building the addon
As in all of the posts in this series, I'll use node-gyp to build the addon.  I've created a `binding.gyp` file:

```js
{
  "targets": [
    {
      "target_name": "nodeprime",
      "sources": [ "../prime4lib/prime_sieve.c", 
                   "../prime4lib/exchange.cpp", 
                   "addon.cpp"],
      "cflags": ["-Wall", "-std=c++11"],
      "include_dirs" : ['../prime4lib', "<!(node -e \"require('nan')\")"],
      "conditions": [ 
        [ 'OS=="mac"', { 
            "xcode_settings": { 
                'OTHER_CPLUSPLUSFLAGS' : ['-std=c++11','-stdlib=libc++'], 
                'OTHER_LDFLAGS': ['-stdlib=libc++'], 
                'MACOSX_DEPLOYMENT_TARGET': '10.7' } 
            }
        ] 
      ] 
    }
  ]
}
``` 

There are a few new things in this bindings file.  First, notice there is no "type" property - by default node-gyp builds a Node.js addon - so no need to specify anything.  I've defined the target to be `nodeprime`, and the output of the build will end up being `nodeprime.node`.  In addition to specifying the build files, and the include directory for primesieve, I've also added Nan to the set of include directories - utilizing a node shell command.  The rest (conditions) is the same compiler stuff from the previous posts, mainly to enable C++ 11.

To build, do a `node-gyp configure build` from `/cpp/nodeprime_sync`.  The `nodeprime.node` file will be located in `/cpp/nodeprimes_sync/build/Release` - which we'll link to in a moment.

## Calling from JavaScript
Now comes the easy part!  First, we need to require the module.  We specify a path in the require command

```js
var nodeprime = require("./cpp/nodeprime_sync/build/Release/nodeprime")

```

Now to get prime numbers under 100, just call the function:

```js
var retval = nodeprime.getPrimes(100);
console.log(retval);
```

`retval` is now just a JavaScript array - however you'll see that there are 100 elements (mostly empty), since we over-allocated in C++.  We can get rid of that pretty easily though:

```js
var retval = nodeprime.getPrimes(100)
            .filter(function(val) { 
                return val != undefined
            });
console.log(retval);
```

# Asynchronous Addon Code
Synchronous code is a real problem if you are integrating a web application.  If the JavaScript code above were executed in response to an HTTP request, no other requests can be processed until the array is returned.  The C++ code is executing in the Node.js event loop.  **Bad**

## Understanding the event-loop / worker thread relationship
We need to get the prime number computation off the Node.js event loop and into another thread.  This actually is fairly easy, thanks to Nan - which abstracts the way you'd normally need to interact with libuv, Node's event/threading framework.  

The key to understanding Nan's abstractions and my code is to recognize that when your JavaScript calls your addon, all data coming from JavaScript belongs to V8.  We are going to launch a worker thread to do the computation, which allows the C++ code initally invoked to return back to JavaScript.  That's critical, because it means any data the worker thread accesses must **not** belong to JavaScript, as the scope of the initial invocation of your C++ code is destroyed.

Likewise, data *produced* inside the worker thread is inaccessible to JavaScript, so when we fire a callback into JavaScript we'll need to do so with newly created data local to V8 and the event loop.  There's a lot more detail on this in my [asynchronous V8 post](/c-processing-from-node-js-part-4-asynchronous-addons).

Thankfully, Nan provides a class - called `AsyncWorker` which sets up an easy pattern for doing all of this - making the thread creation and interaction with libuv completely transparent.  All we have to do is extend `AsyncWorker` and plug our code in.

## Asynchronous Worker Code
I've created the asynchronous addon in `/cpp/nodeprime`.  Within that folder, you'll see a package.json file (you need to do a `npm install`) that sets up Nan.  You'll also see a similar binding.gyp file as before, and another addon.cpp file that contains the asynchronous addon.

First off, inside `addon.cpp`, you'll see the top part (includes/namespaces) and bottom part (`NAN_METHOD` and `NODE_MODULE`) are exactly the same.  The change now is how `CalculatePrimes` is implemented, and the addition of the PrimeWorker class, which inherits `AsyncWorker` and contains all the logic for doing the work.  Before diving in, let's look at what the calling JavaScript code will eventually look like:

```js
// Asynchronously get all prime numbers under 100
nodeprime.getPrimes(100, function (err, primes) {
       console.log(primes);
});
```

Notice that `getPrimes` now gets two parameters, "under" and a callback function that receives the result when it's complete.  That's where we'll start in the C++, because we need to get a reference to that callback so we can invoke it:

```c++
NAN_METHOD(CalculatePrimes) {
    int under = To<int>(info[0]).FromJust();
    Callback *callback = new Callback(info[1].As<Function>());

    AsyncQueueWorker(new PrimeWorker(callback, under));
}

```

Notice now that `CalculatePrimes` extracts two parameters - under and the callback.   Now, instead of actually computing the prime numbers, we call `AsyncQueueWorker` with an instance of our `PrimeWorker` class. Our `PrimeWorker` class is created with the callback and under parameter, since they will be used to process the work.  `AsyncQueueWorker` returns *immediately* - it simply queues the worker.  The C++ now returns control right back to the calling JavaScript code.

Now let's look at what is actually going on inside the `PrimeWorker` class.  The constructor is pretty simple - most importantly it initializes the base class `AsyncWorker` with the callback sent in from JavaScript.  The `under` value is also saved in `PrimeWorker`, and the primes vector that will hold our prime number results is intialized.

```c++
PrimeWorker(Callback *callback, int under)
        : AsyncWorker(callback), under(under), primes(0) {}
```

Here's where there is a big difference from the synchronous addon - we're going to save the prime numbers in a standard C++ vector as opposed to directly into a V8 Local Array.  This is because the prime numbers are being calculated in a worker thread, not in the event loop.

Once we call `AsyncQueueWorker` from `CalculatePrimes`, libuv will dispatch our `PrimeWorker` object onto a worker thread and call it's `Execute` method - which is shown below:

```c++
void Execute () {
  exchange x(
    [&](void * data) {
      primes.push_back(*((int *) data));
    }
  );
  
  generate_primes(under, (void*)&x);
}
```

This is pretty much *exactly* what the synchronous version of `CalculatePrimes` did - it's just being executed in the worker thread.  Once `Execute` completes, libuv will automatically call the `HandleOKCallback` methods on `PrimeWorker` - in the event loop.  

```c++
void HandleOKCallback () {
  Nan:: HandleScope scope;

  v8::Local<v8::Array> results = New<v8::Array>(primes.size());
  int i = 0;
  for_each(primes.begin(), primes.end(),
    [&](int value) {
      Nan::Set(results, i, New<v8::Number>(value));
      i++;
    });


  Local<Value> argv[] = { Null(), results };
  callback->Call(2, argv);
}
```
Since this method is actually called in the Node event loop thread, we can allocate a V8 Local Array that will be returned back to JavaScript.  We create a scope, initialize an array (this time, exactly the right size, since we already have the vector with the primes).  Next we use a `for_each` to fill the array.

The final step is to actually invoke the JavaScript callback that was sent in as an initial parameter to our addon.  We pack an arguments array representing the parameters (Null first, since there is no error, and then the array).  We end by executing the callback - at which time control is sent back to JavaScript again.

You'll need to do another `node-gyp configure build` to build this module, and now we can call it from Node.js.

## Calling from JavaScript
As shown above, we just need to `require` the module now:

```js
var nodeprime = require("./cpp/nodeprime/build/Release/nodeprime")
```

Now to get prime numbers under 100, just call the function - passing in a callback that will be invoked once the prime numbers are generated:

```js
primes.getPrimes(100, function (err, primes){
  console.log(primes);
});

```

# Putting it all together...
OK... so lets put this on the web app we've been developing over the last few posts.  I'll show you the asynchronous version, since the synchronous model really doesn't play well with the web.

Inside the `/web/index.js` file we are going to add another entry for a route called `ffi`.  

```js
var types = [
  {
    title: "pure_node",
    description: "Execute a really primitive implementation of prime sieve in Node.js"
  },
  //... the entries for the automation and shared library routes...
  {
    title: "addon",
    description: "Creating a Node Addon that can be called like any other module.  Based on /cpp/nodeprime"
  }
  ];
```

That type array is used to create the routes by looking for a file named after each `title` property in the `web/routes/` directory:

```js
types.forEach(function (type) {
    app.use('/'+type.title, require('./routes/' + type.title));
});
```

Now let's add our route in `/web/routes/addon.js`.  The relevant post handler is below, and it looks a lot like the the code we've already seen:

```js
router.post('/', function(req, res) {
    var under = parseInt(req.body.under);
    primes.getPrimes(under, function (err, primes) {
        res.setHeader('Content-Type', 'application/json');
        res.end(JSON.stringify({
            results: primes
        }));
    });
    console.log("Primes generated using " + type);
});
````
Fire up the web app (`node index` from `/web`) and try the link for `addon`.  No surprises.

# Conclusion
And that's it!  Over the last four posts I've outlined some useful methods for getting C++ onto the web.  I've gotten a bunch of feedback from readers about other approaches as well - like using [emscripten](https://github.com/kripken/emscripten) to compile C++ into JavaScript (thank you [Hacker News](https://news.ycombinator.com/item?id=10809475)!).  I've detailed [automation](/automating-a-c-program-from-a-node-js-web-app) through child processes, using ffi to invoke [shared libraries](/calling-native-c-dlls-from-a-node-js-web-app), and in this post I've shown you how to use Nan to build an asynchronous addon that can be seamlessly integrated into your Node.js app.  Hope this information has helped!

**P.S.** If you are looking for some more info on writing Node.js addons in C++, check my [ebook](/book/) on Node and C++ integration, along with my previous posts below. You can [buy it here](https://gumroad.com/l/dTVf).

- [C++ processing from Node.js - An Introduction](/c-processing-from-node-js)
- [C++ processing from Node.js - Returning objects to JavaScript from C++](/c-processing-from-node-js-part-2)
- [C++ processing from Node.js - Passing Arrays of Objects](/c-processing-from-node-js-part-3-arrays)
- [C++ processing from Node.js - Asynchronous addons](/c-processing-from-node-js-part-4-asynchronous-addons)
- [C++ and Node.js Integration - ebook](/book/)