---
layout: post
title:  "Mobile web love"
date:   2015-09-11
categories: mobile web development
---

Web development is awesome. No doubts about it. There is one major issue
though, that any web developer could rant about, sometimes for
hours...you guessed it...I'm talking about uneven browser support. We made
some progress in that regard, one could say we see the light at the end of the
tunnel. I don't want to be pessimistic or overly optimistic about it so I'll
just say that, no matter where we are, it still is an issue.

Now most mobile developer can tell you about Android fragmentation and how
annoying it is to deal with different resolutions. Well guess what? Yeah you
got it, when you develop web pages for mobile you get to deal with both issues
at the same time! Isn't that great?

Anyways, I just felt like ranting after I discovered that using in-line
JavaScript in a link tag was not being executed by the Android browser of the 
HTC Hero device I was testing on, while working for 100% of the rest of our
testing fleet.

This is a Merb (yes, it is old) app with haml views:

{% highlight haml %}
    =link_to "Pay #{formatted_price(@item[:price])}", "#", :onclick => "document.getElementById('form_id').submit()", :class => 'confirm_btn'
{% endhighlight %}

This approach isn't working, the browser just appends a '#' at the end of the url and nothing happens. The form isn't submitted, the page isn't refreshed. Debugging was quite easy, clearly the javascript wasn't getting executed. I replaced it with the following:

{% highlight haml %}
    %input{:type => 'submit', :alt => "Pay #{formatted_price(@formatted_price(@item[:price]))}", :value => "Pay #{formatted_price(@formatted_price(@item[:price]))}", :class => 'confirm_btn'}
{% endhighlight %}

This was ugly legacy code anyways and a form should be validated with a button when possible, so it forced me to improve (though only slightly) the quality of that code.
So really it's no big deal but it's just another daily testimony of the work left to do to get to a point where a developer can be confident it will work on any device when pushing code live.

Check out the [Android fragmentation][andro_frag] or [Browser fragmentation][browser_frag] to see why it is important that somewhere along the lines we follow a standard!

[andro_frag]: http://thenextweb.com/insider/2015/08/05/this-is-what-android-fragmentation-looks-like-in-2015/
[browser_frag]: http://www.sitepoint.com/browser-trends-january-2015-ie8-usage-triples/
