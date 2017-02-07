---
layout: post
sidebar : true
blog: true
newsletter: true
post_title:  "Cross-platform addons with node-pre-gyp"
title:  "Cross-platform addons with node-pre-gyp"
date:   2017-01-21 15:40:56
permalink: /cross-platform-addons-with-node-pre-gyp/
name: cross-platform-addons-with-node-pre-gyp
disqus_id : nodeaddons-cross-platform-addons-with-node-pre-gyp
disqus_shortname: scottfrees
---
Node.js applications (including electron apps) are deployed everywhere, including end-user machines without a full development stack.  Usually this is no problem - but things get complicated with native addons are used.  

Native C++ addons are distributed as modules, but `npm` *builds them* on the target machine during the install using `node-gyp`.  `node-gyp` isn't a compiler, it's a build system that adapts to the platform it's on - on Windows it will use Visual Studio and on Linux/macOS it will use g++/clang.  When installing an addon on a machine *without* a compiler, we are out of luck.
<!--more-->

Enter `node-pre-gyp`, a convenient tool that lets you deliver pre-built native addons as binary payloads - targeted to the user's platform.  Sounds magical, and in some respects it is!  In this post, I'm going to walk you through how `node-pre-gyp` works.  I'll create an addon that uses platform (OS) specific API's for Windows, macOS, and Linux.  You'll learn how to package a platform-dependent addon into a module that can be deployed on end user machines as binaries.  
<!--more-->

## What is `node-pre-gyp`?
Probably the first thing to know about `node-pre-gyp` is that it is simply a tool to automate what would normally be a tedious manual set of steps to properly download a pre-built binary.  `node-pre-gyp` is not a compiler, and it's not a package distribution / repository tool.  It just makes doing the following easier:

### During build (development)
1. Automatically names built addon executables based on the current platform (OS), architecture (i.e. x64), and Node.js version.
2. Packages the executable into zipped payloads.
3. Optionally automatically publishes the payload to an Amazon S3 bucket.  You can also manually upload them elsewhere.

### During install on an end-user's machine
1.  Automatically detects and computes a filename for a package corresponding to the end-user's platform/architecture/version.
2.  If found, downloads and unpacks the binary addon from the remote host,
3.  If not found, falls back to utilizing the build system on the end-user's machine.

## Example application - time
Let's get started with an example.  One of the most common reasons devs turn to native addons is to use an operating system's native SDK to access features (webcam, system status, etc.).  To demonstrate this type of scenario, I'm writing an addon that contains a single function that returns the current time.  

*Yes... I know Node.js already has such a provision... and before you point at that C++'s `chrono` brought cross-platform high-resolution timers to C++ years ago, I know :)  I'm writing these addons using the underlying OS API's available in Windows, macOS, and Linux not because it's a smart thing to do - but because it works as a simple, clear example of using native OS calls in an addon.*

Before getting started with the addon project setup, let's first create three platform-specific C++ source files, each containing a `native_now` function which returns the current tick time using OS calls.  The eventual build files will select the appropriate one for the intended platform (see below).  

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

If you want to jam this all into one file and use #if preprocessor conditions to detect the OS, that's fine too - but I tend to like using `node-gyp` conditional inclusion to keep the code a bit easier to read.

Finally, I'm going to create a header file that I'll use to drag this code into my addon - `native-rt.h`

{% highlight cpp %}
// native-rt.h
// to be included by the addon code
double native_now();
{% endhighlight %} 


