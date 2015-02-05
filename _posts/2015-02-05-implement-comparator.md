---
title: Implementing Comparator Sorting
layout: posts
comments: true

---


This shows how to implement a block based sorting with the quicksort algorithm.

in response to this [Stackoverflow question][1]


[1]: http://stackoverflow.com/questions/28348022/how-is-the-sortedarrayusingcomparator-method-written-in-objective-c/28349545#28349545

<!--break-->

### NSArray+FunctionalTool

{% highlight objc %}

#import <Foundation/Foundation.h>

typedef id (^VSTestBlock)(id element);
@interface NSArray (FunctionalTools)
- (NSArray*)filter:(BOOL(^)(id element))filterBlock;

+(NSArray *)arrayByPerformingBlock:(id(^)(NSInteger index))performBlock
        withIndexFromRange:(NSRange)range;

- (NSArray *)arrayByPerformingBlock:(id  (^)(id element))performBlock;

- (NSArray *)arrayByPerformingBlock:(id  (^)(id element))performBlock
        ifElementPassesTest:(BOOL(^)(id element))testBlock;

- (NSArray *)arrayByPerformingBlock:(id   (^)(id element))performBlock
        ifElementPassesTest:(BOOL (^)(id element))testBlock
           elsePerformBlock:(void (^)(id element))elseBlock;

- (NSArray *)arrayByPerformingBlock:(id   (^)(id element))performBlock
        ifElementPassesTest:(BOOL (^)(id element))testBlock
           elsePerformBlock:(void (^)(id element))elseBlock
               succeded:(void (^) ())succesBlock
               failed:(void (^)(id element))failedBlock
           stopOnFailedTest:(BOOL)stop;


-(NSSet *)setByPerformingBlock:(id  (^)(id element))performBlock
           ifElementPassesTest:(BOOL (^)(id element))testBlock;

- (NSDictionary *)dictionaryByFilteringWithBlocks:(NSArray *)filterBlocks forKeys:(NSArray *)keys;

-(NSArray *)arrayOfArraysWithColumnWidth:(NSUInteger)width;


-(void)performBlock:(void (^) (id element))block;
@end


{% endhighlight %}


{% highlight objc %}
//
//  NSArray+FuntionalTools.m
//  arraytools
//
//  Created by vikingosegundo on 28.08.11.
//  Copyright (c) 2011 vikingosegundo. All rights reserved.
//

#import "NSArray+FunctionalTools.h"

@implementation NSArray (FunctionalTools)
-(NSArray *)filter:(BOOL (^)(id))filterBlock
{
  NSMutableArray *filteredArray = [NSMutableArray array];
  for (id element in self)
    if (filterBlock(element))
      [filteredArray addObject:element];
  return [NSArray arrayWithArray:filteredArray];
}

+(NSArray *)arrayByPerformingBlock:(id (^)(NSInteger))performBlock withIndexFromRange:(NSRange)range
{
  NSMutableArray *array = [NSMutableArray array];
  for (NSUInteger i = range.location; i< range.location+range.length; ++i) {
    [array addObject:performBlock(i)];
  }
  return array;
}


- (NSArray *)arrayByPerformingBlock:(id  (^)(id element))performBlock
{
  return [self arrayByPerformingBlock:performBlock
          ifElementPassesTest:^BOOL(id element) { return  YES;}];
}


- (NSArray *)arrayByPerformingBlock:(id(^)(id element))performBlock
             ifElementPassesTest:(BOOL(^)(id element))testBlock
{

  return [self arrayByPerformingBlock:performBlock
               ifElementPassesTest:testBlock
             elsePerformBlock:^(id element){;}];
}


-(NSArray *)arrayByPerformingBlock:(id (^)(id))performBlock
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


- (NSArray *)arrayByPerformingBlock:(id   (^)(id element))performBlock
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

