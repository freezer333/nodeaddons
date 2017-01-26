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

Motivation... to make addons that support a variety of platforms, and do not need to be build on the end users's machine.
<!--more-->
## What is `node-preaTalk-gyp`?
Explain how it works, what the steps are.
Talk a bit about node-abi, platforms, versions.

## Example application - time
Explain that this is silly, show the 3 code examples for linux, macos, and windows

{% highlight cpp %}
// Platform specific (macOS)
#include <mach/mach.h>
#include <mach/mach_time.h>

NAN_METHOD(now) {
    static double timeConvert = 0.0;
	if ( timeConvert == 0.0 )
	{
		mach_timebase_info_data_t timeBase;
		(void)mach_timebase_info( &timeBase );
		timeConvert = (double)timeBase.numer /
			(double)timeBase.denom /
			1000000000.0;
	}
	double time_now =  (double)mach_absolute_time( ) * timeConvert;
    Local<Number> retval = Nan::New(time_now);
    info.GetReturnValue().Set(retval);    
}
{% endhighlight %}

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
