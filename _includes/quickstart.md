# Addon Quickstart
Welcome!  This is a quick guide-sheet to get you started with Node.js C++ addons.  It's meant to be super-easy to skim and copy/paste into your code.  If you are looking for more explanation and depth, checkout the <b>blog</b>, the full <b>code tutorial</b>, the <b>book</b>, or <b> the complete video course</b>.

## Project structure
Always create your addon in it's own directory.  If it's going to be reusable, move the directory outside of your main project directory.  

Each reusable addon needs (1) a package.json, (2) a binding.gyp, and (3) C++ source code.  If your addon isn't reusable, and isn't a standalone module, then you can omit package.json 

{% highlight plaintext %}
  my_addon/
    |---- package.json // (not needed if not reusable)
    |---- binding.gyp
    |---- addon_source.cc
{% endhighlight %}  

## package.json
It's almost always easier to set up your addon as an independent module.  When done this way, you can require it from your local project, and a standard `npm install` will handle the entire build process.

Three key things happen in package.json:  
1. Set `gypfile` to`true` so `npm install` builds the addon automatically.  
2. Set the entry point to the executable ("my_addon" below will be whatever you name your addon).
3. Add NAN as dependency (optional, but **highly recommended**!)
  
{% highlight json %}
{
      "name": "my_addon",
      "version": "0.0.0",
      "description": "My reusable C++ addon",
      "gypfile": true,
      "main": "./build/Release/my_addon",
      "license": "MIT",
      "dependencies": {
        "nan": "^2.3.3"
      }
}
{% endhighlight %}  

## bindings.gyp
  The binding file is where you define your build.  If you only have one C++ file, and it's platform independent (doesn't use OS calls), then this is really straightforward. **Make sure you use the right name for your addon** - what you put for "my_addon" needs to match what you put in package.json, and in the C++ code below).
  
  {% highlight json %}
   {
    "targets": [
      {
        "target_name": "my_addon",
        "sources": [ "addon_source.cc" ],
        "include_dirs" : [
          "<!(node -e \"require('nan')\")"
        ]
      }
    ]
  }
  {% endhighlight %}  


If you have platform specific source, you can add them with conditionals.  In addition, if you have compiler flags they can be added as well.  Check out my [intro blog post](/c-processing-from-node-js/) on this for a bit more depth.

