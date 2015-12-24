---
title: Variadic Lists and Blocks
layout: posts
comments: true

---

Matt Gallagher [posted an great article about veriadic lists][1]. In his conclusion he writes

> In your own code, variadic methods should be used sparingly — passing variables in an NSArray or NSDictionary is safer (if slightly slower and syntactically more verbose) due to the fact that these classes do offer introspection.  
And I think he is right.
But now I found a situation, where it makes much more sense to have a veridic list of arguments rather a NSArray or NSDictionary.

I wrote a Category method on NSArray, that allows the user, to filter the array by blocks and create a dictionary with given keys.


``` objc

- (NSDictionary *) dictionaryByFilteringWithBlocks:(NSArray *)filterBlocks
                                           forKeys:(NSArray *)keys
```
<!--break-->

And it is used like  

``` objc

NSArray *blocks = [NSArray arrayWithObjects:
                ^BOOL(id element) {return [element hasPrefix:@"a"];},
                ^BOOL(id element) {return [element hasPrefix:@"c"];},
                ^BOOL(id element) {return [element hasPrefix:@"z"];},
           nil];

NSArray *keys = [NSArray arrayWithObjects:@"a",@"c", @"z", nil];
NSDictionary *dict = [array dictionaryByFilteringWithBlocks:blocks forKeys:keys];
```

it works as expected, but has two flaws:  

* the corresponding blocks and key are separated — I don't like that
* Xcode's code completion won't help you to create your blocks, as it will just tell you that you need two arrays


I think, that is reason enough to give veriadic methods a try — and here we go:  

``` objc
- (NSDictionary *) dictionaryByFilteringWithBlocksAndKeys:(BOOL (^)(id element))firstBlock, id firstKey,... NS_REQUIRES_NIL_TERMINATION;

-(NSDictionary *)dictionaryByFilteringWithBlocksAndKeys:(TestBlock)firstBlock, id firstKey,...
{
    NSMutableDictionary *results = [NSMutableDictionary dictionary];
    NSMutableArray *blocks = [[NSMutableArray alloc] initWithObjects:firstBlock, nil];
    NSMutableArray *keys = [NSMutableArray arrayWithObject:firstKey];

    va_list args;
    va_start(args, firstKey);
    while (YES) {
        TestBlock block = va_arg(args, TestBlock);

        if (! block)
            break;

        [blocks addObject:Block_copy(block)];
        id key = va_arg(args, id);  
        if (!key) [NSException raise:@"wrong number of arguments" format:@"n blocks need n keys", nil];
        [keys addObject:key];
    }
    va_end(args);

    for(id key in keys){
        NSMutableArray *filtered = [NSMutableArray array] ;
        [results setObject:filtered forKey:key];
        BOOL (^block)(id) = [blocks objectAtIndex:[keys indexOfObject:key]];
        for (id element in self){
            if (block(element)){
                [filtered addObject:element];
            }
        }
    }
    for(BOOL (^block)(id) in blocks){
        Block_release(block);
    }
    [blocks release];
    return  results;
}
```
``` objc
NSArray *array = [NSArray arrayWithObjects:@"a", @"aa", @"ab", @"cc", @"cd", @"dd", nil];
NSDictionary *dict = [array dictionaryByFilteringWithBlocksAndKeys:
                ^BOOL(id element) {return [element hasPrefix:@"a"];},@"a",
                ^BOOL(id element) {return [element hasPrefix:@"c"];},@"c",
             nil];
NSLog(@"%@", dict);
```

result:

``` objc

{
    a =     (
        a,
        aa,
        ab
    );
    c =     (
        cc,
        cd
    );
}
```

[1]: http://cocoawithlove.com/2009/05/variable-argument-lists-in-cocoa.html