*Thanks to [Nadeau Software](http://nadeausoftware.com/articles/2012/04/c_c_tip_how_measure_elapsed_real_time_benchmarking) for the code that I adapted to make this example!*

### Example directory structure
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
Now let's create the addon code.  Inside `/addon`, we'll create a `native-rt.cc` file that will use NAN to create a single addon function called "now", as called on lines 2 and 5 of the example program above.

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

Create a package.json file in the addon director to define your module.

{% highlight json %}
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

Here's addon/binding.gyp

{% highlight json %}
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

The operative word in the sentence above is of course "build the addon".  The goal of this post is now to turn this addon into a pre-built executable (or set of executables) that can be deployed to different platforms with the same ease-of-use, without the build step requirement.

## Setting up `node-pre-gyp`
As described above, `node-pre-gyp` is a tool that makes it easy to deploy platform-specific binaries to a host (i.e. an Amazon S3 bucket), and set your addon up to automatically download the correct binary based on where it's being installed.  This means that as long as you pre-build and deploy your native addon for all of your supported platforms (OS, architecture, Node.js version), your end users won't need to build your addon when adding it to their projects.  

### Updating `addon/package.json`
Our first step is to add `node-pre-gyp` as a dependency in our addon's `package.json` file.  Note that this isn't just a build dependency, this needs to get installed on the end-user's machine as well.

Next, we need to drop a new section into this same file to tell `node-pre-gyp` how to name the binaries that it will create.  The new `package.json` is below.

{% highlight json linenos%}
{
  "name": "native_rt",
  "version": "1.0.1",
  "description": "Example for using node-pre-gyp for cross-platform binaries",
  "gypfile": true,
  "main": "./index.js",
  "license": "MIT",
  "dependencies": {
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

Disregard the `host` field in the `binary` section and the `scripts` entry for now, we'll get to them in a little while.  For now the important bits are the parameters in the `module_path`, `remote_path`, and `package_name` values.  These are all automatically determined by `node-pre-gyp` when it build the binary packages to be deployed.  They go as follows:

- `module_path`:  This is where the binary build output will be placed locally, when we do an `npm install` on the addon before deployment.
- `remote_path`:  This is the path that the binary build output will eventually be placed on the remote host - this will be used to fetch the binary when installing on an end-user's machine.
- `package_name`:  The name of the binary build output - both locally and remote.

The variables being used are as follows:
- `configuration`:  Release or Debug - you can pass --debug to set to Debug during build
- `platform`:  Basically boils down to the OS - `darwin`, `linux`, or `win32`.  This is pulled from `process.platform` unless overridden by build flags.  It's unlikely you'll override this, since you want the name to match up with the actual OS you are building the addon on.
- `arch`:  Pulled from `process.arch`, will be set to `x64`, `ia32`, etc.  
- `version`:  The version of your addon - this is derived from the `package.json` file.
- `node_abi`:  This refers to the C++ Application Binary Interface version supported by the version of Node.js the addon is targeting.  This is derived from the Node.js runtime you are currently using.  Addons developed for different Node.js versions may have different ABI numbers, which prevents them from working with Node.js/V8.  This is automatically detected for you.

Now take a look at the `scripts` element we've added.  The `preinstall` entry is just telling `npm` to install `node-pre-gyp` before doing anything else.  This is critical, because from now on, `npm install` won't do the normal action, instead (as specified in the new `install` entry), `node-pre-gyp` will do the installation - both locally on your development machine, and also on the end-user's machine.  The `--fallback-to-build` flag tells `node-pre-gyp` to do the full build if it cannot locate the required binaries the specified remote host.  Until we actually start deploying, this will always be the case, and the addon will build using the end-user (local) compiler as it has done before.

Lastly, but perhaps most importantly, we need to modify how a program `require`ing this addon finds the main entry point.  Recall our original `package.json` had listed it's entry point as `./build/Release/native_rt`.  This made sense - it's where `npm install` and subsequently `node-gyp` puts the binary output of an addon when it's built.  

This no longer holds though, now the binary addon will (hopefully) be found, pre-built, on a remote host.  If it's not found (based on OS, architecture, Node.js version, etc.), only then will it be built locally.  `node-pre-gyp` does all this magic for us, but we need to let it do it's job.  We do that by creating a new entry point - we'll call it `/addon/index.js` - **on line 6 of the package.json above**.

{% highlight javascript linenos %}
// index.js inside the addon project.
var binary = require('node-pre-gyp');
var path = require('path')
var binding_path = binary.find(path.resolve(path.join(__dirname,'./package.json')));
var binding = require(binding_path);

module.exports = binding;
{% endhighlight %}  

This bit of code loads `node-pre-gyp`, shows it where the `package.json` file is with all the information about the binary files we'll build, and then attaches the loaded binary to the exports property.  Code `require`ing this addon still works the same way - but now `node-pre-gyp` is locating the binary. 

### Updating `addon/binding.gyp`
The `binary` entry in `package.json` won't work unless you add a new build target to your `binding.gyp` file.  The new target is responsible for taking the normal addon output from `npm install` and copying it out to the `module_path` location as specified in `package.json`.

Check out the `binding.gyp` file below, we've added a new `action_after_build` target that copies the primary build output to where `node-pre-gyp` expects it to be.  

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
    }, 
    {
      "target_name": "action_after_build",
      "type": "none",
      "dependencies": [ "<(module_name)" ],
      "copies": [
        {
          "files": [ "<(PRODUCT_DIR)/<(module_name).node" ],
          "destination": "<(module_path)"
        }
      ]
    }
  ]
}
{% endhighlight %} 

