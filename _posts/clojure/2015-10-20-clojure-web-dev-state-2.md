--- 
name: clojure-web-dev-state-2
title: Clojure web development - state of the art - part 2
time: 2015-10-20 10:00:00
layout: post
tags: jvm, clojure, webapps
---

This is part 2 of my “Clojure web development” series. You can discuss first part on [this](https://www.reddit.com/r/Clojure/comments/3lgkj7/clojure_web_development_state_of_the_art/) reddit thread. After reading the comments I must explain two assumptions I had writing this series:

* Keep things easy to understand for people from outside Clojure land, especially Java devs. That’s why I use REST/JSON in favor of transit and component as a “dependency injection” implementation which could be easily explained as Spring framework equivalent. The same goes with Om which is a bit verbose, but in my opinion it’s easier to understand for a start and has wider adoption than the other React wrappers.

* Keep things easy to bootstrap on a developer machine. This is a hands-on walkthrough and all the individual steps have been committed to GitHub. That’s why I use MongoDB, which could not be the best choice for scaling your application to millions of users, but it’s perfect for bootstrapping - no schema, tables, just insert data and start working. I highly recommend Honza Kral [polyglot persistence](https://www.youtube.com/watch?v=W6J_jz6oifs) talk, where he encourages starting simple and optimize for developer happiness at the start of a project.

In previous post we bootstrapped a basic web application serving REST data with (static for now) Clojurescript frontend, all fully reloadable thanks to reloaded repl and figwheel. You can find final working version of it in [this branch](https://github.com/pjagielski/modern-clj-web/tree/part1).

Today we’re going to display contact list stored in MongoDB. I assume you have MongoDB installed, if not - it’s trivial with [docker](https://github.com/dockerfile/mongodb).

Serving contact list from database
----------------------------------

OK, let’s start. In the backend we need to add some dependencies to project.clj:

{% highlight clojure %}
 :dependencies
   ...
   [org.danielsz/system "0.1.9"]
   [com.novemberain/monger "2.0.0"]]
{% endhighlight %}

`monger` is an idiomatic Clojure wrapper for Mongo Java driver and `system` is a nice collection of components for various datastores, including Mongo (a bit like Spring Data for Spring).

In order to interact with a data store I like to use the concept of abstract repository. This should hide the implementation details from the invoker and allows to switch to another store in the future. So let’s create an abstract interface (in Clojure - protocol) in `components/repo.clj`:

{% highlight clojure %}
(ns modern-clj-web.component.repo)

(defprotocol ContactRepository
  (find-all [this]))
{% endhighlight %}

We need this as a parameter to allow Clojure runtime dispatching correct implementation of this repository. Mongo implementation with Monger is really simple:

{% highlight clojure %}
(ns modern-clj-web.component.repo
   (:require [monger.collection :as mc]
             [monger.json]))
...

(defrecord ContactRepoComponent [mongo]
  ContactRepository
  (find-all [this]
    (mc/find-maps (:db mongo) "contacts")))

(defn new-contact-repo-component []
  (->ContactRepoComponent {}))
{% endhighlight %}

Things to notice here:

* `mc/find-maps` just returns all records from collection as Clojure maps
* `ContactComponent` gets injected with mongo component created by system library, which adds Mongo client under `:db` key
* Since this component is stateless, we don’t need to implement `component/Lifecycle` interface, but it still can be wired in system like a typical lifecycle-aware component
* Requiring `monger.json` adds JSON serialization support for Mongo types (e.g. `ObjectId`)

Ok, it’s now time to use our new component in the endpoint/example.clj:

{% highlight clojure %}
(:require
  ...
  [modern-clj-web.component.repo :as r])

(defn example-endpoint [{repo :contact-repo}]
 (routes
...
   (GET "/contacts" [] (response (r/find-all repo)))
{% endhighlight %}

The `{repo :contact-repo}` notation (destructuring) automatically binds `:contact-repo` key from system map to the `repo` value. So we need to assign our component to that key in `system.clj`:

{% highlight clojure %}
(:require
  ...
  [modern-clj-web.component.repo :refer [new-contact-repo-component]]
  [system.components.mongo :refer [new-mongo-db]])

 (-> (component/system-map
          :app  (handler-component (:app config))
          :http (jetty-server (:http config))
          :example (endpoint-component example-endpoint)
          :mongo (new-mongo-db (:mongo-uri config))
          :contact-repo (new-contact-repo-component))
         (component/system-using
          {:http [:app]
           :app  [:example]
           :example [:contact-repo]
           :contact-repo [:mongo]}))))
{% endhighlight %}

In short - we use system’s `new-mongo-db` to create Mongo component, make it a dependency to repository which itself is a dependency of example endpoint.

And finally we need to configure `:mongo-uri` config property in `config.clj`:

{% highlight clojure %}
 (def environ
  {:http {:port (some-> env :port Integer.)}}
   :mongo-uri "mongodb://localhost:27017/contacts"})
{% endhighlight %}

To check if it works fine, restart repl, type `(go)` again and make a GET to [http://localhost:3000/contacts](http://localhost:3000/contacts).

{% highlight bash %}
curl http://localhost:3000/contacts
[]
{% endhighlight %}

OK, so we got empty list since we don’t have any data in Mongo database. Let’s add some with mongo console:

{% highlight bash %}
mongo localhost:27017/contacts
MongoDB shell version: 2.4.9
connecting to: localhost:27017/contacts
>  db.contacts.insert({firstname: "Al", lastname: "Pacino"});
>  db.contacts.insert({firstname: "Johnny", lastname: "Depp"});
{% endhighlight %}

And finally our endpoint should return these two records:

{% highlight bash %}
curl http://localhost:3000/contacts
[{"lastname":"Pacino","firstname":"Al","_id":"56158345fd2dabeddfb18799"},{"lastname":"Depp","firstname":"Johnny","_id":"56158355fd2dabeddfb1879a"}]⏎ 
{% endhighlight %}

Sweet! Again - in case of any problems, check [this commit](https://github.com/pjagielski/modern-clj-web/commit/a4b858127299cfea1774c9efe4ffbab0e45df161).

Getting contacts from ClojureScript
-----------------------------------
In this step we'll fetch the contacts with AJAX call on our ClojureScript frontend. As usual, we need few dependencies in `project.clj` for a start:

{% highlight clojure %}
  :dependencies 
     ...
       [org.clojure/clojurescript "1.7.48"]
       [org.clojure/core.async "0.1.346.0-17112a-alpha"]
       [cljs-http "0.1.37"]
{% endhighlight %}

ClojureScript should be already visible by using `figwheel`, but it's always better to require specific version explicitly. `cljs-http` is a HTTP client for ClojureScript and `core.async` provides facilities for asynchronous communication in CSP model, especially useful in ClojureScript. Let's see how it works in practice.

To make an AJAX call we need to call methods from `cljs-http.client`, so let's add this in `core.cljs`:

{% highlight clojure %}
(ns ^:figwheel-always modern-clj-web.core
  (:require [cljs-http.client :as http]))

(println (http/get "/contacts"))
{% endhighlight %}

You should see `#object[cljs.core.async.impl.channels.ManyToManyChannel]`. What madness is that???

This is the time we enter the `core.async`. The most common way to deal with asynchronous network calls from Javascript is by using callbacks or promises. The `core.async` way is by using channels. It makes your code look more like a sequence of synchronous calls and it's easier to reason about. So the `http/get` function returns a channel on which the result is published when response arrives. In order to receive that message we need to read from this channel by using `<!` function. Since this is blocking, we also need to surround this call with `go` macro, just like in [go language](https://gobyexample.com/goroutines). So the correct way of getting contacts looks like this:

{% highlight clojure %}
(:require
...
  [cljs.core.async :refer [<! >! chan]])

(go
  (let [response (<! (http/get "/contacts"))]
    (println (:body response))))
{% endhighlight %}


Adding Om component
-------------------
Dealing with frontend code without introducing any structure could quickly become a real nightmare. In late 2015 we have basically two major JS frameworks on the field: Angular nad React. ClojureScript paradigms (functional programming, immutable data structures) fit really nice into React philosophy. In short, React application is composed of *components* taking data as input and rendering HTML as output. The output is not a real DOM, but so-called *virtual DOM*, which helps calculating diff from current view to updated one. 

Among many React wrappers in ClojureScript I like using [Om](https://github.com/omcljs/om) with [om-tools](https://github.com/Prismatic/om-tools) to reduce some verbosity. Let's introduce it into our `project.clj`: 

{% highlight clojure %}
  :dependencies 
     ...
   [org.omcljs/om "0.9.0"]
   [prismatic/om-tools "0.3.12"]
{% endhighlight %}

To render a "hello world" component we need to add some code in `core.cljs`:

{% highlight clojure %}
   (:require 
   ...
            [om.core :as om]
            [om-tools.core :refer-macros [defcomponent]]
            [om-tools.dom :as dom :include-macros true]))

(def app-state (atom {:message "hello from om"}))

(defcomponent app [data owner]
  (render [_]
    (dom/div (:message data))))

(om/root app app-state
         {:target (.getElementById js/document "main")})
{% endhighlight %}

What's going on here? The main concept of Om is keeping whole application state in one *global atom*, which is Clojure way of managing state. So we pass this `app-state` map (wrapped in `atom`) as a parameter to `om/root` which mounts components into real DOM (`<div id="main"/>` from `index.html`). The `app` component just displays the `:message` value, so you should see "hello from om" rendered. If you have `fighweel` running, you can change the message value, and it should be updated instantly.

And finally let's render our contacts with Om:

{% highlight clojure %}
(defn get-contacts []
  (go
    (let [response (<! (http/get "/contacts"))]
      (:body response))))

(defcomponent contact-comp [contact _]
  (render [_]
    (dom/li (str (:firstname contact) " " (:lastname contact)))))

(defcomponent app [data _]
  (will-mount [_]
    (go
      (let [contacts (<! (get-contacts))]
        (om/update! data :contacts contacts))))
  (render [_]
    (dom/div
      (dom/h2 (:message data))
      (dom/ul
        (om/build-all contact-comp (:contacts data))))))

{% endhighlight %}

So the `contact-comp` is just rendering a single contact. We use `om/build-all` to render all contacts visible in `:contacts` field in global state. And most tricky part - we use `will-mount` lifecycle method to get contacts from server when `app` component is about to be mounted to the DOM.

Again, in [this commit](https://github.com/pjagielski/modern-clj-web/commit/321890e677fdcd71ee85d1391aa8dda990deaf41) should be a working version in case of any problems.

And if you liked Om, I highly recommend [official tutorials](https://github.com/omcljs/om/wiki/Basic-Tutorial) and [Zero to Om](https://blog.stephanbehnke.com/zero-to-om/) series.