{% highlight json %}
   {
      "targets": [
        {
          "target_name": "my_addon",
          "cflags": ["-Wall", "-std=c++11"],
          "sources": [ "platform-independent-source.cc" ],
          "conditions":[
            ["OS=='linux'", {
              "sources": [ "linux_source.cc" ]
              }],
            ["OS=='mac'", {
              "sources": [ "mac_source.cc" ]
            }],
            ["OS=='win'", {
              "sources": [ "windows_source.cc" ]
            }]
          ], 
          "xcode_settings": {
            "OTHER_CFLAGS": [
              "-std=c++11"
            ]
          },
          "msvs_settings": {
            "VCCLCompilerTool": {
              "ExceptionHandling": "1 # /EHsc"
            }
          },
          "include_dirs" : [
            "<!(node -e \"require('nan')\")"
          ]
        }
      ]
    }
  {% endhighlight %}  
  
  If you are feeling adventurous, check out the [gyp](https://gyp.gsrc.io/docs/UserDocumentation.md) documentation for all the options and use cases - however there is some chromium-only information there, so beware.

## Passing data between C++ and JavaScript
The following demonstrates passing several primitive types to C++ from JavaScript, and synchronously returning primitives back to JavaScript.  It uses <a href="https://github.com/nodejs/nan">NAN</a>, which is the recommended approach for addon development.

**You can get the complete code [here](https://github.com/freezer333/nodecpp-demo/tree/master/quickstart).**

### Numeric data
  
  {% highlight cpp %}
  // contents of addon_source.cc

  // This is a basic addon - a binding.gyp file 
  // would need to include this file as it's source.

  #include <nan.h>
  using namespace std;
  using namespace Nan;
  using namespace v8;

  // Accepts 1 number from JavaScript, adds 42 and returns the result.
  NAN_METHOD(PassNumber) {
    Nan::Maybe<double> value = Nan::To<double>(info[0]); 
    Local<Number> retval = Nan::New(value.FromJust() + 42);
    info.GetReturnValue().Set(retval);    
  }

  // Called by the NODE_MODULE macro below, 
  // exposes a pass_number method to JavaScript, which maps to PassNumber 
  // above.
  NAN_MODULE_INIT(Init) {
     Nan::Set(target, New<String>("pass_number").ToLocalChecked(),
        GetFunction(New<FunctionTemplate>(PassNumber)).ToLocalChecked());
  }

  // macro to load the module when require'd
  NODE_MODULE(my_addon, Init)
  {% endhighlight %}  

The following JavaScript program requires the addon above (see the beginning of this guide for setting this up).

  {% highlight javascript %}
  // contents of index.js
  const addon = require("my_addon");
  const assert = require('assert');

  assert(addon.pass_number(100) == 142);
  assert(addon.pass_number(100.5) == 142.5);
  assert(addon.pass_number("100") == 142);
  assert(isNaN(addon.pass_number("no")));
  assert(isNaN(addon.pass_number()));
  assert(isNaN(addon.pass_number(function(){})));
  {% endhighlight %}  
  

Integer data is similar.  You can also do some error checking so undefined can be returned (or something else) instead of just NaN.

  {% highlight cpp %}
  // contents of addon_source.cc

  NAN_METHOD(PassInteger) {
    if ( info.Length() < 1 ) {
        return;
    }
    if ( !info[0]->IsInt32()) {
        return;
    }
    int value = info[0]->IntegerValue();
    Local<Integer> retval = Nan::New(value + 42);
    info.GetReturnValue().Set(retval); 
  }
  {% endhighlight %}  

You'd need to load PassInteger the same way as PassNumber was added to the module.
  
  {% highlight javascript %}
  // contents of index.js
  
  assert(addon.pass_number(100) == 142);
  assert(addon.pass_number(100.5) == 142.5);
  assert(addon.pass_number("100") == 142);
  assert(isNaN(addon.pass_number("no")));
  assert(isNaN(addon.pass_number()));
  assert(isNaN(addon.pass_number(function(){})));
  {% endhighlight %} 

### Boolean data
  {% highlight cpp %}
  // contents of addon_source.cc

  // flips true to false, false to true.
  NAN_METHOD(PassBoolean) {
    if ( info.Length() < 1 ) {
        return;
    }
    if ( !info[0]->IsBoolean()) {
        return;
    }
    bool value = info[0]->BooleanValue();
    Local<Boolean> retval = Nan::New(!value);
    info.GetReturnValue().Set(retval); 
  }
  {% endhighlight %} 
  
  {% highlight javascript %}
  // contents of index.js
  assert(addon.pass_boolean(false));
  assert(addon.pass_boolean(true) == false);
  assert(addon.pass_boolean() == undefined);
  assert(addon.pass_boolean("no") == undefined);
  assert(addon.pass_boolean(1) == undefined);
  {% endhighlight %} 

### String data
String data uses a slightly different syntax...

  {% highlight cpp %}
  // contents of addon_source.cc
  // Reverses the string given
  NAN_METHOD(PassString) {
    if ( info.Length() < 1 ) {
        return;
    }
    if ( !info[0]->IsString()) {
        return;
    }

    // wrap the string with v8's string type
    v8::String::Utf8Value val(info[0]->ToString());
    
    // use it as a standard C++ string
    std::string str (*val);
    std::reverse(str.begin(), str.end());
    
    info.GetReturnValue().Set(Nan::New<String>(str.c_str()).ToLocalChecked()); 
  }
  {% endhighlight %} 

  {% highlight javascript %}
  // contents of index.js

  assert(addon.pass_string("hello") == "olleh");
  assert(addon.pass_string() == undefined);
  assert(addon.pass_string(88) == undefined);
  {% endhighlight %} 

Strings are pretty loosely interpreted.  If you remove the `info[0]->IsString()` check in the C++
    and pass in `undefined`, you'll get `denifednu` back!

### Objects
Objects are a little more complicated.  Here, pass object accepts an object with two properties, X and Y.  It returns a new object with the sum and product.  Most of the code is really dedicated to creating the necessary strings to serve as property names.

**Remember, you can get the complete code [here](https://github.com/freezer333/nodecpp-demo/tree/master/quickstart).**

{% highlight cpp %}
// contents of addon_source.cc
NAN_METHOD(PassObject) {
    if ( info.Length() > 0 ) {
        Local<Object> input = info[0]->ToObject();
        
        // Make property names to access the input object
        Local<String> x_prop = Nan::New<String>("x").ToLocalChecked();
        Local<String> y_prop = Nan::New<String>("y").ToLocalChecked();
        Local<String> sum_prop = Nan::New<String>("sum").ToLocalChecked();
        Local<String> product_prop = Nan::New<String>("product").ToLocalChecked();

        // create the return object
        Local<Object> retval = Nan::New<Object>();
        
        // pull x and y out of the input.  We'll get NaN if these weren't set, 
        // or if x / y aren't able to be converted to numbers.
        double x = Nan::Get(input, x_prop).ToLocalChecked()->NumberValue();
        double y = Nan::Get(input, y_prop).ToLocalChecked()->NumberValue();
      
        // set the properties on the return object
        Nan::Set(retval, sum_prop, Nan::New<Number>(x+y));
        Nan::Set(retval, product_prop, Nan::New<Number>(x*y));
        
        info.GetReturnValue().Set(retval);
    }
}
{% endhighlight %}

{% highlight cpp %}
// index.js
var check = addon.pass_object({x: 5, y: 6});
assert(check)
assert(check.sum == 11);
assert(check.product == 30);

check = addon.pass_object();
assert(!check);

check = addon.pass_object({foo:5, bar:6});
assert(isNaN(check.sum));
assert(isNaN(check.product));

check = addon.pass_object({x:"6", y:"5"});
assert(check.sum == 11);
assert(check.product == 30);
{% endhighlight %}

If you are looking to work with Object Wrapping, where you can deal with C++ objects within JavaScript, take a look at the [Node.js and C++ Integration book](/book/).  The book also has a more complete set of examples of building objects inside V8 using Nan.  For working directly with V8 without NAN, check out this [post](/type-conversions-from-javascript-to-c-in-v8/) - it's worth checking out!

### Arrays
Finally, here's an example of doing some work with arrays.  Like object, arrays are mutable from within the addon - meaning if you change a property/index on the parameter, that change is happening on the actual JavaScript memory.

{% highlight cpp %}
// contents of addon_source.cc
// Increment each value in the array parameter, 
// Return a new array with the squares of the original
// array and a "sum_of_squares" property.
NAN_METHOD(IncrementArray) {
    Local<Array> array = Local<Array>::Cast(info[0]);
    
    Local<String> ss_prop = Nan::New<String>("sum_of_squares").ToLocalChecked();
    Local<Array> squares = New<v8::Array>(array->Length());
    double ss = 0;

    for (unsigned int i = 0; i < array->Length(); i++ ) {
      if (Nan::Has(array, i).FromJust()) {
        // get data from a particular index
        double value = Nan::Get(array, i).ToLocalChecked()->NumberValue();
        
        // set a particular index - note the array parameter
        // is mutable
        Nan::Set(array, i, Nan::New<Number>(value + 1));
        Nan::Set(squares, i, Nan::New<Number>(value * value));
        ss += value*value;
      }
    }
    // set a non index property on the returned array.
    Nan::Set(squares, ss_prop, Nan::New<Number>(ss));
    info.GetReturnValue().Set(squares);
}
{% endhighlight %}

{% highlight cpp %}
// index.js
var inputs = [1, 2, 3];
var squares = addon.pass_array(inputs);
assert(inputs[0] == 2);
assert(inputs[1] == 3);
assert(inputs[2] == 4);
assert(squares[0] == 1);
assert(squares[1] == 4);
assert(squares[2] == 9);
assert(squares.sum_of_squares == 14);
{% endhighlight %}

## What else should you know?
There are bunch more topics that you'll need to learn before really being able to take full advantage C++ from Node.js.  Here are a few important topics, and some resources on this site for learning them.

- **Asynchronous addons methods**:  Most addon methods shouldn't block the event loop when called, which means you should make them asynchronous - they will accept a callback which will be used to return the results of the call.  You can start learning the asynchronous pattern [here](/c-processing-from-node-js-part-4-asynchronous-addons/), and using NAN with async is covered [here](/building-an-asynchronous-c-addon-for-node-js-using-nan/).  You'll also want to check out [streaming data from C++ to Node.js](/streaming-data-from-c-to-node-js/), which extends the asynchronous model. 

- **Using Buffer objects**:  You need to be mindful of copying data between V8 and C++ proper, you can incur some penalties you might not expect.  In addition, memory management and multithreading (asynchronous addons) can be tricky.  Check out my article on [how not to share memory between async threads](/how-not-to-access-node-js-from-c-worker-threads/), and learn about [using buffers](/buffers-and-c-on-rising-stack-community/) to side step this issue and save yourself a lot of memory copying.

- **Deploying addons**:  You can publish addons to the npm registry just like any other module, but when you do this it means the C++ code in your addon needs to be built on the target machine, using a C++ compiler.  If that's an issue, check out [using `node-pre-gyp`](/cross-platform-addons-with-node-pre-gyp/) to publish and distribute binaries for your supported platforms.  While your at it, check out [deploying your addons to Amazon Lambda](/c-addons-as-aws-lambda-functions/) too! 


