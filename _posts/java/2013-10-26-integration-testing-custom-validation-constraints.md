--- 
name: integration-testing-custom-validation-constraints
title: Integration testing custom validation constraints in Jersey 2
time: 2013-10-03 21:35:00 +01:00
layout: post
tags: java, jaxrs, jersey, jetty, tdd
---

I recently joined a team trying to switch a monolithic legacy system into set of RESTful services in Java. They decided to use latest 2.x version of Jersey as a REST container which was not a first choice for me, since I'm not a big fan of JSR-* specs. But now I must admit that JAX-RS 2.x is doing things right: requires almost zero boilerplate code, support auto-discovery of features and prefers convention over configuration like other modern frameworks. Since the spec is still young, it's hard to find good tutorials and kick-off projects with some working code. I created [jersey2-starter](https://github.com/pjagielski/jersey2-starter) project on GitHub which can be used as starting point for your own production-ready RESTful service. In this post I'd like to cover how to implement and integration test your own validation constraints of REST resources.

## Custom constraints ##
One of the issues which bothers me when coding REST in Java is littering your class model with annotations. Suppose you want to build a simple Todo list REST service, when using Jackson, validation and Spring Data, you can easily end up with this as your entity class:

{% highlight java %}
@Document
public class Todo {
    private Long id;
    @NotNull
    private String description;
    @NotNull
    private Boolean completed;
    @NotNull
    private DateTime dueDate;

    @JsonCreator
    public Todo(@JsonProperty("description") String description, @JsonProperty("dueDate") DateTime dueDate) {
        this.description = description;
        this.dueDate = dueDate;
        this.completed = false;
    }
    // getters and setters
}
{% endhighlight %}

Your domain model is now effectively blured by messy annotations almost everywhere. Let's see what we can do with validation constraints (`@NotNull`s). Some may say that you could introduce some DTO layer with own validation rules, but it conflicts for me with pure REST API design, which stands that you operate on resources which should map to your domain classes. On the other hand - what does it mean that `Todo` object is **valid**? When you create a `Todo` you should provide a description and due date, but what when you're updating? You should be able to change any of description, due date (postponing) and completion flag (marking as done) - but you should provide at least one of these as valid modification. So my idea is to introduce custom validation constraints, different ones for creation and modification: 

{% highlight java %}
@Target({TYPE, PARAMETER})
@Retention(RUNTIME)
@Constraint(validatedBy = ValidForCreation.Validator.class)
public @interface ValidForCreation {
    //...
    class Validator implements ConstraintValidator<ValidForCreation, Todo> {
    /...
        @Override
        public boolean isValid(Todo todo, ConstraintValidatorContext constraintValidatorContext) {
            return todo != null
                && todo.getId() == null
                && todo.getDescription() != null
                && todo.getDueDate() != null;
        }
    }
}

@Target({TYPE, PARAMETER})
@Retention(RUNTIME)
@Constraint(validatedBy = ValidForModification.Validator.class)
public @interface ValidForModification {
    //...
    class Validator implements ConstraintValidator<ValidForModification, Todo> {
    /...
        @Override
        public boolean isValid(Todo todo, ConstraintValidatorContext constraintValidatorContext) {
            return todo != null
                && todo.getId() == null
                && (todo.getDescription() != null || todo.getDueDate() != null || todo.isCompleted() != null);
        }
    }
}

{% endhighlight %}

And now you can move validation annotations to the definition of a REST endpoint:

{% highlight java %}
@POST
@Consumes(APPLICATION_JSON)
public Response create(@ValidForCreation Todo todo) {...}

@PUT
@Consumes(APPLICATION_JSON)
public Response update(@ValidForModification Todo todo) {...}

{% endhighlight %}

And now you can remove those `NotNull`s from your model.

## Integration testing ##
There are in general two approaches to integration testing:

* test is being run on separate JVM than the app, which is deployed on some other integration environment
* test deploys the application programmatically in the setup block.

Both of these have their pros and cons, but for small enough services, I personally prefer the second approach. It's much easier to setup and you have only one JVM started, which makes debugging really easy. You can use a generic framework like [Arquillian](http://arquillian.org/) for starting your application in a container environment, but I prefer simple solutions and just use emdedded Jetty. To make test setup 100% production equivalent, I'm creating full Jetty's `WebAppContext` and have to resolve all runtime dependencies for Jersey auto-discovery to work. This can be simply achieved with Maven resolved from [Shrinkwrap](http://www.jboss.org/shrinkwrap) - an Arquillian subproject: 

{% highlight java %}
    WebAppContext webAppContext = new WebAppContext();
    webAppContext.setResourceBase("src/main/webapp");
    webAppContext.setContextPath("/");
    File[] mavenLibs = Maven.resolver().loadPomFromFile("pom.xml")
                .importCompileAndRuntimeDependencies()
                .resolve().withTransitivity().asFile();
    for (File file: mavenLibs) {
        webAppContext.getMetaData().addWebInfJar(new FileResource(file.toURI()));
    }
    webAppContext.getMetaData().addContainerResource(new FileResource(new File("./target/classes").toURI()));

    webAppContext.setConfigurations(new Configuration[] {
        new AnnotationConfiguration(),
        new WebXmlConfiguration(),
        new WebInfConfiguration()
    });
    server.setHandler(webAppContext);
{% endhighlight %}

([this Stackoverflow thread](http://stackoverflow.com/questions/13222071/spring-3-1-webapplicationinitializer-embedded-jetty-8-annotationconfiguration) inspired me a lot here)

Now it's time for the last part of the post: parametrizing our integration tests. Since we want to test validation constraints, there are many edge paths to check (and make your code coverage close to 100%). Writing one test per each case could be a bad idea. Among the many solutions for JUnit I'm most convinced to the [Junit Params](https://code.google.com/p/junitparams/) by Pragmatists team. It's really simple and have nice concept of JQuery-like helper for creating providers. Here is my tests code (I'm also using builder pattern here to create various kinds of Todos):

{% highlight java %}

@Test
@Parameters(method = "provideInvalidTodosForCreation")
public void shouldRejectInvalidTodoWhenCreate(Todo todo) {
    Response response = createTarget().request().post(Entity.json(todo));

    assertThat(response.getStatus()).isEqualTo(BAD_REQUEST.getStatusCode());
}

private static Object[] provideInvalidTodosForCreation() {
    return $(
        new TodoBuilder().withDescription("test").build(),
        new TodoBuilder().withDueDate(DateTime.now()).build(),
        new TodoBuilder().withId(123L).build(),
        new TodoBuilder().build()
    );
}

{% endhighlight %}

OK, enough of reading, feel free to clone the [project](https://github.com/pjagielski/jersey2-starter) and start writing your REST services!