-(NSSet *)setByPerformingBlock:(id   (^)(id))performBlock
           ifElementPassesTest:(BOOL (^)(id))testBlock
{
    NSMutableSet *set = [NSMutableSet setWithArray:self];
    NSMutableSet *newSet = [NSMutableSet set];
    for (id element in set){
        if (testBlock(element)) {
            [newSet addObject:performBlock(element)];
        }
    }
    if ([newSet count]<1)
        return nil;
    return [NSSet setWithSet:newSet];
}

-(void)performBlock:(void (^)(id))block
{
  for(id element in self) {
    block(element);
  }
}


- (NSDictionary *)dictionaryByFilteringWithBlocks:(NSArray *)filterBlocks forKeys:(NSArray *)keys
{
  NSMutableDictionary *result = [NSMutableDictionary dictionaryWithCapacity:[keys count]];
  for (id key  in keys) {
    [result setObject:[NSMutableArray array] forKey:key];
  }
  for (id element in self) {
    for (NSUInteger i=0; i < [filterBlocks count]; i++) {
      BOOL (^block)(id element)  = [filterBlocks objectAtIndex:i];
      if (block(element))
        [(NSMutableArray *)[result objectForKey:[keys objectAtIndex:i]] addObject:element];
    }
  }
  return result;
}


-(NSArray *)arrayOfArraysWithColumnWidth:(NSUInteger)width
{
    NSAssert(width > 0, @"width need to be 1 or greater");

    NSMutableArray *mArray =[NSMutableArray array];
    [self enumerateObjectsUsingBlock:^(id obj, NSUInteger idx, BOOL *stop) {
        if (idx % width == 0) {
            [mArray addObject:[NSMutableArray array]];
        }
        [[mArray objectAtIndex:[mArray count]-1] addObject:obj];
    }];
    return mArray;
}


@end

{% endhighlight %}




### NSArray+RandomUtils

{% highlight objc %}

@interface NSArray (RandomUtils)
-(id)randomElement;

@end

@interface NSMutableArray (RandomUtils)
-(void)shuffle;
@end
{% endhighlight %}


{% highlight objc %}
#import "NSArray+RandomUtils.h"

@implementation NSArray (RandomUtils)

-(id)randomElement
{
  if ([self count] < 1) return nil;
  int randomIndex = arc4random_uniform((int)[self count]);
  return [self objectAtIndex:randomIndex];
}

@end
{% endhighlight %}



### Example Usage
{% highlight objc %}

//
//  main.m
//  arraycomparator
//
//  Created by Manuel Meyer on 05.02.15.
//  Copyright (c) 2015  All rights reserved.
//

#import <Foundation/Foundation.h>
#import "NSArray+FunctionalTools.h"
#import "NSArray+RandomUtils.h"

@interface NSArray (Comparator)
-(NSArray *)vs_sortedArrayUsingComparator:(NSComparisonResult(^)(id obj1, id obj2))comparator;
@end

@implementation NSArray (Comparator)
-(NSArray *) quicksortUsingComparator:(NSComparisonResult(^)(id obj1, id obj2))comparator
{
    NSArray *array = [self copy];

    if ([array count]<2) return [array copy];

    id pivot = [array randomElement];
    NSMutableArray *array2= [NSMutableArray array];

    array = [array arrayByPerformingBlock:^id(id element) { return element;}
                    ifElementPassesTest:^BOOL(id element) { return comparator(element, pivot) == NSOrderedAscending;}
                       elsePerformBlock:^    (id element) { if (element!=pivot) [array2 addObject:element];}
            ];
    return [[[array quicksortUsingComparator:comparator]
                                    arrayByAddingObject:pivot]
                                        arrayByAddingObjectsFromArray:[array2 quicksortUsingComparator:comparator]];
}

-(NSArray *)vs_sortedArrayUsingComparator:(NSComparisonResult(^)(id obj1, id obj2))comparator
{
    return [self quicksortUsingComparator:comparator];
}
@end


int main(int argc, const char * argv[]) {
    @autoreleasepool {
        NSArray *array = @[@4, @9, @2, @1, @7];

        array = [array vs_sortedArrayUsingComparator:^NSComparisonResult(id obj1, id obj2) {
            return  [obj1 compare:obj2];
        }];
    }
    return 0;
}
{% endhighlight %}