Your final file structure should look like this, with the only change being the new index.js found in the `addon` directory.

{% highlight plaintext %}
 addon/
   |---- native-rt.h
   |---- native-rt_win.cc
   |---- native-rt_linux.cc
   |---- native-rt_mac.cc
   |---- native-rt.cc
   |---- index.js
   |---- binding.gyp
   |---- package.json
 example/
   |---- index.js
   |---- package.json

{% endhighlight %}  

## Testing locally
Before moving forward, it's a good idea to make sure that everything still works, even without the binary being deployed.  Go into the `example` directory and clean out `node_modules`.  Do an `npm install` again, and you should see some new messages printing to the screen...

{% highlight shell %}
$ ~/example rm -r node_modules
$ ~/example npm install
... observe printouts, you should see some node-pre-gyp related messages
$ ~/example node index
1.0043037899886258
{% endhighlight %}

In the printout, you should see the `node-pre-gyp install --fallback-to-build` command being executed.  Since we haven't deployed a binary yet, you'll see messages like `node-pre-gyp ERR! Tried to download(403)` and `node-pre-gyp ERR! Pre-built binaries not found`.  You'll also notice that after those messages, `node-pre-gyp` falls back to doing the normal build.

If all goes well, you can again run the addon with `node index.js` and you'll get a similar output as before.

## Packaging and Publishing
Now it's time to deploy binaries.  Your first decision is what sort of host you want to use.  You have two basic choices - an Amazon S3 bucket, or *anything* else.  The "anything else" option basically just used any hosted endpoint.  Github is a popular choice, but really any web host is fine (etc. https://myspecialsite.com) - **it just needs to support https**.  The advantage of using a custom host is that you don't need to worry about the details of setting up S3 buckets, but the disadvantage is that you must manually deploy binaries to the proper remote path location.  There is a [module to automate](https://www.npmjs.com/package/node-pre-gyp-github) a lot of the process when using github.

The advantage of using S3 buckets is that `node-pre-gyp` can handle the deployment (publishing) process for you, **entirely**.  You do need to have an Amazon AWS account, and you do need to properly setup an S3 bucket and user (with create/write permissions) however - and if you've never done this, expect a bit of heartburn...

I'm going to use the S3 bucket option for this tutorial.  If you elect to use something else, just know that you need to manually upload built binaries to URLs matching those defined in your addon's `package.json` binary entry.

### Setting up AWS
Your first step is to create an S3 bucket, with a few permissions set on a user so `node-pre-gyp` can delete / add binary builds on the bucket, as well as list and retrieve files.  `node-pre-gyp` has some instructions on doing this [here](https://www.npmjs.com/package/node-pre-gyp#1-create-an-s3-bucket).  The basic steps are as follows:

#### Step 1 - Configure AWS S3 Bucket
 Login to your AWS console and create a new S3 bucket (for this tutorial, I've named mine "nodeaddons").  You can keep all the default properties set.

#### Step 2 - Configure AWS Policy
Login to IAM, and create a new policy.  Ensure all the necessary permissions are set, and be sure to set the resource to contain your new S3 bucket.

{% highlight json %}
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "Stmt1394587197000",
            "Effect": "Allow",
            "Action": [
                "s3:DeleteObject",
                "s3:GetObject",
                "s3:GetObjectAcl",
                "s3:ListBucket",
                "s3:PutObject",
                "s3:PutObjectAcl"
            ],
            "Resource": [
                "arn:aws:s3:::nodeaddons/"
            ]
        }
    ]
}
{% endhighlight %} 

#### Step 3 - Configure AWS User
Now attach the policy to either an existing or new user account.  You'll also need to generate and Access key for this user, which will be used by `node-pre-gyp` to do the deployment.  I recommend creating a file that contains your access key id and secret key at this point, and storing it somewhere on your machine (NOT in your source repository... don't ever commit this to git!).  

{% highlight json %}
{
    "accessKeyId": "NOTACTUALLYMYKEY",
    "secretAccessKey": "REALLYNOTMYSECRETKEY"
}
{% endhighlight %} 

