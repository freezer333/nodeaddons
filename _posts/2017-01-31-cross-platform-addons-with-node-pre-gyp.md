---
layout: post
sidebar : true
blog: true
title:  "Cross-platform addons with node-pre-gyp"
date:   2017-01-21 15:40:56
permalink: /cross-platform-addons-with-node-pre-gyp/
name: cross-platform-addons-with-node-pre-gyp
disqus_id : nodeaddons-cross-platform-addons-with-node-pre-gyp
disqus_shortname: scottfrees
---
Node.js applications (including electron apps) are being deployed everywhere, and this includes end-user machines without a full development stack.  Usually this is no problem - most distributed Node.js applications are just running JavaScript, and as long as the Node.js runtime is installed, all is well.  

Native addons present a problem though.  Native C++ addons are distributed as modules just like anything else, but then `npm` *builds them* on the target machine during the install using `node-gyp`.  `node-gyp` isn't a compiler, it's build system that adapts to the platform it's on - on Windows it will use Visual Studio and on Linux/macOS it will use the build system installed as well.  When trying to install an application on a machine *without* a compiler, we are out of luck.

Enter `node-pre-gyp`, a convenient tool that lets you deliver pre-built native addons as binary payloads - targeted to the user's platform.  Sounds magical, and in some respects it is!  In this post, I'm going to walk you through how `node-pre-gyp` works.  I'll create an addon that uses platform (OS) specific API's for Windows, macOS, and Linux.  You'll learn how to package a platform-dependent addon into a module that can be deployed to end user machines as binaries.  
<!--more-->

## What is `node-preaTalk-gyp`?
Explain how it works, what the steps are.
Talk a bit about node-abi, platforms, versions.

## Example application - time
Let's get started with an example.  One of the most common reasons devs turn to native addons is to use an operating system's native SDK to access features (webcam, system status, etc.).  To demonstrate this type of scenario, I'm writing an addon that contains a single function that returns the current time.  

*Yes... I know Node.js already has such a provision... and before you point at that C++'s `chrono` brought cross-platform high-resolution timers to C++ years ago, I know :)  I'm writing these addons using the underlying OS API's available in Windows, macOS, and Linux not because it's a smart thing to do - but because it works as a simple, clear example of using native OS calls in an addon.*

Before getting started with the addon project setup, let's first create three platform-specific C++ source files, each containing a `native_now` function which returns the current tick time using OS calls.  While we'll write these three files, the eventual build files will select the appropriate one for the intended platform (see below).  **Note** that the selection will be based on the `_linux.<ext>`, `_mac.<ext>`, and `_win<ext>` suffixes, which is built into `node-gyp`.


First the macOS implementation...
{% highlight cpp %}
// native-rt_mac.cc
// Platform specific (macOS)
#include <mach/mach.h>
#include <mach/mach_time.h>

double native_now() {
  static double timeConvert = 0.0;
  if ( timeConvert == 0.0 ) {
    mach_timebase_info_data_t timeBase;
    (void)mach_timebase_info( &timeBase );
    timeConvert = (double)timeBase.numer /
      (double)timeBase.denom /
      1000000000.0;
  }
  double time_now =  (double)mach_absolute_time( ) * timeConvert;
  return time_now;
}

{% endhighlight %}

Now the linux implementation...

{% highlight cpp %}
// native-rt_linux.cc
// Platform specific (linux)
#include <unistd.h>	
#include <time.h>	
#include <sys/time.h>

double native_now() {
  struct timespec ts;
#if defined(CLOCK_MONOTONIC_RAW)
  const clockid_t id = CLOCK_MONOTONIC_RAW;
#elif defined(CLOCK_REALTIME)
  const clockid_t id = CLOCK_REALTIME;
#else
  const clockid_t id = (clockid_t)-1;
#endif
  if ( id != (clockid_t)-1 && clock_gettime( id, &ts ) != -1 ){
    double time_now =
       (double)ts.tv_sec + 
      (double)ts.tv_nsec / 1000000000.0;
    return time_now;
  }
  return 0;
}
{% endhighlight %}  

And finally the Windows implementation...

{% highlight cpp %}
// native-rt_win.cc
// Platform specific (windows)
#include <Windows.h>

double native_now() {
  FILETIME tm;
  ULONGLONG t;
#if defined(NTDDI_WIN8) && NTDDI_VERSION >= NTDDI_WIN8
  GetSystemTimePreciseAsFileTime(&tm);
#else
  GetSystemTimeAsFileTime(&tm);
#endif
  t = ((ULONGLONG)tm.dwHighDateTime << 32) | (ULONGLONG)tm.dwLowDateTime;
  double time_now = (double)t / 10000000.0;
  return time_now;
}
{% endhighlight %}  

If you want to jam this all into one file and use #if preprocessor conditions to detect the OS, that's fine too - but I'm tend to like using `node-gyp` conditional file inclusion to keep the code a bit easier to read.

Finally, I'm going to create a header file that I'll use to drag this code into my addon - `native-rt.h`

{% highlight cpp %}
// native-rt.h
// to be included by the addon code
double native_now();
{% endhighlight %} 


