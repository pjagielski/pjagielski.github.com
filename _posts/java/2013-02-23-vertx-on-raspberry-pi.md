--- 
name: vertx-on-raspberry-pi
title: Vert.x on Raspberry Pi
time: 2013-02-23 21:35:00 +01:00
layout: post
tags: java, jvm, vertx, nodejs
---
## Lightweight JVM web server ##
Some people say that using Java on Raspberry Pi is a stupid idea. Sure, JVM uses a lot of system resources but when you can watch HD movies on RPi why don't run a tiny Java app? But I do mean **really** tiny - no Tomcat nor any other application server involved, no war and deployment stuff. That's why I came out with idea of using [vert.x](http://vertx.io) - a relatively new framework for writing modern web application. In this post what lessons I learned after developing a saimple app using vert.x and then running it on the Raspberry Pi.

## Working with vert.x on RPi ##
Vert.x is [node.js](http://nodejs.org/) on steroids for JVM. In addition to all the libs for the JVM it offers you a lot more:

* polyglot - write your application in any language that runs on the JVM (or mix a few)
* really simple programming model
* inter-app communication via event bus and shared data
* ease of scalability and concurrency handling

If you want a gentle start in the world of vert.x I definitely suggest you reading the [manual](http://vertx.io/manual.html) on project website.

### Installation ###
First of all - install Java 8 early access build for ARM from [Oracle website](http://jdk8.java.net/fxarmpreview). Vert.x indeed requires Java 7, but the 8 build has a real hard float support.

The easiest way to manage vert.x installation is to use [GVM](http://gvmtool.net/):

{% highlight bash %}
curl -s get.gvmtool.net | bash
gvm install vertx
{% endhighlight %}

Now you should be able to run `vertx` from the command line.

### Development process ###
In my approach I'm developing the app on my laptop and want to have straight path of running the code on RPI. Since I love the Intellij Idea 12 I put my sources into common maven/gradle structure and use `Verticle` interface to get full benefits of my IDE: code completion, refactoring and testing support. I use Groovy as a language for my verticles but I want the sources to be compiled on my laptop instead of RPI which a bit improves startup time of the app. 

The examples for vert.x distribution are not reaaly development-friendly but I've found great example of [Groovy starter project](https://github.com/rmangi/vertx-groovy-starter) on GitHub which introduces two huge ideas: separating configuration of the app in single JSON file and a `runVertx` gradle task to build and run the app in a single step. I also use the [web-server vert.x module](https://github.com/vert-x/mod-web-server) to serve static web files of the app frontend. So I have SockJS and REST verticles in the backend and JavaScript frontend in a single project root.

Here is the final layout explained:
{% highlight bash %}
src/main/groovy - verticles code
    App.groovy - app lancher
src/main/test - verticles tests
web - frontend stuff
build.gradle - gradle script with runVertx task
run.sh - script for running app on RPI (no compilation)
{% endhighlight %}

And a typical development process looks like this:

* write verticles code (\*)
* write unit tests for verticles (\*)
* run the app on laptop and modify the frontend
* copy binaries to RPi and run the app from command line

(\*) change the order of these two for TDD ;)

The only thing I'm only struggling from is lack of runtime reloading of verticles. Maybe using JRebel could help there but I ain't got the licence (yet).

You can see my example project [here](https://github.com/pjagielski/vertx-raspberry-console).

## Benchmark ##
I also did some benchmarking of vert.x app on RPI throughput. Sure, you cannot expect that single RPI could handle a lot of transactions per sec, so the test was just for fun. In the test I served a static CSS (bootstrap) and invoked a really simple REST resource (example `details/Joe/123` mapping from `RouteMatcher`). I used siege with 2 and 200 consumers and compared the results between the very.x (1 instance), node.js express and nginx (just for static CSS). I installed node.js 0.8.18 from binaries found [here](https://gist.github.com/adammw/3245130).

Here are the results:

<img src="/assets/img/vertx-node-nginx-perf.png"/>

* It is nice that serving static files is quite as fast as in the nginx. There is [a benchmark](http://monkey-project.com/benchmarks/raspberry_pi_monkey_nginx) showing that Monkey web server is a bit faster, but in my environment I wasn't able to achieve more than 45 req/sec on nginx, so ~40 req/sec on vert.x is really good result.
* Vert.x seems to be about 2-2.5 times faster than node.js with express. This is similar to Tim Fox' [benchmark](http://vertxproject.wordpress.com/2012/05/09/vert-x-vs-node-js-simple-http-benchmarks/) with both node.js and vert.x in single instance. (BTW: I have no idea how to get 30k req/sec on a workstation as in Tim's test).
* The benchmark shows that RPi is indeed a toy PC - on my workstation it's really easy to achieve thousands of requests per second. So forget about handling hundreds of requests per seconds on a single RPi instance.

## (a bit) Speeding up startup time ##
The thing that really puts node.js in front is the startup time. Java is slow at both JVM startup and while classloading. My first application deployed on RPi took about 30 sec to start, while on my laptop it was about 2-3 seconds. It is possible to cut it down to 18 seconds, here are a few hints:

* don't compile sources directly on RPi - build a jar on your workstation. In my setup I run the server from gradle on my laptop but also have a bash script to run it on raspberry which passes vert.x a `-cp` parameter pointing to builded jar.
* don't run a SockJS server when not needed. It tooks about 5 sec to start. You can disable it by setting `bridge` to false in mod-web configuration.
* you can write your own mod-web in just a few lines of code. You will save some milliseconds but also this setup allows you to pass custom RESTful routes to `RouteMatcher`. Look at this example:

{% highlight ruby %}
    def rm = new RouteMatcher()
    rm.get('/details/:user/:id', { HttpServerRequest req ->
        req.response.end "User: ${req.params()['user']} ID: ${req.params()['id']}"
    } as Handler<HttpServerRequest>)

    rm.get('/', { HttpServerRequest req ->
        req.response.sendFile("web/index.html")
    } as Handler<HttpServerRequest>)

    // Catch all - serve the index page
    rm.getWithRegEx('.*', { HttpServerRequest req ->
        req.response.sendFile "web/${req.path}"
    } as Handler<HttpServerRequest>)
{% endhighlight %}

To debug the most time consuming steps on app startup, you should configure log4j logging. Vert.x outputs some info about deploying verticles, you can also put your own logging messages. Just copy log4j.jar to vert.x lib directory and add 
```
-Dorg.vertx.logger-delegate-factory-class-name=org.vertx.java.core.logging.impl.Log4jLogDelegateFactory
```
 parameter in vert.x startup script.

## Sample project ##
You can check these concepts by cloning a [top monitor](https://github.com/pjagielski/vertx-top-monitor) GitHub project. It is a [Dashing](http://shopify.github.com/dashing/)-inspired web app showing results from calling a `top` command by lucid [JQuery knobs](http://anthonyterrien.com/knob/).

<img src="/assets/img/vertx-top.png" style="max-width:700px;"/>

The project itself it not smashing, you can just check the structure and the tests when I'm mocking ARM-specific `top` command results. Enjoy!
