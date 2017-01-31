---
layout: post
sidebar : true
blog: true
newsletter: true
title:  "C++ and Node.js Integration - eBook Published"
date:   2016-06-17 15:40:56
permalink: /c-and-node-js-integration-ebook-published/
name: c-and-node-js-integration-ebook-published
disqus_id : silvrback-scottfrees-25389
disqus_shortname: scottfrees
---
I’ve got good news, I’ve finally completed the [C++ and Node.js Integration ebook](https://gumroad.com/l/dTVf)! I want to thank all of you for signing up for my newsletter to keep posted about the book - you kept me motivated!  Thanks also to all the people who reached out to me with ideas, tips, and insights into the topic - you were extremely helpful.
<!--more-->
You can [buy the book on gumroad](https://gumroad.com/l/dTVf) now, and there is a lot more information on the [book's webpage](/book/).  For those of you that purchase the book, it would be great to hear feedback from you - either on this post or just email me!

The book is available in PDF, HTML, and epub.  **There's over 200 pages of text**, walking you through dozens of addons.  You can also get full access to the code discussed in the text over at [nodecpp-demo](https://github.com/freezer333/nodecpp-demo).  

**If you'd like a sample chapter**, [sign up](http://eepurl.com/bIAQOb) for my Node.js Integration newsletter to keep up with posts to this blog.

Here's a summary of the book contents - you can find more details [here](/book/).

- **Chapter 1 - Introduction to Node.js Addons:**  Learn the basics of creating Node.js Addons and how to use node-gyp to create and test them.  We'll go through a few quick examples, and outline the rest of the book.
-  __Chapter 2 - Understanding the V8 API:__  Learn about the C++ underpinnings of Node.js and how it all works.  You'll see why Node's interface with C++ is so efficient, and how V8's design effects how you write your addons.  In this chapter we'll cover all the V8 data types, and how the V8 memory system can be accessed from C++.
- **Chapter 3 - Basic Integration Patterns:**  See how to create your first full-featured C++ addon and pass JavaScript objects to C++.  We'll look at turning your JavaScript objects into first-class objects matching a C++ class definition and how to return data back to Node.js.
- **Chapter 4 - Asynchronous Addons:**  What good is a Node.js module if it blocks your event loop?  In this chapter you'll learn how to do the heavy computational task you've written in C++ asynchronously.  Just give your C++ code a JavaScript callback function, and your JavaScript can just keep on running until the C++ is done.
- **Chapter 5 - Object Wrapping:**  If you have existing C++ classes that you want to be able to use as "native" JavaScript objects, then you'll need to learn to wrap your C++ classes with V8 data structures.  
- **Chapter 6 - Native Abstractions for Node (NAN):**  The V8 API has undergone many "breaking" changes - and since different version of Node.js use different versions of V8, the situation can get pretty complex to manage.  This chapter will show you how to work at a higher level of abstraction using &lt;a href="https://github.com/nodejs/nan"&gt;Nan&lt;/a&gt; - the Native Abstractions for Node.js library.  Using Nan, you'll defend yourself from the changing V8 API.
- **Chapter 7 - Streaming between Node and C++:**  Learn how to build great interfaces to your C++ addon by supporting an event emitter and streaming interface for both sending data to youR addon, and emitting/streaming data asynchronously back to JavaScript.
- **Chapter 8 - Publishing Addons:**  Publishing your typical Node module to an &lt;a href=""&gt;npm&lt;/a&gt; repo is pretty simple - but since your C++ is &lt;i&gt;native executable code&lt;/i&gt;, there are some hoops you'll need to jump through so your addon works on all operating systems.  In this chapter, we'll cover solutions to common pitfalls involved in publishing/deploying C++ addons.
- **Appendix A - Getting your C++ to the Web:**  Addons aren't the only way of moving C++ to the web - in this section we'll take a look at some of the other options, like automation and shared libraries.
- **Appendix B - Buffers:**  One of the most efficient ways to move data between JavaScript and your C++ addons is by using buffers - which get stored outside V8's memory.  In this section we'll go through an image processing example that uses buffers to convert pixel data from PNG to BMP.

You can [get the book here](https://gumroad.com/l/dTVf) - enjoy!