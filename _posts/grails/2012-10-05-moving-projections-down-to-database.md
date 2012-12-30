--- 
name: moving-projections-down-to-database
layout: post
time: 2012-10-05 16:18:00 +02:00
title: Moving projections down to database layer in Grails
---

While developing a Grails application you often deal with aggregations. You have some report page where you sum (avg/whatever) up properties of objects in given time interval.

Grails (and Hibernate underneath) has great projection api, but using it in a real world could lead to some messy code. Introducing database views can be much more helpful.

# The problem #

Let's consider some reddit-like application: you have link and votes for each link. When displaying a list of top-rated links, you have to sum up all the votes to show how many points each link received. 

Sample criteria query to show best links may look like this: 

{% highlight java %}
Link.withCriteria {
    votes {
        projections {
            groupProperty("link")
            sum("score")
        }
    }
}
{% endhighlight %}

The code looks not bad. But what is the result returned? It's a **List of Arrays**, each with 2 values: a `Link` object and `score` value... So we must introduce a helper class (DTO?) for aggregating scores with given link and passing it to view model.

Ok but how to order by votes score? Just adding the `order("score","desc")` gives some messy errors. The correct solution (via [GRAILS-3655](http://jira.grails.org/browse/GRAILS-3655)) is:

{% highlight ruby %}
Link.withCriteria {
    createAlias('votes', 'v')
    projections {
        groupProperty('v.link')
        sum('v.score', 'score')
    }
    order 'score', 'desc'
}
{% endhighlight %}

Still readable, but far not intiutive.

And what if we want some more complicated features:
- to get links with no votes?
- to get only links from last 7 days
- to sort links by click count
- and so on...

It ends with writing a swiss-knife service method which receives a bunch of parameters or even a few closures on the input and produces a messy 
and not testable criteria query with projection builder.
  
# The solution #

So there come database views. Why not move all that projections to a
database layer? When working with Grails you often forget that RDBMS is
not a simple datastore which can persist and retrieve objects. But it
offers much more features which can lead you to much simpler code in
your Grails application.
  
But how do we create a view in development mode our H2-based-by-default
application?
  
Proceed with these steps:

-  remove ‘mem’ from datasource definition: `url="jdbc:h2:devDb..."`
-  change `dbCreate` to ‘update’
-  start application to execute Bootstrap and fill database with data
-  stop application
-  remove DB initialization from Bootstrap
-  open H2 console (or db-console on running app via Grails plugin)
-  navigate to `devDb` from your applications’ base directory

And we have now access to our H2 DB schema!
  
In our case we can define a view `v\_link\_statistics` as follows:

{% highlight sql %}
SELECT l.id, l.id as link_id, sum(nvl(v.score,0)), l.date_created
FROM link l LEFT JOIN vote v ON v.link_id = v.id 
GROUP by l.id;
{% endhighlight %}
  
And a domain class:

{% highlight ruby %}
class LinkStatistics { 
    static belongsTo = [link: Link]
    Integer score
    Date dateCreated
 
    static mapping = {
        table("v_link_statistics")
        link(fetch: FetchMode.EAGER)
        version(false)
    } 
}
{% endhighlight %}

What you have to pay attention here is:

-   the id column is required by Grails, we can just pass Link id to it
-   the nvl with left join lets us easily include links with no votes
-   we map this table to a view with mapping-table 
-   the link object is eagerly fetched to deal with n+1 selects problem,
    while we are displaying urls for the retrieved link list
-   we have to set version = false, versioning doesn’t make sense here

And now query is much simpler:
{% highlight ruby %}
  LinkStatistics.list(sort: 'score', order: 'desc')
{% endhighlight %}

Look at sorting multiple links with same score:
  
{% highlight ruby %}
LinkStatistics.withCriteria {
    createAlias('link','l')
    order('score', 'desc')
    order('dateCreated', 'desc')
    order('id', 'desc')
}
{% endhighlight %}

Sorting by one unique field (id) is important for pagination. As we exposed date\_created in our view, we don’t have to create alias to order by ‘link.dateCreated’ field.