*Thanks to [Nadeau Software](http://nadeausoftware.com/articles/2012/04/c_c_tip_how_measure_elapsed_real_time_benchmarking) for the code that I adapted to make this example!*

## Setting up the addon project
I'm going to create two projects - the addon, and an example program that uses the addon as a dependency.  I'll create these in their own folder, and we'll start by bringing the 4 source files from above into the addon directory.  In addition, I'm going to start the example program by just adding an `index.js` file to it.

{% highlight plaintext %}
 addon/
   |---- native-rt.h
   |---- native-rt_win.cc
   |---- native-rt_linux.cc
   |---- native-rt_mac.cc
 example/
   |---- index.js

{% endhighlight %}  

The contents of index.js is just a quick program to call our soon-to-be created addon, which will be named `native_rt`.

{% highlight javascript linenos %}
// index.js inside the example project.
var rt = require('native_rt');
var start = rt.now();
setTimeout(function() {
  let end = rt.now();
  console.log(end - start);
}, 1000)
{% endhighlight %}  

### Addon code
Now let's create the addon code.  Inside `/addon`, we'll create a `native-rt.cc` file that will use NAN to create a single addon function called "now", as called online 2 of the example program above.

{% highlight cpp linenos %}
// native-rt.cc inside the addon 
#include <nan.h>
using namespace Nan;
using namespace v8;

#include "native-rt.h"

NAN_METHOD(now) {
  double time_now = native_now();
  Local<Number> retval = Nan::New(time_now);
  info.GetReturnValue().Set(retval);    
}

NAN_MODULE_INIT(Init) {
  Nan::Set(target, New<String>("now").ToLocalChecked(), 
    GetFunction(New<FunctionTemplate>(now)).ToLocalChecked());
}

NODE_MODULE(native_rt, Init)
{% endhighlight %}  

At this point, we need to add a package.json and binding.gyp file to the addon folder so we can build it and include it in the example project.  These files will end up changing a bit when we add `node-pre-gyp` support.  The package.json is straightforward:

### Addon setup
{% highlight json %}
// addon/package.json
{
  "name": "native_rt",
  "version": "1.0.1",
  "description": "Example for using node-pre-gyp for cross-platform binaries",
  "gypfile": true,
  "main": "./build/Release/native_rt",
  "license": "MIT",
  "dependencies": {
    "nan": "^2.3.3"
  }
}
{% endhighlight %} 

Note the dependency on `nan`, and that the entry point has been defined as the actual binary (the filename will be `native_rt.node`).  The `binding.gyp` file is pretty straightforward as well, other than the use of conditionals for including source code files - `node-gyp` will determine the platform and automatically add the right source code files when we set things up this way.

{% highlight json %}
// addon/binding.gyp
{
  "targets": [
    {
      "target_name": "native_rt",
      "sources": [ "native-rt.cc" ],
      "conditions":[
      	["OS=='linux'", {
      	  "sources": [ "native-rt_linux.cc" ]
      	  }],
      	["OS=='mac'", {
      	  "sources": [ "native-rt_mac.cc" ]
      	}],
        ["OS=='win'", {
      	  "sources": [ "native-rt_win.cc" ]
      	}]
      ], 
      "include_dirs" : [
        "<!(node -e \"require('nan')\")"
      ]
    }
  ]
}
{% endhighlight %} 

### Importing the addon
Our addon is actually ready to be published to the npm registry at this point, but since this is just a (halfway done) example, let's include it in the example by defining it as a local dependency.  In the `/example` directory, we'll create another `package.json` file declaring `native-rt` as a dependency:

{% highlight json %}
// example/package.json
{
  "name": "example",
  "version": "1.0.0",
  "main": "index.js",
  "license": "MIT",
  "dependencies": {
    "native_rt": "file:../addon" 
  }
}
{% endhighlight %} 

Now your directory structure should look like this:

{% highlight plaintext %}
 addon/
   |---- native-rt.h
   |---- native-rt_win.cc
   |---- native-rt_linux.cc
   |---- native-rt_mac.cc
   |---- native-rt.cc
   |---- binding.gyp
   |---- package.json
 example/
   |---- index.js
   |---- package.json

{% endhighlight %}  

We can now do an `npm install` from the `example` directory to build the addon and copy it into the `node_modules` directory within the example.  Executing the project will result in the realization that the Node.js timer is not exact...

{% highlight shell %}
$ ~/example npm install
...
$ ~/example node index
1.0026869329158217
{% endhighlight %}

The operative word in the sentance above is of course "build the addon".  The goal of this post is now to turn this addon into a pre-built executable (or set of executables) that can be deployed to different platforms with the same ease-of-use, without the build step requirement.

## Setting up `node-pre-gyp`
The binding file, the package.json file.

{% highlight json %}
{
  "name": "native_rt",
  "version": "1.0.0",
  "description": "Example for using node-pre-gyp for cross-platform binaries",
  "gypfile": true,
  "main": "./index.js",
  "author": "Scott Frees <scott.frees@gmail.com> (http://scottfrees.com/)",
  "license": "MIT",
  "dependencies": {
    "aws-sdk": "2.7.27",
    "nan": "^2.3.3",
    "node-pre-gyp": "0.6.32"
  },
  "binary": {
    "module_name": "native_rt",
    "module_path": "./lib/binding/{configuration}/{node_abi}-{platform}-{arch}/",
    "remote_path": "./{module_name}/v{version}/{configuration}/",
    "package_name": "{module_name}-v{version}-{node_abi}-{platform}-{arch}.tar.gz",
    "host": "https://nodeaddons.s3-us-west-2.amazonaws.com"
  },
  "scripts": {
    "preinstall": "npm install node-pre-gyp",
    "install": "node-pre-gyp install --fallback-to-build"
  }
}
{% endhighlight %}

## Testing locally
Explain the "example", link locally.  Test without packaging (always builds).

## Packaging and Publishing
Discuss S3, and also explain that it's not absolutely necessary either.
Explain that at this point, you could publish the "addon" to npm's registry 

## Closing
Plug the book :)
