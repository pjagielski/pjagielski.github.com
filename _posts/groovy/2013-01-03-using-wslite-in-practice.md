--- 
name: using-wslite-in-practice
title: Using WsLite in practice
time: 2013-01-03 13:38:00 +02:00
layout: post
---

## Why Groovy WsLite ? ##
I'm a huge fan of [Groovy WsLite](https://github.com/jwagenleitner/groovy-wslite) project for calling SOAP web services. Yes, in a real world you have to deal with those - big companies have huge amount of "legacy" code and are crazy about homogeneous architecture - only SOAP, Java, Oracle, AIX... 

But I also never been comfortable with XFire/CXF approach of web service client code generation. I wrote a bit about other posibilites in [this post](http://jagielu.com/2010/09/16/virtual-esb-application-integration). With JAXB you can also experience some freaky classloading errors - as Tomek described [on his blog](http://refaktor.blogspot.com/2012/09/classloader-problem-with-java-7-and.html). In a large commercial project the "the less code the better" principle is significant. And the code generated from XSD could look kinda ugly - especially more complicated structures like sequences, choices, anys etc.

Using WsLite with native Groovy concepts like `XmlSlurper` could be a great choice. But since it's a dynamic approach you have to be really careful - write good unit tests and log requests. Below are my few hints for using WsLite in practice.

## Unit testing ##
Suppose you have some invocation of WsLite SOAPClient (original WsLite example):
{% highlight ruby %}
def getMothersDay(long _year) {
    def response = client.send(SOAPAction: action) {
       body {
           GetMothersDay('xmlns':'http://www.27seconds.com/Holidays/US/Dates/') {
              year(_year)
           }
       }
    }
    response.GetMothersDayResponse.GetMothersDayResult.text()
}
{% endhighlight %}

How can the unit test like? My suggestion is to mock `SOAPClient` and write a simple helper to test that builded XML is correct. Example using great [SpockFramework](http://code.google.com/p/spock/): 

{% highlight ruby %}
void setup() {
   client = Mock(SOAPClient)
   service.client = client
}

def "should pass year to GetMothersDay and return date"() {
  given:
      def year = 2013
  when:
      def date = service.getMothersDay(year)
  then:
      1 * client.send(_, _) >> { Map params, Closure requestBuilder ->
            Document doc = buildAndParseXml(requestBuilder)
            assertXpathEvaluatesTo("$year", '//ns:GetMothersDay/ns:year', doc)
            return mockResponse(Responses.mothersDay)
      }
      date == "2013-05-12T00:00:00"
}
{% endhighlight %}

This uses a real cool feature of Spock - even when you mock the invocation with "any mark" (_), you are able to get actual arguments. So we can build XML that would be passed to `SOAPClient's` `send` method and check that specific XPaths are correct:

{% highlight ruby %}
void setup() {
    engine = XMLUnit.newXpathEngine()
    engine.setNamespaceContext(new SimpleNamespaceContext(namespaces()))
}

protected Document buildAndParseXml(Closure xmlBuilder) {
    def writer = new StringWriter()
    def builder = new MarkupBuilder(writer)
    builder.xml(xmlBuilder)
    return XMLUnit.buildControlDocument(writer.toString())
}

protected void assertXpathEvaluatesTo(String expectedValue,
                                      String xpathExpression, Document doc) throws XpathException {
    Assert.assertEquals(expectedValue,
            engine.evaluate(xpathExpression, doc))
}

protected Map namespaces() {
    return [ns: 'http://www.27seconds.com/Holidays/US/Dates/']
}
{% endhighlight %}

The [XMLUnit](http://xmlunit.sourceforge.net/) library is used just for `XpathEngine`, but it is much more powerful for comparing XML documents. The `NamespaceContext` is needed to use correct prefixes (e.g. `ns:GetMothersDay`) in your Xpath expressions.

Finally - the mock returns `SOAPResponse` instance filled with envelope parsed from some constant XML:
{% highlight ruby %}
protected SOAPResponse mockResponse(String resp) {
    def envelope = new XmlSlurper().parseText(resp)
    new SOAPResponse(envelope: envelope)
}
{% endhighlight %}
<div>&nbsp;</div>
## Request and response logging #
The WsLite itself doesn't use any logging framework. I usually handle it by adding own `sendWithLogging` method:
{% highlight ruby %}
private SOAPResponse sendWithLogging(String action, Closure cl) {
    SOAPResponse response = client.send(SOAPAction: action, cl)
    log(response?.httpRequest, response?.httpResponse)
    return response
}

private void log(HTTPRequest request, HTTPResponse response) {
    log.debug("HTTPRequest $request with content:\n${request?.contentAsString}")
    log.debug("HTTPResponse $response with content:\n${response?.contentAsString}")
}
{% endhighlight %}

This logs the actual request and response send through `SOAPClient`. 
But it logs only when invocation is successful and errors are much more interesting... So here goes `withExceptionHandler` method:

{% highlight ruby %}
private SOAPResponse withExceptionHandler(Closure cl) {
    try {
        cl.call()
    } catch (SOAPFaultException soapEx) {
        log(soapEx.httpRequest, soapEx.httpResponse)
        def message = soapEx.hasFault() ? soapEx.fault.text() : soapEx.message
        throw new InfrastructureException(message)
    } catch (HTTPClientException httpEx) {
        log(httpEx.request, httpEx.response)
        throw new InfrastructureException(httpEx.message)
    }
}
def send(String action, Closure cl) {
    withExceptionHandler {
        sendWithLogging(action, cl)
    }
}
{% endhighlight %}
<div>&nbsp;</div>
## XmlSlurper gotchas ##
Working with XML document with `XmlSlurper` is generally great fun, but is some cases could introduce some problems.
A trivial example is parsing an id with a number to `Long` value:
{% highlight ruby %}
def id = Long.valueOf(edit.'@id' as String)
{% endhighlight %}
The `Attribute` class (which `edit.'@id'` evaluates to) can be converted to String using `as` operator, but converting to `Long` requires using `valueOf`.

The second example is a bit more complicated. Consider following XML fragment:
{% highlight xml %}
<edit id="3">
   <params>
      <param value="label1" name="label"/>
      <param value="2" name="param2"/>
   </params>
   <value>123</value>
</edit>
<edit id="6">
   <params>
      <param value="label2" name="label"/>
      <param value="2" name="param2"/>
   </params>
   <value>456</value>
</edit>
{% endhighlight %}
We want to find id of `edit` whose `label` is **label1**. The simplest solution seems to be:

{% highlight ruby %}
def param = doc.edit.params.param.find { it['@value'] == 'label1' }
def edit = params.parent().parent()
{% endhighlight %}

But it doesn't work! The `parent` method returns multiple `edits`, not only the one that is parent of given `param`... 

Here's the correct solution:

{% highlight ruby %}
doc.edit.find { edit ->
    edit.params.param.find { it['@value'] == 'label1' }
}
{% endhighlight %}
<div>&nbsp;</div>
## Example ##
The example working project covering those hints is on [GitHub](https://github.com/pjagielski/wslite-example).
