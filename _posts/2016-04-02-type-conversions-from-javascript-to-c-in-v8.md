---
layout: post
sidebar : true
blog: true
title:  "Type conversions from JavaScript to C++ in V8"
date:   2016-04-02 15:40:56
permalink: /type-conversions-from-javascript-to-c-in-v8/
name: type-conversions-from-javascript-to-c-in-v8
disqus_id : silvrback-scottfrees-23813
disqus_shortname: scottfrees
---
Learning how to pass information from JavaScript to C++ addons can be tricky.  Confusion stems from the extreme differences between the JavaScript and C++ type system.  While C++ is strongly typed ("42" is not an integer... it's a string, and only a string!), JavaScript is very eager to convert datatypes for us.  
<!--more-->
JavaScript primitives include Strings, Numbers, and Booleans - along with `null` and `undefined`.  V8 uses an inheritance hierarchy where `Primitive` extends `Value`, and all the individual primitives subclass `Primitive`.  In addition to the standard JavaScript primitives, V8 also supports integers (`Int32` and `Uint32`).  You can see all the rest of the Value types [here](https://v8docs.nodesource.com/io.js-3.0/dc/d0a/classv8_1_1_value.html).

In this post, I'll show you some basics for working with numerics, and I'll point you to a **cheat sheet** I've used to learn how to do the [rest of the conversions](https://github.com/freezer333/nodecpp-demo/tree/master/conversions).  I also cover this topic in a lot more detail in my ebook on [Node.js C++ Integration](https://scottfrees.com/ebooks/nodecpp/), which you can [buy here](https://gumroad.com/l/dTVf).  

All references to JavaScript values are held in C++ through a Handle object - in most cases a `Local`.  The handles point to V8 storage cells within JavaScript runtime.  You can learn a lot more about storage cells in my previous post on [memory issues in V8](http://blog.scottfrees.com/how-not-to-access-node-js-from-c-worker-threads).

As you go through the API for working with primitives, you'll notice there is no assignment of `Local` objects - which at first may seem odd!  It makes a lot of sense however - for three reasons:

1. JavaScript primitives are *immutable* - variables "containing" primitives are just pointing to unchanging V8 storage cells.  Assignment of a `var x = 5;` makes `x` point to a storage cell with 5 in it - reassigning `x = 6` does not change this storage cell - it simply makes `x` point to another storage cell that contains 6.  If `x` and `y` are both assigned the value of `10`, they both point to the same storage cell.  
2. Function calls are pass-by-value, so whenever JavaScript calls a C++ addon with a parameter - if that parameter is a primitive, it is always a distinct *copy* - changing it's value has no effect in the calling code.
3. Handles (`Local&lt;Value&gt;`) are references to *storage cells*.  Thus, given #1 above, it doesn't make sense to allow handle values to change - since primitives don't change!

Hopefully that makes some sense - however it's still likely you'll need to *modify* V8 variables... we'll just need to do this by *creating* new ones and assigning the new value to them.

## Number Example
Now let's look at the Number primitive type and what happens when we construct a C++ addon to accept them from JavaScript.  I've built an addon that exposes the following C++ function as an export called `pass_number`:

```c++
void PassNumber(const FunctionCallbackInfo&lt;Value&gt;&amp; args) {
    Isolate * isolate = args.GetIsolate();

    double value = args[0]-&gt;NumberValue();

    value+= 42;

    Local&lt;Number&gt; retval = Number::New(isolate, value);

    args.GetReturnValue().Set(retval);
}
```

The complete addon code is [here](https://github.com/freezer333/nodecpp-demo/blob/master/conversions/loose/loose_type_demo.cpp).

The addon does virtually nothing to ensure that the argument passed into the function is a Number, or if it even exists!  Here's the associated mocha tests - we can see how V8 handles numbers, and more importantly, other input that can or cannot be converted to numbers in JavaScript.

```javascript

describe('pass_number()', function () {
  it('return input + 42 when given a valid number', function () {
    assert.equal(23+42, loose.pass_number(23));
    assert.equal(0.5+42, loose.pass_number(0.5));
  });

  it('return input + 42 when given numeric as a string', function () {
    assert.equal(23+42, loose.pass_number("23"));
    assert.equal(0.5+42, loose.pass_number("0.5"));
  });

  it('return 42 when given null (null converts to 0)', function () {
    assert.equal(42, loose.pass_number(null));
  });

  it('return NaN when given undefined', function () {
    assert(isNaN(loose.pass_number()));
    assert(isNaN(loose.pass_number(undefined)));
  });

  it("return NaN when given a non-number string", function () {
    assert(isNaN(loose.pass_number("this is not a number")));
  });

```


## Complete type conversion cheat sheet
I've created a [repository](https://github.com/freezer333/nodecpp-demo/tree/master/conversions) which has a type conversion cheat sheet that I think is pretty useful.  To get it:

```
&gt; git clone https://github.com/freezer333/nodecpp-demo.git
```

To build both addons, go into the `loose` and `strict` directories and issue a `node-gyp configure build` command in each.  You'll need to have installed `node-gyp` globally first.  If your completely new to this - [check this out](http://blog.scottfrees.com/c-processing-from-node-js).

```
&gt; cd nodecpp-demo/conversion/loose
&gt; node-gyp configure build
...
&gt; cd ../strict
&gt; node-gyp configure build
```

The two addons (loose and strict) expose a series of functions that accept different types - Numbers, Integers, Strings, Booleans, Objects, and Arrays - and perform (somewhat silly) operations on them before returning a value.  I've included a JavaScript test program that shows you the expected outputs of each function - but the real learning value is in the addons' C++ code ([strict](https://github.com/freezer333/nodecpp-demo/blob/master/conversions/loose/loose_type_demo.cpp)/[loose](https://github.com/freezer333/nodecpp-demo/blob/master/conversions/strict/strict_type_demo.cpp))

To run the tests (you'll need `mocha` installed), go into the `conversions` directory (with `index.js`):

```
&gt; npm test
```

The 'loose' addon has very little type checking - it's basically mimicking how a pure JavaScript function would work.  For example, the `pass_string` function accepts anything that could be converted to a string in JavaScript and returns the reverse of it:

```javascript

describe('pass_string()', function () {
  var str = "The truth is out there";

  it('reverse a proper string', function () {
    assert.equal(reverse(str), loose.pass_string(str));
  });
  it('reverse a numeric/boolean since numbers are turned into strings', function () {
    assert.equal("24", loose.pass_string(42));
    assert.equal("eurt", loose.pass_string(true));
  });
  it('return "llun" when given null - null is turned into "null"', function () {
    assert.equal("llun", loose.pass_string(null));
  });
  it('return "denifednu" when given undefined', function () {
    assert.equal(reverse("undefined"), loose.pass_string(undefined));
  });
  it('return reverse of object serialized to string', function () {
    assert.equal(reverse("[object Object]"), loose.pass_string({x: 5}));
  });
  it('return reverse of array serialized to string', function () {
    assert.equal(reverse("9,0"), loose.pass_string([9, 0]));
  });
});
```

Here's the C++ code for the loose conversions of string input:

```c++
void PassString(const FunctionCallbackInfo&lt;Value&gt;&amp; args) {
    Isolate * isolate = args.GetIsolate();

    v8::String::Utf8Value s(args[0]);
    std::string str(*s);
    std::reverse(str.begin(), str.end());    

    Local&lt;String&gt; retval = String::NewFromUtf8(isolate, str.c_str());
    args.GetReturnValue().Set(retval);
}
```

The 'strict' addon performs full type and error checking, behaving more like a C++ function than a JavaScript. For all of the strict addon methods, `undefined` is always returned if the input to the function isn't *exactly* what was expected. For example, the pass_string function behaves quite differently than the loose interpretation:

```javascript
describe('pass_string()', function () {

  it('return reverse a proper string', function () {
    var str = "The truth is out there";
    it('reverse a proper string', function () {
      assert.equal(reverse(str), strict.pass_string(str));
    });
  });

  it('return undefined for non-strings', function () {
    assert.equal(undefined, strict.pass_string(42));
    assert.equal(undefined, strict.pass_string(true));
    assert.equal(undefined, strict.pass_string(null));
    assert.equal(undefined, strict.pass_string(undefined));
    assert.equal(undefined, strict.pass_string({x: 5}));
    assert.equal(undefined, strict.pass_string([9, 0]));
  });
});
```

```c++
void PassString(const FunctionCallbackInfo&lt;Value&gt;&amp; args) {
    Isolate * isolate = args.GetIsolate();

    if ( args.Length() &lt; 1 ) {
        return;
    }
    else if ( args[0]-&gt;IsNull() ) {
        return;
    }
    else if ( args[0]-&gt;IsUndefined() ) {
        return;
    }
    else if (!args[0]-&gt;IsString()) {
        // This clause would catch IsNull and IsUndefined too...
        return ;
    }

    v8::String::Utf8Value s(args[0]);
    std::string str(*s);
    std::reverse(str.begin(), str.end());    

    Local&lt;String&gt; retval = String::NewFromUtf8(isolate, str.c_str());
    args.GetReturnValue().Set(retval);
}
```

Go [ahead and download](https://github.com/freezer333/nodecpp-demo) the complete source and take a look - the code is in the `/conversions` directory.  You'll see examples using Integers, Booleans, Objects, and Arrays.

## Looking for more info?
This post is actually a small excerpt from a book  I've published - [Node.js C++ Addons](http://scottfrees.com/ebooks/nodecpp/) that covers this in detail.  In it, you'll also find equivalent code when using NaN.  If you are interested, click [here](http://scottfrees.com/ebooks/nodecpp/) for the full contents and info on how to get your copy.