I named this file `node-pre-gyp.config` and stored it in the root directory of the project.  I added it to my `.gitignore` to avoid problems.  I'm going to use this file in the next step, although you could also add these parameters to your environment variables - more information is found [here](https://www.npmjs.com/package/node-pre-gyp#3-configure-aws-credentials).

#### Step 4 - Install AWS SDK 
In order to do the publishing, we need to add `aws-sdk` to our dependencies.  In `/addon/package.json` add it, or just do `npm install aws-sdk --save`.  

Now, we can publish by using `node-pre-gyp` directly.  If you installed `node-pre-gyp` globally, you can use it by typing `node-pre-gyp` at the command line.  We didn't do that in this tutorial though, so I'm going to use it's local install.  Note I'm also adding the `node-pre-gyp.config` as a command line option so my AWS credentials are available to `node-pre-gyp`.

Before publishing, review the `addon/package.json` file - recall we skipped over the `host` entry in the `binary` section.  That host value is the root URL for your Amazon S3 bucket.

{% highlight json %}

"binary" : {
    "module_name": "native_rt",
    "module_path": "./lib/binding/{configuration}/{node_abi}-{platform}-{arch}/",
    "remote_path": "./{module_name}/v{version}/{configuration}/",
    "package_name": "{module_name}-v{version}-{node_abi}-{platform}-{arch}.tar.gz",
    "host": "https://nodeaddons.s3-us-west-2.amazonaws.com"
  },
{% endhighlight %} 

Make sure that host name matches up with **YOUR S3 bucket URL**.  Pay special attention to the S3 region, it needs to match up with the endpoint specified in your bucket's properties, which can be accessed using the Amazon AWS console.

Now publish like this:

{% highlight shell %}
$ ~/addon npm install 
$ ~/addon ./node_modules/.bin/node-pre-gyp package publish --config ../node-pre-gyp.config
{% endhighlight %}

The `npm install` builds the package, the `node-pre-gyp` command packages (makes the zip file) and publishes it to S3 - you should see a message at the end giving you the URL where the package was published.  Running this on a Mac gave me the following URL - [https://nodeaddons.s3.amazonaws.com/native_rt/v1.0.1/Release/native_rt-v1.0.1-node-v48-darwin-x64.tar.gz](https://nodeaddons.s3.amazonaws.com/native_rt/v1.0.1/Release/native_rt-v1.0.1-node-v48-darwin-x64.tar.gz).

Test this URL out (your URL, not mine...).  You should be able to download the zip file.

### Testing the deployment
As a first quick test, go ahead and remove all the following directories we've created in this project that include build artifacts:

{% highlight shell %}
$ ~/addon rm -r build
$ ~/addon rm -r lib
$ ~/addon rm -r node_modules
$ ~/addon cd ../example
$ ~/example rm -r node_modules
{% endhighlight %}

Now, in the `example` directory, do a fresh `npm install`.  Notice the output - it won't contain anything related to building.  You'll see a message saying something along the lines of "Success: ... package installed via remote".  `node-pre-gyp` automatically downloads the binary!  Note that your AWS credentials aren't needed for this, since you S3 bucket doesn't require authentication just to download the file.

{% highlight shell %}
$ ~/example npm install
$ ~/example node index.js
1.0020250650122762
{% endhighlight %}

## Publishing to multiple platforms
Now comes the "fun" part.  In the steps above, we've only published binaries for the specific setup we have on our development machine.  The next step is to re-run the `npm install` and `./node_modules/.bin/node-pre-gyp package publish --config ../node-pre-gyp.config` on every configuration you intend to support.  This means you'll need to do this on Linux, macOS, and Windows.  It also means you likely need to do this with various versions of Node.js.  You may even go as far as different CPU architectures.  This is obviously a tedious process, and typically it's automated - `node-pre-gyp` has documentation specifically discussing using Appveyor and Travis.  This is outside the scope of this article, but is the next logical step.

## Summary
It takes some work to setup, and it requires you to build your addon on all your intended platforms, but `node-pre=gyp` gives you the ability to distributed npm packages with native addons anywhere - including end-user machines without the necessary build tools for C++.  The code we've developed in the `/addon` directory is 100% ready to be published to an npm repository, and it can be `require`ed by any Node.js program.

You can find the full source code for this in the [nodecpp-demo](https://github.com/freezer333/nodecpp-demo) repository, this example is found in the `prebuilt` directory.

Now of course it's time to create a useful addon, worth distributing!  Check out some of my eariler [posts](/c-processing-from-node-js/) and my [book](/book/) for more help on that part.