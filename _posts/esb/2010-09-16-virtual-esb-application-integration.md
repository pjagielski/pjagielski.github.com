--- 
name: virtual-esb-application-integration
title: Virtual ESB - application integration made painless with Apache Camel
time: 2010-09-16 22:56:00 +02:00
layout: post
---
Inspired by great <a href="http://www.infoq.com/presentations/Pragmatic-SOA">talk by Stefan Tilkov</a>, we recently debated at TouK about concept of <i>virtual ESB</i>. The idea is to accept the fact that integration of many applications ends with some kind of spaghetti architecture. Even when you provide an ESB layer, which pretends to be a box with clear in and out dependencies, when you look inside it, you will see the same spaghetti as before. <br />The <i>virtual ESB</i> is then a set of common practices and conventions to integrate systems together, at an application level, even with no bus between.<br /><br />OK, so you try to convince your application developers to implement some webservice-based integration with another application. And what's their response? 'Why to do that? We have now to generate some classes by our JAXB tools, made complex mappings to our domain classes, handle all that faults, etc. What about debugging, all that messy and unreadable XML messages, you cannot just set a breakpoint and evaluate some expression by our IDE...'

The suggestion of Stefan is to use REST instead of WS-* for integration. It's not always possible in a real world, so our answer is to switch from complex JAX-WS frameworks such as CXF to a lightweight, but powerful integration framework - <a href="http://camel.apache.org">Apache Camel</a>. To avoid code generation and JAXB mappings and return to pure XML, which have great processing features. To use templating frameworks to generate XML responses. And to write mocks and unit tests, just like you do for your DAO and service layers.<br /> 

<h1>XML is not evil</h1>
XML may be too verbose and complicated, but it has great tools to process it - XPath and XQuery, which evaluation in Camel is really easy and straightforward. They are integrated within Camel's <i>expression</i> concept, so whole API is adapted to use XPath. <br />Our most common case is to bind bean method arguments to XPath expressions evaluated on incoming XML messages:

{% highlight java %}
public List findCustomers(@XPath(name="//lastName") String lastName) {
    return getJdbcTemplate().queryForList("select * from customer where last_name = ?", new Object[]{lastName});
}
{% endhighlight %}

Then, you can route your XML message to any bean in Spring context - Camel will evaluate XPath's for you and inject results in method parameters:
{% highlight java %}
  from("direct:someXMLService") // XML message on input
  .to("bean:customerService") // String parameters on input, list on output
{% endhighlight %}
You can also make your own XPath annotation (e.g. adding namespaces support) by subclassing `DefaultAnnotationExpressionFactory`: (implemented by Maciek Próchniak)<br />

{% highlight java %}
public class XpathExpressionAnnotationFactoryImproved extends DefaultAnnotationExpressionFactory {
    @SuppressWarnings("unchecked")
    @Override
        public Expression createExpression(CamelContext camelContext, Annotation annotation, LanguageAnnotation languageAnnotation, Class expressionReturnType) {
        ... // return an XPathBuilder with injected namespaces
    }
}
{% endhighlight %}
{% highlight java %}
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.FIELD, ElementType.METHOD, ElementType.PARAMETER})
@LanguageAnnotation(language = "xpath", factory = XpathExpressionAnnotationFactoryImproved.class)
public @interface MyXPath {
    String value();
    // You can add the namespaces as the default value of the annotation
    NamespacePrefix[] namespaces() default @NamespacePrefix(prefix = "sch", uri = "http://touk.pl/common/customer/schema");
    Class<?> resultType() default String.class;
}
{% endhighlight %}     
{% highlight java %}
public List findCustomers(@MyXPath(name="//sch:lastName") String lastName) {...}
{% endhighlight %}

Another technique (also 'invented' by Maciek) is combining XPath's local-name function with current node (\*), which returns the name of root element of XML message. This could result in nice WSDL operation to Java bean method mapping without using any external tools. Consider following example:
{% highlight java %}
Namespaces ns = new Namespaces("sch", "http://touk.pl/common/customer");
from("direct:someXMLService") // XML message on input
    ... // some normalization - e.g. removing 'request' suffix
    .setProperty("operation").xpath("local-name(/*)", String.class, ns))
    .recipientList().simple("bean:customerService?method=${property.operation}");
{% endhighlight %}

Camel gets input name of root element from incoming request and sets it as an <i>operation</i> property. Then, by another nice feature of Camel - simple expression language, we are able to route message to particular method in our (Spring) bean. With this technique, we can map <i>findCustomersRequest</i> element in XML to <i>findCustomers</i> method in Java. And we could use XPath to parameter binding to extract parameters of this method from XPath expressions. All this without using JAXB, Axis/CXF, code generation etc. :)

<h1>Groovy XML features</h1>
Another usable set of tools based on XPath capabilities, it the Groovy language XML builders. You can take a look at <a href="http://groovy.codehaus.org/Processing+XML">Groovy documentation</a> or <i>Groovy in Action</i> book for some self-explaining examples. Imagine a case we had recently in one of our projects: you have to call two services, each returning a list of results (e.g. customers), and then match them to eliminate duplicates (e.g. by matching both first and last name). With JAXB, you will have to traverse first list, build a map of names pointing to customer objects and then add those from second list, that are no already in the map. With Groovy's XMLBuilder you can do a bit simpler, and much more readable:<br />

{% highlight ruby %}
def doc = parse(...)
    use(DOMCategory) {
        def customersFromSysA = doc[sch.sysAResponse].'*'[sch.customer]
        def customersFromSysB = doc[sch.sysBResponse].'*'[sch.customer]
        
        def notMatchingCustomers = customersFromSysB.findAll { customerB -> 
            def matcher = matchingClosure.curry(customerB)
            customersFromSysA.findAll(matcher) as boolean   
        }
        notMatchingCustomers.each { customersFromSysA.add(it) } 
    }
    
    def matchingClosure = { customerB, customerA ->
        customerB.'@firstName' == customerA.'@firstName && 
        customerB.'@lastName' == customerA.'@lastName
    }
{% endhighlight %}
(I assume, that you are familiar how to merge few responses in Camel - if not, read <a href="http://mcl.jogger.pl/2010/08/26/complex-flows-with-apache-camel/">this article wrote by Marcin</a>)<br />So, you can use Groovy's <i>findAll</i> method to iterate the customers from system B, leaving only those, which are not in results from system A. And you can use closure to define conditions of customer matching. <br />Finally, you can modify XML messages in Groovy, in our example - by adding unique customers from system B to results from system A.

<h1>Templating with Velocity</h1>
But how do you generate whole XML response from scratch, without JAXB mapping, which is most commonly used for that?<br />Idea it to use tools traditionally used to generating HTML responses - e.g. templating. Here is example of Velocity template:

{% highlight xml %}
<sch:${exchange.properties.operation}Response xmlns:sch="http://touk.pl/common/customer/schema">
    <sch:customers>
    #foreach( $customer in $exchange.properties.customers )
       <sch:customer #if ($customer.firstName) firstName="$customer.firstName" #end 
 ... 
       >
    #end
    </sch:customers>
</sch:${exchange.properties.operation}Response>
{% endhighlight %}

As you can see, implementing (web)services could be done well and easy, by using the right tools, like Apache Camel. You can embed it in your existing application or use an integration application platform (which is not a synonym for ESB) like Apache Servicemix 4.<br />WebServices are here an implementation detail. You can easily switch to REST if you are huge REST-enthusiast ;)
