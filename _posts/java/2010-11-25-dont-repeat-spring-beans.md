--- 
name: dont-repeat-spring-beans
title: Don't repeat the Spring beans!
time: 2010-11-25 15:52:00 +01:00
layout: post
tags: java, esb, camel, spring, dao
---
The 'Don't repeat the DAO!' <a href="http://www.ibm.com/developerworks/java/library/j-genericdao.html">article</a> is a fundamental one for all Spring/ORM/Java world. The generic DAO pattern works well and is widely used in both commercial projects and open-source implementations. <br /><br />But what if you only need one class for all of your DAOs? In one of my projects I just needed to put all records from a database view to a file. I decided not to use an ORM, but just Spring's JdbcTemplate, configured by view name and column to use in ORDER BY section. So I started to write Spring XML wiring and realized.... that I'm repeating the Spring beans! Every declaration looked something like:<br />

{% highlight xml %}
<bean id="customerDAO" class="pl.touk.blog.GenericDao">
     <property name="table" value="V_CUSTOMERS"/>
     <property name="orderColumn" value="CUST_ID"/>
  </bean>
 
  <bean id="contractDAO" class="pl.touk.blog.GenericDao">
     <property name="table" value="V_CONTRACTS"/>
     <property name="orderColumn" value="CONTRACT_ID"/>
  </bean>
...
{% endhighlight %}

There were about 15 twin beans in my Spring XML files! That looked really ugly...

Googling a bit introduced to me concept of writing own Spring's `BeanFactoryPostProcessor`, in which I could instantiate DAOs and register them in CamelContext. My domain classes already had some annotations - I used  <a href="http://camel.apache.org/bindy.html">camel-bindy</a> component to specify output file format. So the idea was to write another annotation containing each DAO configuration:

{% highlight java %}
@GenericDao(type = "customer", table = "V_CUSTOMERS", orderColumn = "CUST_ID")
public class Customer { ... }
{% endhighlight %}
{% highlight java %}
@GenericDao(type = "contract", table = "V_CONTRACTS", orderColumn = "CONTRACT_ID")
public class Contract { ... }
{% endhighlight %}

The `BeanFactoryPostProcessor` implementation could look like this:<br />

{% highlight java %}
public class GenericDaoBeanFactoryPostProcessor implements BeanFactoryPostProcessor {
 
  public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
    Collection<Class<?>> classes = findClassesWithAnnotation(GenericDao.class); 
    for (Class<?> c : classes) {
      GeneratedFile gfa = c.getAnnotation(GenericDao.class);
      String type = gfa.type();
      GenericDaoImpl dao = new GenericDaoImpl();
      dao.setDataSource(dataSource);
      dao.setClazz(c);
      dao.setTable(gfa.table());
      dao.setOrderColumn(gfa.orderColumn());
      beanFactory.registerSingleton(type + "Dao", dao);
    }
  } 
}
{% endhighlight %}
So it seems to be rather easy. But what with `findClassesWithAnnotation` method? My first attempt was to use Camel's `DefaultPackageScanClassResolver` implementation, and it worked fine in my unit tests. But when I deployed the bundle within an OSGi container (Servicemix 4.0), no annotated classes were found at all... Looking inside `DefaultPackageScanClassResolver` implementation showed the reason:<br />

{% highlight java %}
// osgi bundles should be skipped
 if (url.toString().startsWith("bundle:") || urlPath.startsWith("bundle:")) {
   log.trace("It's a virtual osgi bundle, skipping");
   continue;
 } 
{% endhighlight %}

Loading classes from bundle is an OSGi container-specific matter, and Camel correctly doesn't implement it (more precisely: there is an OsgiPackageScanClassResolver implementation, but it uses OSGi APIs).<br /><br />The solution is to use `CamelContext's getPackageScanClassResolver()` method to get correct resolver. In unit tests it returns default one, in OSGi environment - container-aware implementation.<br />

{% highlight java %}
public class GeneratedFileBeanFactoryPostProcessor implements BeanFactoryPostProcessor {
  protected CamelContext camelContext;
 
  public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
    camelContext.getPackageScanClassResolver().findAnnotated(GenericDao.class, defaultPackage);
  ...
}
{% endhighlight %}

And now it works fine and Spring XML wirings look much better :)
Finally my route also uses `PackageScanClassResolver` to configure all endpoints:

{% highlight java %}
from("direct:start") 
   .recipientList(constant(makeEndpointsFromTypes())  // makes 'direct:type' endpoint for all object types
 
  for (String type: types) {
    from("direct:"+type).to("bean:"+type+"Dao?method=dumpAll")...
  }
{% endhighlight %}

PS. Sure, all of my classes could just subclass `GenericDaoImpl`, but I didn't want to mix database logic with the file structure logic. Using annotations feels a bit more like configuration, not implementation itself ;)<br /><br /></span>
