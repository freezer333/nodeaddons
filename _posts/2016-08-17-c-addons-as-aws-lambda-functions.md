---
layout: post
sidebar : true
blog: true
title:  "C++ Addons as AWS Lambda functions"
date:   2016-08-17 15:40:56
permalink: /c-addons-as-aws-lambda-functions/
name: c-addons-as-aws-lambda-functions
disqus_id : silvrback-scottfrees-26666
disqus_shortname: scottfrees
---
In this post I'm going to walk you through creating and deploying a Node.js AWS Lambda function that uses a native C++ addon.  As you'll see, the process isn't much different than creating normal AWS Lambda functions with Node.js - you'll just need to get your development environment to match the requirements for AWS.  
<!--more-->
## What is AWS Lambda?
Quoting directly from AWS, 

&gt; AWS Lambda is a compute service where you can upload your code to AWS Lambda and the service can run the code on your behalf using AWS infrastructure. After you upload your code and create what we call a Lambda function, AWS Lambda takes care of provisioning and managing the servers that you use to run the  code. You can use AWS Lambda as follows:

&gt; - As an event-driven compute service where AWS Lambda runs your code in response to events, such as changes to data in an Amazon S3 bucket or an Amazon DynamoDB table.
&gt; - As a compute service to run your code in response to HTTP requests using Amazon API Gateway or API calls made using AWS SDKs.

Lambda is a cool platform, but it only supports a few languages - Java, Node.js, and Python.  So... what if you want to expose some C++ code using Lambda?  Well, you can certainly link Java to C++ libraries, and Python can do the same.  In the Node.js world, the way we typically integrate C++ with JavaScript is through addons.  Node.js C++ addons are compiled (native) Node.js modules which are directly callable from JavaScript as any other Node.js module.  

Node.js addons is a big topic - if you are new to addons, check out my [intro series](/c-processing-from-node-js) of posts.   I also have a [series](/getting-your-c-to-the-web-with-node-js) specifically on integrating legacy C++ into Node.js web apps - which discusses some alternatives to addons.  If you are looking for a full treatment of integrating C++ and Node.js, check out my book - [C++ and Node.js Integration](/blog/).

## Addons on Lambda
So, why is addon development for AWS Lambda different than the typical scenario?  The biggest issue is that AWS Lambda isn't going to invoke `node-gyp` or any other build tool you need might need before launching your function - you are responsible for creating the full binary package.  This means that at the very least, you'll need to build your addon on Linux before deploying to Lambda, and you'll possibly need to go as far as building it on Amazon Linux itself if you have dependencies.  There are also some quirks to getting the deployment to Lambda just right - and I'll cover those too here.

This post isn't about building sophisticated Lambda-based applications - and its only going to cover some basic deployment techniques.  Amazon has tons of documentation about Lambda, best practices, etc. - so I'm just sticking to the basics to demonstrate addons.

I'm going to build a C++ addon with a function that accepts three numbers and returns the average.  *I know... this is clearly something only C++ can possibly do...*  We'll expose this function as an AWS Lambda function, and test it out with the AWS CLI. 

## Setting up your development environment
There's a reason Java became famous when it's "[write once, run anywhere](https://en.wikipedia.org/wiki/Write_once,_run_anywhere)" slogan was introduced - distributing compiled native code is fraught with problems.  Java didn't really solve all that back then (... write once, debug everywhere...), but we've come a long way since then.  Normally we are blissfully unaware of platform-specific deployments when writing Node.js - the JavaScript we write is platform independent.  Even Node.js programs that depend on native addons can typically be installed/deployed on different systems without much developer intervention - thanks to `npm` and `node-gyp`.  

