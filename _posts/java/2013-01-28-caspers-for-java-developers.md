--- 
name: casperjs-for-java-developers
title: CasperJS for Java developers
time: 2013-01-28 21:35:00 +01:00
layout: post
tags: java, casperjs, angularjs, grails
---
## Why CasperJS ##
Being a Java developer is kinda hard these days. Java [may not be dead yet](http://1.bp.blogspot.com/-GLkCsR5eEIA/TXELWvpXsEI/AAAAAAAAACc/Oym-S4t7nSc/s1600/java-is-dead.png), but when keeping in sync with all the hipster JavaScript frameworks could make us feel a bit outside the playground. It's even hard to [list](http://devrates.com/project/list?query=%5Bjavascript%5D) [JavaScript](http://jster.net/) [frameworks](http://todomvc.com/) with latest releases on one website.

In my current project, we are using [AngularJS](http://angularjs.org/). It'a a nice abstraction of MV* pattern in frontend layer of any web application (we use Grails underneath). [Here](http://thesmithfam.org/blog/2012/12/02/angularjs-is-too-humble-to-say-youre-doing-it-wrong/) is a nice article with an 8-point Win List of Angular way of handling AJAX calls and updating the view. So it's not only a funny new framework but a truly helper of keeping your code clean and neat.

But there is also another area when you can put helpful JS framework in place of plan-old-java one - functional tests. Especially when you are dealing with one page app with lots of asynchronous REST/JSON communication. 

## Selenium and Geb ##

In Java/JVM project the typical is to use [Selenium](http://seleniumhq.org/) with some wrapper like [Geb](http://www.gebish.org/). So you start your project, setup your CI-functional testing pipeline and... after 1 month of coding your tests stop working and being maintainable. The frameworks itselves are not bad, but the typical setup is so heavy and has so many points of failure that keeping it working in a real life project is really hard.

Here is my list of common myths about Selenium:
* *It allows you to record test scripts via handy GUI* - maybe some static request/response sites. In modern web applications with asynchronous REST/JSON communication your tests must contain a lot of "waitFor" statements and you cannot automate where these should be included.
* *It allows you to test your web app against many browsers* - don't try to automate IE tests! You have to manually open your app in IE to see how it actually bahaves!
* *It integrates well with continuous integration servers like Jenkins* - you have to setup Selenium Grid on server with X installed to run tests on Chrome or Firefox and a Windows server for IE. And the headless HtmlUnit driver lacks a lot of JS support.

So I decided to try something different and introduce a bit of JavaScript tooling in our project by using CasperJS.

## Introduction ##
[CasperJS](http://casperjs.org) is simple but powerful navigation scripting & testing utility for [PhantomJS](http://phantomjs.org) - scritable headless WebKit (which is an rendering engine used by Safari and Chrome). In short - CasperJS allows you to navigate and make assertions about web pages as they'd been rendered in Google Chrome. It is enough for me to automate the functional tests of my application.

If you want a gentle introduction to the world of CasperJS I suggest you to read:
* [Official website](http://casperjs.org), especially [installation guide](http://casperjs.org/installation.html) and [API](http://casperjs.org/api.html#casper)
* [Introductionary article](https://nicolas.perriault.net/code/2012/introducing-casperjs-toolkit-phantomjs/) from CasperJS creator **Nicolas Perriault**
* [Highlevel testing with CasperJS](http://kvz.io/blog/2012/11/03/highlevel-testing-with-casperjs/) by **Kevin van Zonneveld**
* [grails-angular-scaffolding](https://github.com/robfletcher/grails-angular-scaffolding) plugin by **Rob Fletcher** with some working CasperJS [tests](https://github.com/robfletcher/grails-angular-scaffolding/tree/master/test/apps/grails-ng)

## Full example  ##
I run my test suite via following script:
{% highlight bash %}
casperjs test --direct --log-level=debug --testhost=localhost:8080 --includes=test/casper/includes/casper-angular.coffee,test/casper/includes/pages.coffee test/casper/specs/
{% endhighlight %}

casper-angular.coffe
{% highlight coffeescript %}
casper.test.on "fail", (failure) ->
    casper.capture(screenshot)

testhost   = casper.cli.get "testhost"
screenshot = 'test-fail.png'

casper
    .log("Using testhost: #{testhost}", "info")
    .log("Using screenshot: #{screenshot}", "info")

casper.waitUntilVisible = (selector, message, callback) ->
    @waitFor ->
        @visible selector
    , callback, (timeout) ->
        @log("Selector [#{selector}] not visible, failing")
        withParentSelector selector, (parent) ->
            casper.log("Output of parent selector [#{parent}]")
            casper.debugHTML(parent)
        @echo message, "RED_BAR"
        @capture(screenshot)
        @test.fail(f("Wait timeout occured (%dms)", timeout))

withParentSelector = (selector, callback) ->
    if selector.lastIndexOf(" ") > 0
       parent = selector[0..selector.lastIndexOf(" ")-1]
       callback(parent)
{% endhighlight %}

Sample pages.coffee:
{% highlight coffeescript %}
x = require('casper').selectXPath

class EditDocumentPage

    assertAt: ->
        casper.test.assertSelectorExists("div.customerAccountInfo", 'at EditDocumentPage')

    templatesTreeFirstCategory: 'ul.tree li label'

    templatesTreeFirstTemplate: 'ul.tree li a'

    closePreview: '.closePreview a'

    smallPreview: '.smallPreviewContent img'

    bigPreview: 'img.previewImage'

    confirmDelete: x("//div[@class='modal-footer']/a[1]")

casper.editDocument = new EditDocumentPage()
{% endhighlight %}

End a test script:
{% highlight coffeescript %}
testhost = casper.cli.get "testhost" or 'localhost:8080'

casper.start "http://#{testhost}/app", ->
    @test.assertHttpStatus 302
    @test.assertUrlMatch /\/fakeLogin/, 'auto login'
    @test.assert @visible('input#Create'), 'mock login button'
    @click 'input#Create'

casper.then ->
    @test.assertUrlMatch /document#\/edit/, 'new document'
    @editDocument.assertAt()
    @waitUntilVisible @editDocument.templatesTreeFirstCategory, 'template categories not visible', ->
        @click @editDocument.templatesTreeFirstCategory
        @waitUntilVisible @editDocument.templatesTreeFirstTemplate, 'template not visible', ->
            @click @editDocument.templatesTreeFirstTemplate

casper.then ->
    @waitUntilVisible @editDocument.smallPreview, 'small preview not visible', ->
        # could be dblclick / whatever
        @mouseEvent('click', @editDocument.smallPreview)

casper.then ->
    @waitUntilVisible @editDocument.bigPreview, 'big preview should be visible', ->
        @test.assertEvalEquals ->
            $('.pageCounter').text()
        , '1/1', 'page counter should be visible'
        @click @editDocument.closePreview

casper.then ->
    @click 'button.cancel'
    @waitUntilVisible '.modal-footer', 'delete confirmation not visible', ->
        @click @editDocument.confirmDelete

casper.run ->
    @test.done()
{% endhighlight %}

Here is a list of CasperJS features/caveats used here:
* Using [CoffeeScript](http://coffeescript.org/) is a huge win for your test code to look neat
* When using [casper test](http://casperjs.org/testing.html#casper-test-command) command, beware of different (than above articles) logging setup. You can pass `--direct --log-level=debug` from commandline for best results. Logging is essential here since Phantom often exists without any error and you do want to know what just happened.
* Extract your helper code into separate files and include them by using `--includes` switch.
* When passing server URL as a commandline switch remember that in CoffeeScript variables are not visible between multiple source files (unless getting them via window object)
* It's good to override standard waitUntilVisible with capting a screenshot and making a proper `log` statement. In my version I also look for a parent selector and `debugHTML` the content of it - great for debugging what is actually rendered by the browser.
* Selenium and Geb have a nice concept of [Page Objects](http://code.google.com/p/selenium/wiki/PageObjects) - an abstract models of pages rendered by your application. Using CoffeeScript you can write your own classes, bind selectors to properties and use then in your code script. Assigning the objects to casper instance will end up with quite nice syntax like `@editDocument.assertAt()`.
* There is some issue with CSS `:first` and `:last` selectors. I cannot get them working (but maybe I'm doing something wrong?). But in CasperJS you can also use XPath selectors which are fine for matching n-th child of some element (`x("//div[@class='modal-footer']/a[1]")`).

Working with CasperJS can lead you to a few hour stall, but after getting things working you have a new, cool tool in your box!
