--- 
name: microservices-jvm-clojure
title: Micro services on the JVM part 1 - Clojure
time: 2014-02-24 15:52:00 +01:00
layout: post
tags: jvm, micro-services, clojure
---
Micro services could be a buzzword of 2014 for me. Few months ago I was curious to try [Dropwizard](http://www.dropwizard.io/) framework as a separate backend, but didn't get the whole idea yet. But then I watched a mind-blowing ["Micro-Services Architecture"](http://www.youtube.com/watch?v=2rKEveL55TY) talk by Fred George. Also, the 4.0 release notes of **Spring** [covers microservices](https://spring.io/blog/2013/12/12/announcing-spring-framework-4-0-ga-release) as an important rising trend as well. After 10 years of having SOA in mind, but still developing monoliths, it's a really tempting idea to try to decouple systems into a set of independently developed and deployed RESTful services.

So when I decided to write a simple API for my [DevRates.com](http://devrates.com) website, instead of adding some code to existing codebase, I wanted to build a separate tiny app. But what's the best stack for micro-services? In this series of posts I'll try to compare various JVM technology stacks for this approach.

Here is my list of must-have features for the stack:

* declarative REST support (no manual URL parsing)
* native JSON support (bidirectional JSON-object mapping)
* single "fat" jar packaging, no web container needed
* fast development feedback loop (eg. runtime code reloading)
* [Swagger](https://github.com/wordnik/swagger-core) and [Metrics](http://metrics.codahale.com/) integration

In this post I'll try to cover **Clojure** with [Ring](https://github.com/ring-clojure/ring) and [Compojure](https://github.com/weavejester/compojure).

## TL;DR ##
You can find all the covered concepts in the following GitHub examples:

* [compojure-rest](https://github.com/pjagielski/microservices-jvm/tree/master/compojure-rest) - basic **Compojure** app with main class
* [compojure-swag](https://github.com/pjagielski/microservices-jvm/tree/master/compojure-swag) - **Swagger** and **Metrics** integration
* [clorates](http://github.com/pjagielski/clorates) - complete implementation of [DevRates.com API](http://devrates.com/api/swagger/index.html)

## Basic setup ##
There is an excellent [Zaiste's tutorial](http://zaiste.net/2014/02/web_applications_in_clojure_all_the_way_with_compojure_and_om/) showing how to kickstart REST app with **Compojure**, just follow these few simple steps (the rest of the post assumes `compojure-rest` as the app name).

My sample route from `handler.clj`:
{% highlight clojure %}
(defroutes app-routes
  (GET "/messages/:name" [name] {:body {:message (str "Hello World" " " name)}})
  (route/resources "/")
  (route/not-found "Not Found"))
{% endhighlight %}

## Fat jar ##
In a simple setup, **Compojure** app is being run through lein ring plugin. To enable running it as a standalone command-line app, you have to write a main method which starts **Jetty** server.

`project.clj`
{% highlight clojure %}
 :dependencies ...
         [ring/ring-jetty-adapter "1.2.0"]
         ..
 :main compojure-rest.handler
{% endhighlight %}

`handler.clj`
{% highlight clojure %}
(ns compojure-rest.handler
 ...
 (:require ...
   [ring.adapter.jetty :refer (run-jetty)])
   (:gen-class))
...
(defn -main [& args]
  (run-jetty app {:port 3000 :join? false }))
{% endhighlight %}

To build a single "fat" jar just run `lein uberjar`, and then `java -jar target/compojure-rest-0.1.0-SNAPSHOT-standalone.jar` runs the app.

## Swagger ##
The nice thing about **Compojure** is that you can easy expose Swagger documentation by using [swag](https://github.com/narkisr/swag) library. There are some conflicts between swag and ring lein plugin, so just look at the [compojure-swag](https://github.com/pjagielski/microservices-jvm/tree/master/compojure-swag) for a working example. 

Here is a typical snippet from `handler.clj`:
{% highlight clojure %}
(set-base "http://localhost:3000")

(defroutes- messages {:path "/messages" :description "Messages management"}
  (GET- "/messages/:name" [^:string name] {:nickname "getMessages" :summary "Get message"}
      {:body {:message (str "Hello World" " " name)}})
  (route/resources "/")
  (route/not-found "Not Found"))
{% endhighlight %}

So, `swag` introduces `defroutes-`, `GET-`, `POST-` which take additional metadata as parameters to generate Swagger docs. If you're little scared with this `^:string` fragment - check [metadata](http://clojure.org/metadata) section from Clojure manual. Swagger-compatible definition should be available at `http://localhost:3000/api-docs.json` after running the app.

## Metrics ##

To expose basic metrics of your REST API calls just use Ring-compatible `metrics-clojure-ring` library.

`project.clj`
{% highlight clojure %}
 :dependencies ...
         [metrics-clojure-ring "1.0.1"]
         ...
{% endhighlight %}

`handler.clj`
{% highlight clojure %}
(ns compojure-rest.handler
 ...
 (:require ...
    [metrics.ring.expose :refer [expose-metrics-as-json]]
    [metrics.ring.instrument :refer [instrument]]))
...
(def app (expose-metrics-as-json (instrument app) "/stats/"))
{% endhighlight %}

After sending some requests, you can check the collected stats by visiting `http://localhost:3000/stats/`:

{% highlight javascript %}
ring.requests.rate.GET: {
    type: "meter",
        rates: {
        1: 189.5836593065824,
        5: 39.21602480726734,
        15: 13.146759983907245
        }
    }
{% endhighlight %}

## Some random Clojure thoughts ##

* The best newbie guide to Clojure is Kyle Kingsbury's ["Clojure from the ground up"](http://aphyr.com/tags/Clojure-from-the-ground-up) series.
* Leiningen is probably the best build tool for the JVM. Easy to install, fast, simple, no XML - just doing it right. And the "new" project templates is what's Maven been missing from ages (anyone using archetypes?).
* Lighttable is great! I'm really impressed with the fast feedback loop by just ctrl+entering the expressions. 
* Also, live reloading with `ring server` works fine. Just change the change code and see the changes immediately. Rapid!
* Unlike other recently popular languages, Clojure has no killer-framework. Rails, Play/Akka, Grails/Gradle - all of these are key parts of Ruby, Scala and Groovy ecosystems. What about Clojure? A collection of small (micro?) libraries doing one thing well and working great together - just like Unix commands.
* It may be true that Clojure is not good for large projects. With all the complex contructs (meta or ) and no control of the visibility, it could be hard to maintain large codebase. But it's not a first-class problem in a micro-services world..

## Resources ##
* [Talk by Fred George](https://www.youtube.com/watch?v=2rKEveL55TY)
* [Java, the Unix Way - James Lewis from ThoughtWorks](http://www.infoq.com/presentations/Micro-Services)
* [Micro Service Architecture - nice article](http://yobriefca.se/blog/2013/04/29/micro-service-architecture/)