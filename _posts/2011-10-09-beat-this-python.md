---
title: Beat This, Python!
layout: posts
---

**I love Python. I love Objective-C.**

"Huh?!" I hear you thinking, "they are complete different, how can you like both"?

Well, nothing is perfect, even in a lovely relationship.
There are things, that drive me crazy in one — and things that are missing, form my point of view, in the other language

One thing I'd love to see in Python, that Objective-C has, is extending a class — Categories.
Every language should have functional tools like pythons List Comprehensions, but Objective-C hasn't.

But since Apple extended Objective-C by introducing Blocks to the underlaying C, it is now possible to have something similar to List Comprehensions — and this article is about how.
When people ask me, why I am such a big fan of Python, I usually refer to a sample Quicksort algorithm using List Comprehensions.
here we go:  



{% highlight python %}

def quicksort(alist):
  if len(alist) <= 1: return alist
    pivotelement = alist.pop()
    left  = [element for element in alist if element < pivotelement]
    right = [element for element in alist if element >= pivotelement]
  return quicksort(left) + [pivotelement] + quicksort(right)

{% endhighlight %}
<!--break-->


Compact and beautiful, isn't it?

But wait, there is a down-side in it:

With every recursive call of quicksort() we iterate over alist twice: Once to filter for smaller, once to filter for greater elements. With an average performance of O(n log n) this can be problematic for big lists.

But why do we iterate over alist twice? Can't we just say "…if less than pivot do… else do"?. We can — but not with List Comprehensions. For a very simple reason: They don't feature else-statements. And how could they? What need to be done in a else-clause is not predictable in such a language-level. Would we fill another array always? Do we want to be notified, if a else case is true? There are many cases the language architect cannot predict. One thing that actually could be done is to call a callback or a lambda with the element that raises the else case.

And that is exactly how I did my Objective-C array tools.  
Have a look at this signature:

{% highlight objc %}



- (NSArray *) arrayByPerformingBlock:(id   (^)(id element))performBlock
                 ifElementPassesTest:(BOOL (^)(id element))testBlock
                    elsePerformBlock:(void (^)(id element))elseBlock;

{% endhighlight %}
