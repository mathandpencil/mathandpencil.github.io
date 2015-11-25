---
layout: post
title: Optimizing Slow Django REST Framework Performance
author: Scott Stafford
comments: true
categories: django,djangorestframework,performance
description: Solve slow Django REST framework API performance problems by eager-loading data, instead of inefficient database querying due to nested serializers.
canonical: http://ses4j.github.io/2015/11/23/optimizing-slow-django-rest-framework-performance/
---

The [Django REST Framework](http://www.django-rest-framework.org/) (DRF for short) allows Django developers to build simple yet robust standards-based REST API for their applications.  We've used it successfully on a number of Django web design projects.  However, even seemingly simple, straightforward usage of the Django REST Framework and its [nested serializers][nested-serializers] can *kill performance* of your API endpoints. And that matters: if your web server is wasting its time inefficiently responding to a REST API call, it will drag the rest of the server's responsiveness down with it.

At it's root, the problem is called the **"N+1 selects problem"**; the database is queried once for data in a table (say, `Customers`), and then, one or more times *per customer* inside a loop to get, say, `customer.country.Name`.  Using the Django ORM, this mistake is easy to make.  Using DRF, it is hard *not* to make.

Luckily, there is a solution that can be used to fix this common Django REST Framework performance problem, without any major restructuring of the code.  It requires use of the underutilized [`select_related`][select_related]
and [`prefetch_related`][prefetch_related] methods on the Django ORM (and the newer [`Prefetch`][Prefetch] object as well) to perform what is called "eager loading".

This approach can have a **big** effect.  On the most recent project we applied this too, important API calls 
were taking *5-10 seconds* to return results.  After applying appropriate eager loading, the same
calls were well below 1s.  **Speedups of 20x or more are typical**.


## Why does Django REST Framework cause this issue so readily?

When you build a DRF view, you often want the return to include data from more than one related table.  Writing this is straightforward and [covered](http://www.django-rest-framework.org/api-guide/serializers/#dealing-with-nested-objects) in the DRF docs [in depth](http://www.django-rest-framework.org/api-guide/relations/). Unfortunately, **as soon as you use a nested relationship in your serializer, you risk crushing your performance**, and like so many performance problems, it often **only shows itself in production** with larger, real world data sets.

This happens because the Django ORM is *lazy*; it only fetches the minimum amount of data needed to respond to the current query.  It does not know you're about to ask a hundred (or ten thousand) times for the same or very similar data.  

And these days, when talking about database-backed websites, generally, the most important metric when determining site responsiveness is **number of trips to the database**.  

In DRF, we run into trouble whenever a serializer has a nested relationship, such as either of these:

{% highlight python %}
class CustomerSerializer(serializers.ModelSerializer):
    # This can kill performance!
    order_descriptions = serializers.StringRelatedField(many=True) 
    # So can this, same exact problem...
    orders = OrderSerializer(many=True, read_only=True) # This can kill performance!
{% endhighlight %}

The code inside DRF that populates either `CustomerSerializer` does this:

1. Fetch all `customers`. (Requires a round-trip to the database.)
2. For the first returned customer, fetch their `orders`.  (Requires another round-trip to the database.)
2. For the second returned customer, fetch its `orders`.  (Requires another round-trip to the database.)
2. For the third returned customer, fetch its `orders`.  (Requires another round-trip to the database.)
2. For the fourth returned customer, fetch its `orders`.  (Requires another round-trip to the database.)
2. For the fifth returned customer, fetch its `orders`.  (Requires another round-trip to the database.)
2. For the sixth returned customer, fetch its `orders`.  (Requires another round-trip to the database.)
3. ... you get the idea.  **Lets hope you don't have too many customers!**

And it quickly can get worse.  If your `OrderSerializer` itself has a nested relationship, you have a loop-inside-a-loop, and you're quickly in trouble, even for smallish amount of data.  As a rule of thumb, these days, on a modest traffic website, you can probably afford 50 trips to the database before you start getting into real trouble.  

## The basic approach to solving Django's "laziness"

Our approach to fixing this problem is called "eager loading".  Essentially, you warn the Django ORM ahead of time that you're going to ask it the same inane question over and over, "so get ready".  In the above example, simply do this before DRF starts fetching:

{% highlight python %}
queryset = queryset.prefetch_related('orders')
{% endhighlight %}

Then, when DRF makes the same call as above to serialize customers, this happens instead:

1. Fetch all `customers`. (Makes TWO round-trips to the database.  The first is for customers.  The second fetches all orders related to any of the fetched customers.)
2. For the first returned customer, fetch their `orders`.  (Does NOT require a trip to the database, we already fetch the needed data in step 1.)
2. For the second returned customer, fetch its `orders`.  (Does NOT require a trip to the database.)
2. For the third returned customer, fetch its `orders`.  (Does NOT require a trip to the database.)
2. For the fourth returned customer, fetch its `orders`.  (Does NOT require a trip to the database.)
2. For the fifth returned customer, fetch its `orders`.  (Does NOT require a trip to the database.)
2. For the sixth returned customer, fetch its `orders`. (Does NOT require a trip to the database.)
3. ... you get the idea.  **You can have LOTS of customers** and not have to keep waiting on trips to the database.

In short, the Django ORM "eagerly" asked for the data in step 1, then could supply the data requested in steps 2+ from it's local data cache.  Fetching data from the local data cache is essentially instantaneous when compared with the database round-trip, so we just got an enormous performance speedup in conditions when there are many customers.

## Standardizing a pattern to fix the Django REST Framework performance problem

We have settled on a common pattern to optimize this Django REST Framework performance problem.  Whenever a serializer will query nested fields, we add a new `@classmethod` called `setup_eager_loading` to the serializer, like so:

{% highlight python %}
class CustomerSerializer(serializers.ModelSerializer):
    orders = OrderSerializer(many=True, read_only=True)

    def setup_eager_loading(cls, queryset):
        """ Perform necessary eager loading of data. """
        queryset = queryset.prefetch_related('orders')
        return queryset
{% endhighlight %}

And then, wherever that serializer is going to be used, simply call `setup_eager_loading` on the 
queryset before the serializer is invoked, like so:

{% highlight python %}
customer_qs = Customers.objects.all()
customer_qs = CustomerSerializer.setup_eager_loading(customer_qs)  # Set up eager loading to avoid N+1 selects
post_data = CustomerSerializer(customer_qs, many=True).data
{% endhighlight %}

...or, if you have an `APIView` or a `ViewSet`, you can call `setup_eager_loading` in the `get_queryset` method:

{% highlight python %}
def get_queryset(self):
    queryset = Customers.objects.all()
    # Set up eager loading to avoid N+1 selects
    self.get_serializer_class().setup_eager_loading(queryset)  
    return queryset
{% endhighlight %}

## How do I write `setup_eager_loading`?

The hard part of solving this Django performance problem is becoming adept with how `select_related` and its friends work.  Here, we'll detail how each is used in the context of the Django ORM and the Django REST Framework.

   - `select_related`: The simplest eager loading tool in the Django ORM, for all **one-to-one** or **many-to-one relationships**, where you need data from the "one" parent object, such as a customer's company name.  This translates into a SQL join so the parent rows are fetched in the same query as the child rows.  [(See Official Documentation)][select_related]
   - `prefetch_related`: For more complex relationships where there are multiple rows per result (ie many=True), like **many-to-many** or **one-to-many** relationships, such as a customer's orders as above. This translates to a second SQL query on the related table, usually with a long `WHERE ... IN` clause to select only relevant rows. [(See Official Documentation)][prefetch_related]
   - `Prefetch`: Used for complex `prefetch_related` queries, such as filtered subsets. It can also be used to nest `setup_eager_loading` calls. [(See Official Documentation)][Prefetch]

### An example model with the appropriate eager loading

For our example, let's optimize the Django REST Framework-related performance problems of an imaginary event-planning website (which surprisingly parallels our ongoing project getfetcher.com).  We have a simple database structure:

{% highlight python %}
from django.contrib.auth.models import User

class Event:
    """ A single occasion that has many `attendees` from a number of organizations."""
    creator = models.ForeignKey(User)
    name = models.TextField()
    event_date = models.DateTimeField()
    
class Attendee:
    """ A party-goer who (usually) represents an `organization`, who may attend many `events`."""
    events = models.ManyToManyField(Event, related_name='attendees')
    organization = models.ForeignKey(Organization, null=True)

class Organization:
    name = models.TextField()
{% endhighlight %}

For this example, to fetch all events, our eager loading code would look like this:

{% highlight python %}
class EventSerializer(serializers.ModelSerializer):
    creator = serializers.StringRelatedField()
    attendees = AttendeeSerializer(many=True)
    unaffiliated_attendees = AttendeeSerializer(many=True)

    @classmethod
    def setup_eager_loading(cls, queryset):
        """ Perform necessary eager loading of data. """
        # select_related for "to-one" relationships
        queryset = queryset.select_related('creator')
        
        # prefetch_related for "to-many" relationships
        queryset = queryset.prefetch_related(
            'attendees',
            'attendees__organization')

        # Prefetch for subsets of relationships
        queryset = queryset.prefetch_related(
            Prefetch('unaffiliated_attendees', 
                queryset=Attendee.objects.filter(organization__isnull=True))
            )
        return queryset
{% endhighlight %}

When we make sure to invoke `setup_eager_loading` before using the EventSerializer, we will only have two large queries instead of N+1 smaller queries, and our performance will usually be MUCH better!

## Conclusion

Eager loading is a common performance optimization that has application well 
beyond Django REST Framework.

Any time you are querying nested relationships via an ORM, you should think about
setting up the proper eager loading.  In my experience, it is the most commonplace 
performance-related problem in modern small- and midsize web development.

In a followup blog post, I'll write some debugging strategies for figuring out elusive queries spawned by more complex Serializers and some more advanced usages of `Prefetch`.  

## References

 - [Django REST Framework](http://www.django-rest-framework.org/) documentation
 - Github issue to automatically perform this prefetch_related: [Automatically determine `select_related` and `prefetch_related` on ModelSerializer](https://github.com/tomchristie/django-rest-framework/issues/1964).
 - Tom Christie, the author of DRF, in a blog post about DRF performance, touches on the issue we've treated above in [Get your ORM lookups right](http://www.dabapps.com/blog/api-performance-profiling-django-rest-framework/).

Thank you for reading!

If you have a website with performance problems, I am available for consulting work. Please feel free to 
reach out to me at [{{ site.email }}](mailto:scott.stafford+gh@gmail.com) if you are looking for help.


[select_related]: https://docs.djangoproject.com/en/dev/ref/models/querysets/#django.db.models.query.QuerySet.select_related
[prefetch_related]: https://docs.djangoproject.com/en/dev/ref/models/querysets/#django.db.models.query.QuerySet.prefetch_related
[Prefetch]: https://docs.djangoproject.com/en/dev/ref/models/querysets/#prefetch-objects
[nested-serializers]: http://www.django-rest-framework.org/api-guide/relations/#nested-relationships
   
   