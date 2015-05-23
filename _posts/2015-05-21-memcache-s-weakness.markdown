---
layout: post
title:  "Memcached's Achilles Heel"
date:   2015-05-21 15:14:37
categories: technology memcached
---

I like Memcached, it's fairly easy to use, it's integrated to many platforms and languages, performances are pretty good, it's stable etc. etc.
We have been using it for a long time now, mostly to store counters, locks or ruby objects we need to communicate across multiple applications. Our main stack is ruby so we use the now deprecated memcache-client (Dali is the gem you should use) but that's not where the issue we stumbled upon came from.
No, the problem comes from inside, deep inside Memcached memory allocation logic.

Back story
---------------------

Let's explain a little bit what brought us to realize this Memcached issue. We started using queues to improve API response speed and load capacity. Our messaging API is now storing the transaction object that is created when receiving a request and queues it for our workers to process later on. This gives the API blazing fast response speed under heavy load. 

When we call `api.transaction.cache!` we trigger our caching process that basically uses a unique hash as a key and stores the transaction object in Memcached.

`MemCache.set(transaction_hash, self)`

Then, using rabbitMQ - bunny, we queue the hash and the worker processes it as soon as possible getting back the transaction object from Memcached using the hash key. 

`transaction = MemCache.get(transaction_hash)`

Then we normally process it. 

When it broke
---------------------

We couldn't decently start using this new logic for all traffic without having
conducted serious load-tests first. That's when we discovered an interesting
behavior.
We overloaded the API with about 200k requests and looked at the logs of the
workers to see if everything was going through. Sure enough, it wasn't. Every
other call to MemCache.get was apparently returning a nil value. Did we run out
of disk space on the Memcached server? No. Out of RAM? Any hardware issue?
Nope, nothing on the radar! WTH was happening?

The Magic
---------------------

So what kind of black magic is going on here? Turns out that Memcached
dynamically allocates free memory into pages of chunks until it runs out of
free pages. When a set request is sent to Memcached it tries to find an optimal
chunk size to store whatever you are trying to store. If no suitable chunk
exists it will take a free page and format it with suitable chunks. Once you
filled a page (all the chunks in that page are storing something), it will try
to create a new identical page and keep going.
After years of using Memcached for various operations we have allocated all the
memory of our Memcached server into pages of various chunk sizes. You can look
up your Memcached stats using the command line tool, giving you and output like
that:

{% highlight ruby %}
stats slabs
STAT 1:chunk_size 96
STAT 1:chunks_per_page 10922
STAT 1:total_pages 1
STAT 1:total_chunks 10922
STAT 1:used_chunks 820
STAT 1:free_chunks 10102
....
STAT 2:chunk_size 120
STAT 2:chunks_per_page 8738
STAT 2:total_pages 219
STAT 2:total_chunks 1913622
STAT 2:used_chunks 1913461
STAT 2:free_chunks 161
...
{% endhighlight %}

You can see the size of the chunks, the number of pages allocated with such
chunks and the number of chunks/free chunks.
We realized that our transaction was about 516 bytes, including the memcache
overhead the closest chunk was 600 and here is the report for these chunks:

{% highlight ruby %}
stats slabs
STAT 9:chunk_size 600
STAT 9:chunks_per_page 1747
STAT 9:total_pages 17
STAT 9:total_chunks 29699
STAT 9:used_chunks 29699
STAT 9:free_chunks 0
...
{% endhighlight %}

I think you got it now, out loadtest sending more than 200k requests in a few
seconds saturated the poor 29699 chunks of this size. Memcached's solution when
it runs out of chunks and no free pages are left to increase the number of this
type of chunks is to loop back to the first chunk of the first page and start
replacing its content, erasing previous transactions. We lost half of our
transaction objects in the process.

Wrapping up
---------------------

The really bad part of that issue is that we still have plenty of unused space
on Memcached, it is just not allocated to the chunks we need and is therefore
unusable. There are two solutions:

+    Throwing more hardware at Memcached so it can now have free pages to bump up
the number of those 600 bytes chunks.

+    Reset (wipe) the whole cache to let Memcached reallocate all the memory and
hope that this time we don't run into this issue again.

We chose the 3rd solution (I lied, there is a 3rd solution) and decided to
build a workaround for now and to move to couchbase in the future.

Useful link
---------------------

You can click
[here](https://www.adayinthelifeof.nl/2011/02/06/memcache-internals/) to learn more about Memcached's internals.
