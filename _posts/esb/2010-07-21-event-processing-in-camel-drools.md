--- 
name: event-processing-in-camel-drools
title: Event processing in camel-drools
time: 2010-07-21 13:38:00 +02:00
layout: post
---
In a <a href="http://touk.pl/blog/?p=146">previous post about camel-drools</a> I've introduced <a href="https://github.com/TouK/camel-drools">camel-drools component</a> and implemented some simple task-oriented process using rules inside Camel route. Today I'll show how to extend this example by adding <a href="http://en.wikipedia.org/wiki/Complex_event_processing">event processing</a>.<br /><br />So how to describe an event? Each event occur at some time and lasts for some duration, events happen in some particular order. We have then a 'cloud of events' from which we want to identify those, which form some interesting correlations. And here the usage of Drools becomes reasonable - we don't have to react for each event, just describe set of rules and consequences for those interesting correlations. Drools engine will find them and fire matching rules.<br /><br />Suppose our system has to monitor execution of task assigned to users. After a task is created, user has 10 days to complete it. When he doesn't - an e-mail remainder should be sent.

Rule definition may look like this:
{%highlight ruby%}
import org.apache.camel.component.drools.stateful.model.*
global org.apache.camel.component.drools.CamelDroolsHelper helper

declare TaskCreated
    @role( event )
    @expires( 365d )
end


declare TaskCompleted
    @role( event )
    @expires( 365d )
end

rule "Task not completed after 10 days"
    when
       $t : TaskCreated()
       not(TaskCompleted(name==$t.name, this after [-*, 10d] $t))
    then
       helper.send("direct:escalation", $t.getName());
end 
{%endhighlight%}

As you can see, there are two types of events: TaskCreated - when system assigns task to users, and TaskCompleted - when user finishes task. We correlate those two by the 'name' property.<br />Firstly, we need to declare our model classes as events by adding @role(event) and @expires annotations.<br />Then we describe rule: 'when there are is no TaskCompleted event after 10 days of TaskCreated event, send task name to direct:escalation route'. Again, this could be example of declarative programming - we don't event have to specify actual names of tasks, just correlate TaskCreated with TaskCompleted events by name.<br /><br />In this example, I used 'after' temporal operator. For description of others - see <a href="http://downloads.jboss.com/drools/docs/5.0.1.26597.FINAL/drools-fusion/html/ch02.html#d0e558">Drools Fusion documentation</a>.

And finally, here is JUnit test code snippet:
{%highlight java%}
public class TaskEventsTest extends GenericTest {

    DefaultCamelContext ctx;

    @Test
    public void testCompleted() throws Exception {
        insertAdvanceDays(new TaskCreated("Task1"), 4);
        assertContains(0);
        insertAdvanceDays(new TaskCompleted("Task1"), 4);
        advanceDays(5);
        assertContains(0);
    }

    @Test
    public void testNotCompleted() throws Exception {
        insertAdvanceDays(new TaskCreated("Task1"), 5);
        assertContains(0);
        advanceDays(5);
        assertContains("Task1");
    }

    @Test
    public void testOneNotCompleted() throws Exception {
        ksession.insert(new TaskCreated("Task1"));
        insertAdvanceDays(new TaskCreated("Task2"), 5);
        assertContains(0);
        insertAdvanceDays(new TaskCompleted("Task1"), 4);
        assertContains(0);
        advanceDays(1);
        assertContains("Task2");
        advanceDays(10);
        assertContains("Task2");
    }
    
    @Override
    protected void setUpResources(KnowledgeBuilder kbuilder) throws Exception {
        kbuilder.add(new ReaderResource(new StringReader(
                IOUtils.toString(getClass()
                 .getResourceAsStream("/stateful/task-event.drl")))), 
                 ResourceType.DRL);
    }
    
    @Override
    public void setUpInternal() throws Exception {
        this.ctx = new DefaultCamelContext();
        CamelDroolsHelper helper = new CamelDroolsHelper(ctx, 
                new DefaultExchange(ctx)) {
            public Object send(String uri, Object body) {
                sentStuff.add(body.toString());
                return null;
            };
        };
        ksession.setGlobal("helper", helper);
    }
}
{%endhighlight%}

You can find source code for this example <a href="https://github.com/TouK/camel-drools/blob/master/src/test/resources/stateful/task-event.drl">here</a>.