Much of this convenience is lost, however, when dealing with Amazon Lambda - we truly must **pre-build** our Node.js program (and it's dependencies).  If we are using a native addon, then this means when building our deployment packages, **we must be on the same architecture and platform** as Lambda (64-bit, Linux), and we must specifically use the **same Node.js runtime** used with Lambda.

### Requirement 1:  Linux
We can certainly develop / test / debug Lambda functions with addons on OS X or Windows, but the bottom line is that when we deploy to Lambda, we are going to deploy a zip file with our *entire* Node.js module - including it's dependencies (`node_modules`).  All native code within the deployed zip must run on Amazon's Lambda infrastructure.  Therefore, at a minimum, we must build our addon on Linux to deploy.  Note that in this post, I'm not using any shared / OS libraries - I'm using a C++ addon that is completely standalone.  As I'll describe later - if you need to use external libraries you may need to go further than simply using *any* Linux distribution.

I'm going to do all my development for this post on Linux Mint.  

### Requirement 2:  64-bit
This probably should have been Requirement 1... For much the same reasons as above - you'll need to create your deployable zip file with binaries targeting x64 architectures... so an old 32-bit Linux running on a VM won't cut it.

### Requirement 3:  Node.js Version 4.3
At the time of this writing, AWS Lambda supports Node.js 0.10 and 4.3,  You should absolutely pick 4.3.  In the future these versions could change - so edit accordingly.  I like to use [`nvm`](https://github.com/creationix/nvm) to install and switch between Node.js versions.  If you don't already have it, go ahead and install:


```
&gt; curl https://raw.githubusercontent.com/creationix/nvm/master/install.sh | bash
&gt; source ~/.profile
```

Now install Node.js 4.3 and install `node-gyp` while you are at it.

```
&gt; nvm install 4.3
&gt; npm install -g node-gyp
```

### Requirement 4:  C++ build tools (C++11)
When you are doing Node.js addon development for Node.js v4+, you must use a compiler that supports C++ 11.  Recent versions of Visual Studio (Windows) and XCode (Mac OS X) all will due - but since we need to build on Linux, we just need to make sure we install g++ 4.7 or above.  Here's how you'd install g++ 4.9 on Mint/Ubuntu:

```
&gt; sudo add-apt-repository ppa:ubuntu-toolchain-r/test
&gt; sudo apt-get update
&gt; sudo apt-get install g++-4.9
&gt; sudo update-alternatives --install /usr/bin/g++ g++ /usr/bin/g++-4.9 20
```

If you already have an older version of g++ on your machine, you'll need to make sure you set things up so 4.9 is now used (see [here](http://www.techerina.com/2015/04/installing-upgrading-gcc-in-ubuntu-linuxmint-machine.html) for some more details).


## Creating the addon (locally)
We're actually going to actually create two Node.js projects.  One will be the C++ addon itself, which won't have any AWS Lambda artifacts at all - it's just a straight up addon.  The second will in fact be the Lambda function - it will be a Node.js module that imports the addon, and exposes the Lambda handler.  If you want to follow along on your own machine, all the source code is [here](https://github.com/freezer333/nodecpp-demo) - this particular example in the `lambda-cpp` folder.  

Lets start first with the addon.

```
&gt; mkdir lambda-cpp
&gt; mkdir lambda-cpp/addon
&gt; cd lambda-cpp/addon
```

To create the addon, we need three files - our C++ source, a `package.json` to tell Node.js how to deal with this addon, and a `binding.gyp` file to handle the build process.  We're creating a super simple addon here - I'll skip most of the discussion.  Again, if you are looking for more details on addon development in general - check out my [other posts](/c-processing-from-node-js) and [my book](/book/).

Let's start with the easiest - `binding.gyp`

```js
{
  "targets": [
    {
      "target_name": "average",
      "sources": [ "average.cpp" ]
    }
  ]
}
```
This is probably the most basic `binding.gyp` file you could have - we simply specify the target name and the source files to compile.  The sky's the limit (almost) in terms of what else you can do in terms of compiler options, external files/libraries, etc.  Just remember that anything you are linking too must be [statically compiled](https://aws.amazon.com/blogs/compute/nodejs-packages-in-lambda/) into the resulting executable, and be built for x64 Linux.

Now let's create `package.json`, which must be setup so it defines the entry point of this addon to the binary target.

```js
{
  "name": "average",
  "version": "1.0.0",
  "main": "./build/Release/average",
  "gypfile": true,
  "author": "Scott Frees &lt;scott.frees@gmail.com&gt; (http://scottfrees.com/)",
  "license": "ISC"  
}
```

The key thing here is the `main` property, it's telling Node.js that the eventual native executable is the entry point to this module - which will be loaded whenever anyone does a `require('average')`.

Now the source code...  Lets open up `average.cpp` and create a simple addon function that returns the average of any/all numeric parameters it is sent (we don't need to limit the code to just three!).

```cpp
#include &lt;node.h&gt;

using namespace v8;

void Average(const FunctionCallbackInfo&lt;Value&gt;&amp; args) {
    Isolate * isolate = args.GetIsolate();
    double sum = 0;
    int count = 0;
    
    for (int i = 0; i &lt; args.Length(); i++){
    	if ( args[i]-&gt;IsNumber()) {
    		sum += args[i]-&gt;NumberValue();
    		count++;
    	}
    }
    
    Local&lt;Number&gt; retval = Number::New(isolate, sum / count);
    args.GetReturnValue().Set(retval);
}


void init(Local&lt;Object&gt; exports) {
  NODE_SET_METHOD(exports, "average", Average);
}

NODE_MODULE(average, init)
```

Again, if you aren't familiar with using the V8 (or NAN) API to build addons, please check out my [other posts](/c-processing-from-node-js) and [my book](/book/).  In short, the `NODE_MODULE` macro at the bottom defines `init` as the function V8 should call when this module is loaded.  `init` adds a new function to the `exports` object for the module - associating `Average` with what now will be a callable function `average`.

We can build this by issuing a `node-gyp configure build` command.  If you have everything setup correctly, you should see `gyp info ok` at the bottom of the output.    

As a quick test, let's create `test.js` right along side this all - and give our addon a few calls:

```js
// test.js
const addon = require('./build/Release/average');

console.log(addon.average(1, 2, 3, 4));
console.log(addon.average(1, "hello", "world", 42));
```

Run this with `node test.js` and you should see `2.5` and `21.5` print out.  Note the "hello" and "world" parameters haven't messed up the calculation since the addon inspects the types of the parameters before using them to do the averaging.

*You should delete `test.js` now - we don't want it to be part of the addon, and we don't want to deploy it to AWS Lambda*

### Creating the Lambda Function
Now let's actually create the AWS Lambda handler.   As you (probably) already know, all AWS Lambda functions must expose a *handler* which gets called whenever an event is triggered.  This handler receives the event (which can be associated with an S3 put, DynamoDB Update, etc.) when it's invoked.  Events are just standard JS objects, and for now we'll use a simple test event that looks like this:

```js
{
    op1:  4,
    op2:  15, 
    op3:  2
}
```

While I could do this right in the `addon` directory (requiring the addon using the relative path to the `average.node` binary), I prefer to create this as a distinct Node.js program - and pull in the local addon as an `npm` dependency. Let's create a new directory along side the `lambda-cpp/addon` directory called `lambda-cpp/lambda`.

```
&gt; cd ..
&gt; mkdir lambda
&gt; cd lambda
```

Here's the handler code - which you should put in `index.js`:

```js
exports.averageHandler = function(event, context, callback) {
   const addon = require('average');
   var result = addon.average(event.op1, event.op2, event.op3)
   callback(null, result);
}
```

Note that we've required `average` as if it were an external dependency.  Let's create a `package.json` file to target the local addon:

```json
{
  "name": "lambda-demo",
  "version": "1.0.0",
  "main": "index.js",
  "author": "Scott Frees &lt;scott.frees@gmail.com&gt; (http://scottfrees.com/)",
  "license": "ISC", 
  "dependencies": {
    "average": "file:../addon"
  }
}
```

When you do an `npm install`, `npm` will pull your local addon and make a copy of it inside `/node_modules`, and invoke `node-gyp` to build it.  Your directory structure should be as follows:

```
/lambda-cpp
 -- /addon
    -- average.cpp
    -- binding.gyp
    -- package.json
 -- /lambda
    -- index.js
    -- package.json
    -- node_modules/
      -- average/  (contains the binary addon)
```

## Testing locally
Now that `index.js` is exporting a Lambda handler, we could upload it directly to Amazon Lambda - but first you might want to test it out locally to make sure things are working well.  There's a nice module - `lambda-local` - that can help us with this.

```
npm install -g lambda-local
```

Once installed, we can invoke our Lambda function by specifying the handler name (`averageHandler`) and a sample event.  Lets create an event object and put it in `sample.js`:

```js
module.exports = {
    op1:  4,
    op2:  15, 
    op3:  2
};
```

Now we can execute our lambda locally with the following command:

```
&gt; lambda-local -l index.js -h averageHandler -e sample.js
Logs
------
START RequestId: 33711c24-01b6-fb59-803d-b96070ccdda5
END


Message
------
7
```

As expected, our resulting message is 7, the average of 4, 15, 2.

## Deploying with AWS CLI
There are two ways to deploy the Lambda - through the AWS web interface, and through the CLI.  I'm going to use the CLI here, I think it's more general purpose - but everything I'm going to do in this post specifically can also be done through the web UI.

The first step is to get an AWS account if you don't already have one, and to create an Administrator User.  Full instructions for this can be found in [Amazon's documentation](http://docs.aws.amazon.com/lambda/latest/dg/setup.html).  Make sure you [add the `AWSLambdaBasicExecutionRole`](http://docs.aws.amazon.com/lambda/latest/dg/with-userapp-walkthrough-custom-events-create-iam-role.html) role to the user account.  *You'll see when we deploy with the CLI, I'll specify the role  for the Lambda - you can name the role anything you want (I'm using `lambda_execute`), but you must create a role with Lambda execution permissions.  See more [here](https://docs.aws.amazon.com/lambda/latest/dg/intro-permission-model.html#lambda-intro-execution-role).

Assuming you have an Administrator AWS user account, you need to grab an Access Key to configure the AWS CLI with.  You do this through the IAM console.  You can download your access key credentials as a csv file using the instructions in the AWS docs [here](http://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-set-up.html).

Once you have your access keys, you need to install the CLI.  There are a few ways to do this, and it requires Python to be installed on your machine.  The most straight foward install (IMO) is the Bundled Installer:

```
&gt; curl "https://s3.amazonaws.com/aws-cli/awscli-bundle.zip" -o "awscli-bundle.zip"
&gt; unzip awscli-bundle.zip
&gt; sudo ./awscli-bundle/install -i /usr/local/aws -b /usr/local/bin/aws
```

Next, you'll need to configure the CLI.  Type `aws configure` and enter your access key and secret key downloaded a moment ago.  You can also choose a default region and output format.  You probably should attach a profile to this configuration (you'll need it later) using the --profile &lt;profile name&gt; argument.

```
&gt; aws configure --profile lambdaProfile
AWS Access Key ID [None]: XXXXXXXXX
AWS Secret Access Key [None]: XXXXXXXXXXXXXXXXXXXX
Default region name [None]: us-west-2
Default output format [None]: 

```
You can verify that you've set this all up correctly by listing your Lambda functions:

```
&gt; aws lambda list-functions
{
    "Functions": []
}
```

Of course, if you just created this account, you won't have any functions - but you shouldn't see any errors at this point at least.

### Packaging the Lambda Function &amp; Addon
The most crucial (and judging by questions online, often overlooked) step in this process is now to ensure the *entire* module makes its way into a zip file correctly - which we can deploy to Lambda.  Here's the most important things to remember:

1. The `index.js` file must be at the top (root) of the zip file's hierarchy.  You should not zip the `/lambda-addon/lambda` folder itself - just it's contents.  In other words, if you uzip the zip file you create, index.js should NOT be in a directory.
2. The `node_modules` directory - *and all of its contents* must be in the zip file too.  
3. You need to build the addon and zip it on the right platform (see requirements above... Linux, x64, etc.).

Inside the directory containing `index.js`, zip up all the files that should be deployed.  I'll put the zip in the `lambda-addon` parent directory.

```
&gt; zip -r ../average.zip node_modules/ average.cpp index.js binding.gyp package.json 
```
**Note the -r ** - you need to entire `node_modules` directory.  Verify the package with `less`.  You should see something along the lines of this:

```
less ../average.zip

Archive:  ../average.zip
 Length   Method    Size  Cmpr    Date    Time   CRC-32   Name
--------  ------  ------- ---- ---------- ----- --------  ----
       0  Stored        0   0% 2016-08-17 19:02 00000000  node_modules/
       0  Stored        0   0% 2016-08-17 19:02 00000000  node_modules/average/
       1  Stored        1   0% 2016-08-17 17:39 6abf4a82  node_modules/average/output.txt
     478  Defl:N      285  40% 2016-08-17 19:02 e1d45ac4  node_modules/average/package.json
     102  Defl:N       70  31% 2016-08-17 15:03 1f1fa0b3  node_modules/average/binding.gyp
       0  Stored        0   0% 2016-08-17 19:02 00000000  node_modules/average/build/
     115  Defl:N      110   4% 2016-08-17 19:02 c79d3594  node_modules/average/build/binding.Makefile
    3243  Defl:N      990  70% 2016-08-17 19:02 d3905d6b  node_modules/average/build/average.target.mk
    3805  Defl:N     1294  66% 2016-08-17 19:02 654f090c  node_modules/average/build/config.gypi
       0  Stored        0   0% 2016-08-17 19:02 00000000  node_modules/average/build/Release/
       0  Stored        0   0% 2016-08-17 19:02 00000000  node_modules/average/build/Release/.deps/
       0  Stored        0   0% 2016-08-17 19:02 00000000  node_modules/average/build/Release/.deps/Release/
     125  Defl:N       67  46% 2016-08-17 19:02 daf7c95b  node_modules/average/build/Release/.deps/Release/average.node.d
       0  Stored        0   0% 2016-08-17 19:02 00000000  node_modules/average/build/Release/.deps/Release/obj.target/
       0  Stored        0   0% 2016-08-17 19:02 00000000  node_modules/average/build/Release/.deps/Release/obj.target/average/
    1213  Defl:N      386  68% 2016-08-17 19:02 b5e711d9  node_modules/average/build/Release/.deps/Release/obj.target/average/average.o.d
     208  Defl:N      118  43% 2016-08-17 19:02 c8a1d92a  node_modules/average/build/Release/.deps/Release/obj.target/average.node.d
   13416  Defl:N     3279  76% 2016-08-17 19:02 d18dc3d5  node_modules/average/build/Release/average.node
       0  Stored        0   0% 2016-08-17 19:02 00000000  node_modules/average/build/Release/obj.target/
       0  Stored        0   0% 2016-08-17 19:02 00000000  node_modules/average/build/Release/obj.target/average/
    5080  Defl:N     1587  69% 2016-08-17 19:02 6aae9857  node_modules/average/build/Release/obj.target/average/average.o
   13416  Defl:N     3279  76% 2016-08-17 19:02 d18dc3d5  node_modules/average/build/Release/obj.target/average.node
   12824  Defl:N     4759  63% 2016-08-17 19:02 f8435fef  node_modules/average/build/Makefile
     554  Defl:N      331  40% 2016-08-17 15:38 18255a6e  node_modules/average/average.cpp
     237  Defl:N      141  41% 2016-08-17 19:02 7942bb01  index.js
     224  Defl:N      159  29% 2016-08-17 18:53 d3d59efb  package.json
--------          -------  ---                            -------
   55041            16856  69%                            26 files

```
(type 'q' to exit less)

If you don't see contents of the `node_modules` directory inside the zip, or if all of these files have an extra common parent directory - read the text above more carefully!

### Uploading to AWS Lambda
Now we can can create the Lambda function using the `lambda create-function` command.  Make sure you update the region accordingly for your setup:

```
&gt; aws lambda create-function \
--region us-west-2 \
--function-name average \
--zip-file fileb://../average.zip \
--handler index.averageHandler \
--runtime nodejs4.3 \
--role arn:aws:iam::729041145942:role/lambda_execute
```

Most of the above is self-explanatory - but if you are totally unfamiliar with Lambda the "role" value can be a bit mystifying.  As described above, in order to work with Lambda you need to create a role using IAM that has (at least) the `AWSLambdaBasicExecutionRole` permissions.  You can get the "arn:" string for that role using the IAM web interface (click on the Role itself).

If all goes well, you should receive a JSON response with some additional info about the newly deployed function (for example, `FunctionArn`).

### Testing with AWS CLI
Now that we've deployed the function, let's test it out again - this time using the CLI.  Invoke the function, specifying the same event as last time (which is called `payload` by the CLI).

```
&gt; aws lambda invoke \
--invocation-type RequestResponse \
--function-name average \
--region us-west-2 \
--log-type Tail \
--payload '{"op1":4, "op2":15, "op3":2}' \
--profile lambdaProfile \
output.txt

```

You'll get something that looks like this:

```
{
    "LogResult": "U1RBUlQgUmVxdWVzdElkOiAxM2UxNTk4ZC02NGMxLTExZTYtODQ0Ny0wZDZjMjJjMTRhZWYgVmVyc2lvbjogJExBVEVTVApFTkQgUmVxdWVzdElkOiAxM2UxNTk4ZC02NGMxLTExZTYtODQ0Ny0wZDZjMjJjMTRhZWYKUkVQT1JUIFJlcXVlc3RJZDogMTNlMTU5OGQtNjRjMS0xMWU2LTg0NDctMGQ2YzIyYzE0YWVmCUR1cmF0aW9uOiAwLjUxIG1zCUJpbGxlZCBEdXJhdGlvbjogMTAwIG1zIAlNZW1vcnkgU2l6ZTogMTI4IE1CCU1heCBNZW1vcnkgVXNlZDogMzUgTUIJCg==", 
    "StatusCode": 200
}
```

Not helpful - but easily decoded.  The LogResult is encoded in base64, so you can do the following:

``` 
&gt; echo U1RBUlQgUmVxdWVzdElkOiAxM2UxNTk4ZC02NGMxLTExZTYtODQ0Ny0wZDZjMjJjMTRhZWYgVmVyc2lvbjogJExBVEVTVApFTkQgUmVxdWVzdElkOiAxM2UxNTk4ZC02NGMxLTExZTYtODQ0Ny0wZDZjMjJjMTRhZWYKUkVQT1JUIFJlcXVlc3RJZDogMTNlMTU5OGQtNjRjMS0xMWU2LTg0NDctMGQ2YzIyYzE0YWVmCUR1cmF0aW9uOiAwLjUxIG1zCUJpbGxlZCBEdXJhdGlvbjogMTAwIG1zIAlNZW1vcnkgU2l6ZTogMTI4IE1CCU1heCBNZW1vcnkgVXNlZDogMzUgTUIJCg== |  base64 --decode 

START RequestId: 13e1598d-64c1-11e6-8447-0d6c22c14aef Version: $LATEST
END RequestId: 13e1598d-64c1-11e6-8447-0d6c22c14aef
REPORT RequestId: 13e1598d-64c1-11e6-8447-0d6c22c14aef	Duration: 0.51 ms	Billed Duration: 100 ms 	Memory Size: 128 MB	Max Memory Used: 35 MB	
```
While a bit more readable - this output really isn't telling you much - sine our Lambda didn't print anything that would appear in the log file.  If you want to see something a bit more satisfying, you could test the function on the AWS web interface where the input/outputs are more easily seen.  For now, go ahead and add some printouts to your `index.js` function, repackage the zip file, redeploy, and then invoke your function again.

```js
exports.averageHandler = function(event, context, callback) {
   const addon = require('./build/Release/average');
   console.log(event);
   var result = addon.average(event.op1, event.op2, event.op3)
   console.log(result);
   callback(null, result);
}
```

After decoding the output, you'll see something like this:

```
START RequestId: 1081efc9-64c3-11e6-ac21-43355c8afb1e Version: $LATEST
2016-08-17T21:39:24.013Z	1081efc9-64c3-11e6-ac21-43355c8afb1e	{ op1: 4, op2: 15, op3: 2 }
2016-08-17T21:39:24.013Z	1081efc9-64c3-11e6-ac21-43355c8afb1e	7
END RequestId: 1081efc9-64c3-11e6-ac21-43355c8afb1e
REPORT RequestId: 1081efc9-64c3-11e6-ac21-43355c8afb1e	Duration: 1.75 ms	Billed Duration: 100 ms 	Memory Size: 128 MB	Max Memory Used: 17 MB	
```

## Tips and next steps
At this point, we have a 100% working AWS Lambda function that invokes a C++ addon.  Of course, we really haven't done anything interesting at all with the Lambda function.  Since our Lambda addon function is geared towards simple computation, a next step might be to hook it up as an Gateway API.  You can take a look Amazon's [Getting Started](http://docs.aws.amazon.com/apigateway/latest/developerguide/getting-started.html) page - specifically the section concerning Calling Lambda Functions.

I hope you've seen now that deploying C++ addons on Lambda isn't too difficult - in fact, as long as you keep in mind the requirements for building the addons on the right platform, it's really just the same process as any other Lambda deployment.  As pointed out earlier, don't forget that if your Addon needs additional third-party libraries, you'll need to ensure they are statically linked to the deployed binary. 

All of the code from this post, plus a whole bunch of other addon examples are in my [nodecpp-demo repository](https://github.com/freezer333/nodecpp-demo).
