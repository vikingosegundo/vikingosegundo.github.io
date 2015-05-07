---
title: Beat This, Python!
layout: posts
comments: true

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


If the testBlock returns YES, the element will be processed in the performBlock and returned
If the test fails the elseBlock will be executed with the element.

How would the initial example look like?

{% highlight objc %}

NSArray* quicksort(NSArray *array)
{
    if ([array count]<2) return array;
    id pivot = [array randomElement];
    NSMutableArray *array2= [NSMutableArray array];
    array = [array arrayByPerformingBlock:^id  (id element) { return element;}
                      ifElementPassesTest:^BOOL(id element) { return [element intValue] < [pivot intValue];}
                         elsePerformBlock:^    (id element) { if (element!=pivot) [array2 addObject:element];}
    ];
    return [[quicksort(array) arrayByAddingObject:pivot] arrayByAddingObjectsFromArray:quicksort(array2)];}
{% endhighlight %}

Not bad, huh?

Well, I mentioned in the beginning, that I like Categories, so lets do the last step and transform this C-style function into a Category.

{% highlight objc %}

-(NSArray *) quicksort
{
    if ([self count]<2) return self;

    id pivot = [self randomElement];
    NSMutableArray *array2= [NSMutableArray array];

    self = [self arrayByPerformingBlock:^id(id element)   { return element;}
                      ifElementPassesTest:^BOOL(id element) { return [element intValue] < [pivot intValue];}
                         elsePerformBlock:^    (id element) { if (element!=pivot) [array2 addObject:element];}
    ];
    return [[[self quicksort] arrayByAddingObject:pivot] arrayByAddingObjectsFromArray:[array2 quicksort]];
}
{% endhighlight %}


You may ask "How is that Quicksort implementation useful to me?" It isn't. It is just an example. Please do sorting with the APIs offered by Apple. I am pretty sure, that they are much better.
But If you a requirement where you need to filter a array in maybe two arrays, you can do this:

{% highlight objc %}
NSMutableArray *falsePositives = [NSMutableArray array];
NSArray *array = [NSArray arrayWithObjects:@"aa", @"ab",@"c",@"ad",@"dd", nil];
array = [array arrayByPerformingBlock:^id  (id element) {return [element stringByAppendingString:element];}
                  ifElementPassesTest:^BOOL(id element) {return [element hasPrefix:@"a"];}
                     elsePerformBlock:^    (id element) {[falsePositives addObject:element];}];
{% endhighlight %}


You see I use block to active a more functional programming style and to emulate List Comprehensions. But I go a bit further and introduce a else-clause. **Beat This, Python!**

{% highlight objc %}

#import <Foundation/Foundation.h>

@interface NSArray (FunctionalTools)
-(NSArray *) filter:(BOOL(^)(id element))filterBlock;

+(NSArray *) arrayByPerformingBlock:(id(^)(NSInteger index))performBlock
                withIndexFromRange:(NSRange)range;

-(NSArray *) arrayByPerformingBlock:(id  (^)(id element))performBlock;

-(NSArray *) arrayByPerformingBlock:(id  (^)(id element))performBlock
                ifElementPassesTest:(BOOL(^)(id element))testBlock;

-(NSArray *) arrayByPerformingBlock:(id   (^)(id element))performBlock
                ifElementPassesTest:(BOOL (^)(id element))testBlock
                   elsePerformBlock:(void (^)(id element))elseBlock;

-(NSArray *) arrayByPerformingBlock:(id   (^)(id element))performBlock
                ifElementPassesTest:(BOOL (^)(id element))testBlock
                   elsePerformBlock:(void (^)(id element))elseBlock
                           succeded:(void (^) ())succesBlock
                             failed:(void (^)(id element))failedBlock
                   stopOnFailedTest:(BOOL)stop;


-(void)performBlock:(void (^) (id element))block;
@end
{% endhighlight %}

{% highlight objc %}
#import "NSArray+FunctionalTools.h"

@implementation NSArray (FunctionalTools)
-(NSArray *) filter:(BOOL (^)(id))filterBlock
{
    NSMutableArray *filteredArray = [NSMutableArray array];
    for (id element in self)
        if (filterBlock(element))
            [filteredArray addObject:element];
    return [NSArray arrayWithArray:filteredArray];
}

+(NSArray *) arrayByPerformingBlock:(id (^)(NSInteger))performBlock withIndexFromRange:(NSRange)range
{
    NSMutableArray *array = [NSMutableArray array];
    for (NSUInteger i = range.location; i< range.location+range.length; ++i) {
        [array addObject:performBlock(i)];
    }
    return array;
}
-(NSArray *) arrayByPerformingBlock:(id  (^)(id element))performBlock
{
    return [self arrayByPerformingBlock:performBlock
                    ifElementPassesTest:^BOOL(id element) { return  YES;}];
}

-(NSArray *) arrayByPerformingBlock:(id(^)(id element))performBlock
                       ifElementPassesTest:(BOOL(^)(id element))testBlock
{

    return [self arrayByPerformingBlock:performBlock
                    ifElementPassesTest:testBlock
                       elsePerformBlock:^(id element){;}];
}

-(NSArray *) arrayByPerformingBlock:(id (^)(id))performBlock
           ifElementPassesTest:(BOOL (^)(id))testBlock
              elsePerformBlock:(void (^)(id))elseBlock
{
    NSMutableArray *array = [NSMutableArray array];
    for (id element in self)
        if(testBlock(element))
            [array addObject:performBlock(element)];
        else
            elseBlock(element);
    return array;
}


-(NSArray *) arrayByPerformingBlock:(id   (^)(id element))performBlock
                ifElementPassesTest:(BOOL (^)(id element))testBlock
                   elsePerformBlock:(void (^)(id element))elseBlock
                           succeded:(void (^)())succesBlock
                             failed:(void (^)(id element))failedBlock
                   stopOnFailedTest:(BOOL)stop

{
    NSMutableArray *array = [NSMutableArray array];
    for (id element in self){
        if(testBlock(element))
            [array addObject:performBlock(element)];
        else{
            elseBlock(element);
            if (stop) {
                failedBlock(element);
                break;
            }
        }
    }

    if (!stop) {
        succesBlock();
    }


    return array;

}

-(void) performBlock:(void (^)(id))block
{
    for(id element in self) {
        block(element);
    }
}

@end
{% endhighlight %}
