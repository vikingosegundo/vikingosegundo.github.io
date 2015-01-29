---
title: Once and For All — Data Source Adapter Pattern
layout: default
---

In iOS and Mac development a plague is spreading around the world.  
Many view controller become so heavy and obese that we must expect our devices
to suffer from Type-II Diabetes. This disease is super-contangeous and it's
ground zero is well known: [developer.apple.com][apple]

### What are the symptoms of this syndrome?
Among others others they are the usual suspects when it come to obisity:

* hard to move
* unflexibel
* hard to breed
* no sex appeal

### Where and how did the current outbreak start?

It is not known by now where the gems that transport this disease came to
existence, but once they reached [developer.apple.com][apple], it broke loose.  
Since than it is spreading through the main transport infrastructure like Apple's
sample codes, mailinglists, stackoverflow.com, Books,…
<!--break-->

### How do I know my view controllers are infected?

Simple answer: If you are asking this ion they probably are already.  
Infected view controllers are showing symptoms known from ADHD: They want to do
everything at the same time and are failing in focussing on one thing —
controlling the view.

If you find your view controller implementing a bunch of protocols, it is
certain: it is infected

### The Cure

There are quite a few possible cures, excotic ones like [Intentions][intentions]
(possible by-effect: Massive Storyboards), moving MVC to MVVM, …  

The cure that I want to present here is so simple that I classify it as home
remedy

### Data Source Adapter Pattern

Following the Single Responsibility Principle a view controller should only do one
and exactly one thing: Controlling one view (that actually might be formed by
  a view hierachy). It should *not* do any of the following: perform network
  request, parse data, fetch from disk, perform computation as redering and
  resizing images,…

But how can we achieve this? Simply by applying our knowledge form CS101 —
patterns!

If it is the single responsibility of a viewcontroller to controll the views, than
it should be single responsibility of a data source should be to provide the data
that is about to be displayed — but not to fetch that data from somewhere. That
would be responsibility of a data fetcher object.

So by now we identified three classes we need:

* View Controller for handling views
* Data Source to provide the view controller with data to display
* Data Fetcher to get/render/calculate that data

By implementing the adapter pattern for Source and fetcher we are gaining simple
to use and re-use classes.

### Demo

I want to demonstrate a simple UITableView that has a twist: every section gets
it objetcs to display from another source.


{% highlight objc %}
#import <Foundation/Foundation.h>

@protocol OFADataFetcher <NSObject>

- (void)fetchSuccess:(void (^)(void))success;
- (void)fetchedData:(id)obj onDataFetcher:(id<OFADataFetcher>)dataFetcher;
- (void)fetchingDataFaildWithError:(NSError *)error onDataFetcher:(id<OFADataFetcher>)dataFetcher;
- (NSArray *)objects;
@end

{% endhighlight %}

The `OFADataFetcher` protocol describes the contract a data fetcher object must fulfill
to be usable by the data source.
{% highlight objc %}
- (void)fetchSuccess:(void (^)(void))success;
{% endhighlight %}

This method is called by the data source. The block is a call back mechanism. We could
 use dlegation instead.

 {% highlight objc %}
 - (void)fetchedData:(id)obj onDataFetcher:(id<OFADataFetcher>)dataFetcher;
 {% endhighlight %}
 Called in case of successful fetching.

 {% highlight objc %}
 - (void)fetchingDataFaildWithError:(NSError *)error onDataFetcher:(id<OFADataFetcher>)dataFetcher;
 {% endhighlight %}
 called if fetching failed.


As I said, I want to have different sources for the different sections of the
table view, but it can have only one data source. We have to have a tree of sources.

{% highlight objc %}
#import <UIKit/UIKit.h>
#import "OFASectionDataSource.h"
@protocol OFASectionDataSource;

@interface OFASectionedTableViewDataSource : NSObject <UITableViewDataSource, UITableViewDelegate>
@property (nonatomic, weak, readonly) UITableView *tableView;
@property(nonatomic, strong, readonly) NSArray    *sectionDataSources;


- (instancetype)initWithTableView:(UITableView *)tableView;
- (void)addSectionDataSource:(id<OFASectionDataSource>)sectionDataSource;
- (void)selectRowAtIndexPath:(NSIndexPath *)indexPath;

- (NSArray *)selectedObjects;
@end

{% endhighlight %}

----

{% highlight objc %}
- (void)addSectionDataSource:(id<OFASectionDataSource>)sectionDataSource;
{% endhighlight %}
allows us to add a data source that fulfills `OFASectionDataSource` for every section.  


{% highlight objc %}
#import "OFASectionedTableViewDataSource.h"

@interface OFASectionedTableViewDataSource ()
@property (nonatomic, weak) UITableView     *tableView;
@property(nonatomic, strong) NSMutableArray *sectionDataSources;

@end

@implementation OFASectionedTableViewDataSource

- (instancetype)initWithTableView:(UITableView *)tableView
{
  self = [super init];
  if (self) {
    _tableView            = tableView;
    _tableView.dataSource = self;
    _tableView.delegate   = self;
    _sectionDataSources   = [@[] mutableCopy];
  }
  return self;
}

- (void)addSectionDataSource:(id<OFASectionDataSource>)sectionDataSource
{
  [(NSMutableArray *)self.sectionDataSources addObject:sectionDataSource];
}

- (void)selectRowAtIndexPath:(NSIndexPath *)indexPath
{
  id<OFASectionDataSource> sectionDataSource = self.sectionDataSources[indexPath.section];
  if ([sectionDataSource respondsToSelector:@selector(selectRowAtIndexPath:)]) {
    [sectionDataSource selectRowAtIndexPath:indexPath];
  }
}

- (NSArray *)selectedObjects
{
  NSMutableArray *array = [@[] mutableCopy];
  [self.sectionDataSources enumerateObjectsUsingBlock:^(id < OFASectionDataSource > ds, NSUInteger idx, BOOL *stop) {
    if ([ds respondsToSelector:@selector(selectedObjects)])
    [array addObjectsFromArray:[ds selectedObjects]];
    }];
    return array;
  }

  - (NSInteger)numberOfSectionsInTableView:(UITableView *)tableView
  {
    return self.sectionDataSources.count;
  }

  - (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section
  {
    id<OFASectionDataSource> sectionDataSource = self.sectionDataSources[section];
    return [sectionDataSource tableView:tableView numberOfRowsInSection:section];
  }

  - (NSString *)tableView:(UITableView *)tableView titleForHeaderInSection:(NSInteger)section
  {
    id<OFASectionDataSource> sectionDataSource = self.sectionDataSources[section];
    return [sectionDataSource tableView:tableView titleForHeaderInSection:section];
  }

  - (NSString *)tableView:(UITableView *)tableView titleForFooterInSection:(NSInteger)section
  {
    id<OFASectionDataSource> sectionDataSource = self.sectionDataSources[section];
    return [sectionDataSource tableView:tableView titleForFooterInSection:section];
  }



  - (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath
  {
    id<OFASectionDataSource> sectionDataSource = self.sectionDataSources[indexPath.section];
    UITableViewCell       *cell             = [sectionDataSource tableView:tableView cellForRowAtIndexPath:indexPath];
    cell.editing = NO;
    return cell;
  }

  -(CGFloat)tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath
  {
    id<OFASectionDataSource> sectionDataSource = self.sectionDataSources[indexPath.section];
    if ([sectionDataSource respondsToSelector:@selector(tableView:heightForRowAtIndexPath:)]) {
      return [sectionDataSource tableView:tableView heightForRowAtIndexPath:indexPath];
    }
    return 44.0;
  }

  -(void)tableView:(UITableView *)tableView willDisplayCell:(UITableViewCell *)cell forRowAtIndexPath:(NSIndexPath *)indexPath
  {
    id<OFASectionDataSource> sectionDataSource = self.sectionDataSources[indexPath.section];
    if ([sectionDataSource respondsToSelector:@selector(tableView:willDisplayCell:forRowAtIndexPath:)]) {
      [sectionDataSource tableView:tableView willDisplayCell:cell forRowAtIndexPath:indexPath];
    }
  }

  - (NSArray *)sectionIndexTitlesForTableView:(UITableView *)tableView
  {
    NSMutableArray *sectionTitles = [@[] mutableCopy];
    [self.sectionDataSources enumerateObjectsUsingBlock:^(id < OFASectionDataSource > obj, NSUInteger idx, BOOL *stop) {
      if ([obj sectionIndexTitle])
      [sectionTitles addObject:[obj sectionIndexTitle](idx)];
      }];
      return sectionTitles;
    }

    - (BOOL)tableView:(UITableView *)tableView canMoveRowAtIndexPath:(NSIndexPath *)indexPath
    {
      id<OFASectionDataSource> sectionDataSource = self.sectionDataSources[indexPath.section];
      if ([sectionDataSource respondsToSelector:@selector(tableView:canMoveRowAtIndexPath:)]) {
        return [sectionDataSource tableView:tableView canMoveRowAtIndexPath:indexPath];
      }
      return NO;
    }

    - (void)tableView:(UITableView *)tableView moveRowAtIndexPath:(NSIndexPath *)sourceIndexPath toIndexPath:(NSIndexPath *)destinationIndexPath
    {
      id<OFASectionDataSource> sectionDataSource = self.sectionDataSources[sourceIndexPath.section];
      if ([sectionDataSource respondsToSelector:@selector(tableView:moveRowAtIndexPath:toIndexPath:)]) {
        [sectionDataSource tableView:tableView moveRowAtIndexPath:sourceIndexPath toIndexPath:destinationIndexPath];
      }
    }

    - (BOOL)tableView:(UITableView *)tableView canEditRowAtIndexPath:(NSIndexPath *)indexPath
    {
      id<OFASectionDataSource> sectionDataSource = self.sectionDataSources[indexPath.section];
      if ([sectionDataSource respondsToSelector:@selector(tableView:canEditRowAtIndexPath:)]) {
        return [sectionDataSource tableView:tableView canEditRowAtIndexPath:indexPath];
      }
      return NO;
    }

    - (UITableViewCellEditingStyle)tableView:(UITableView *)tableView
    editingStyleForRowAtIndexPath:(NSIndexPath *)indexPath
    {
      id<OFASectionDataSource> sectionDataSource = self.sectionDataSources[indexPath.section];
      if ([sectionDataSource respondsToSelector:@selector(tableView:editingStyleForRowAtIndexPath:)]) {
        return [sectionDataSource tableView:tableView editingStyleForRowAtIndexPath:indexPath];
      }
      return UITableViewCellEditingStyleNone;
    }

    - (void)tableView:(UITableView *)tableView didSelectRowAtIndexPath:(NSIndexPath *)indexPath
    {
      [self selectRowAtIndexPath:indexPath];
      id<OFASectionDataSource> ds = self.sectionDataSources[indexPath.section];

      if (ds.didSelect) {
        ds.didSelect([ds sectionObjects][indexPath.row],ds, indexPath, self.tableView);
      }
    }

    - (void)tableView:(UITableView *)tableView didDeselectRowAtIndexPath:(NSIndexPath *)indexPath
    {
      [self selectRowAtIndexPath:indexPath];
      id<OFASectionDataSource> ds = self.sectionDataSources[indexPath.section];
      if (ds.didSelect) {
        ds.didSelect([ds sectionObjects][indexPath.row],ds, indexPath, self.tableView);
      }
    }

    - (BOOL)tableView:(UITableView *)tableView shouldIndentWhileEditingRowAtIndexPath:(NSIndexPath *)indexPath
    {
      return NO;
    }

    - (NSIndexPath *)tableView:(UITableView *)tableView targetIndexPathForMoveFromRowAtIndexPath:(NSIndexPath *)sourceIndexPath toProposedIndexPath:(NSIndexPath *)proposedDestinationIndexPath
    {
      id<OFASectionDataSource> ds = self.sectionDataSources[sourceIndexPath.section];
      if ([ds respondsToSelector:@selector(tableView:targetIndexPathForMoveFromRowAtIndexPath:toProposedIndexPath:)]) {
        return [ds tableView:tableView targetIndexPathForMoveFromRowAtIndexPath:sourceIndexPath toProposedIndexPath:proposedDestinationIndexPath];
      }
      return sourceIndexPath;

    }

    @end
{% endhighlight %}

So it basically proxies any method call to a section data source chosen by the indexPath's section.


A section data source implements the `OFASectionDataSource` protocol or might subclass the `OFASectionDataSource` class.

{% highlight objc %}
@protocol OFASectionDataSource <UITableViewDataSource, UITableViewDelegate>
- (void)setObjectsFromArray:(NSArray *)array;
- (void)addObjectsFromArray:(NSArray *)array;
@property (nonatomic, strong, readonly) NSArray   *sectionObjects;


@optional
@property (nonatomic, copy) NSString * (^sectionHeaderTitle)(NSUInteger section);
@property (nonatomic, copy) NSString * (^sectionFooterTitle)(NSUInteger section);
@property (nonatomic, copy) NSString * (^sectionIndexTitle)(NSUInteger section);
@property (nonatomic, copy) CGFloat (^cellHeight)(NSIndexPath *indexPath, id obj);
@property (nonatomic,copy) void (^didSelect)(id object,id<OFASectionDataSource> dataSource, NSIndexPath *indexPath, UITableView *tableView);

- (id<OFADataFetcher>)                               dataFetcher;
- (NSString *)                                    cellIdentifier;
- (void (^)(id, UITableViewCell *, NSIndexPath *, UITableView*))cellConfigurator;
- (Class)                                         cellClass;

- (void)selectRowAtIndexPath:(NSIndexPath *)indexPath;
- (NSArray *)selectedObjects;
@end


@interface OFASectionDataSource : NSObject <OFASectionDataSource>
- (instancetype)initWithTableView:(UITableView *)tableView
cellClass:(Class)cellClass
cellIdentifier:(NSString *)cellIdentifier
dataFetcher:(id<OFADataFetcher>)dataFetcher
configureCell:(void (^)(id obj, UITableViewCell *cell, NSIndexPath *indexPath, UITableView *tableView))configureCell;

@property (nonatomic, weak, readonly) UITableView *tableView;

@end

{% endhighlight %}


{% highlight objc %}
#import "OFASectionDataSource.h"
#import "OFADataFetcher.h"

@interface OFASectionDataSource ()
@property (nonatomic, weak) UITableView *tableView;
@property (nonatomic, copy) NSString    *cellIdentifier;
@property (nonatomic, strong) Class     cellClass;
@property (nonatomic, copy) void (^configureCell)(id obj, UITableViewCell *cell, NSIndexPath *indexPath, UITableView *tableView);
@property (nonatomic, strong) id<OFADataFetcher>datafetcher;
@end

@implementation OFASectionDataSource
@synthesize sectionHeaderTitle;
@synthesize sectionFooterTitle;
@synthesize sectionIndexTitle;
@synthesize cellHeight;
@synthesize didSelect;
@synthesize sectionObjects = _sectionObjects;

- (instancetype)initWithTableView:(UITableView *)tableView
cellClass:(Class)cellClass
cellIdentifier:(NSString *)cellIdentifier
dataFetcher:(id<OFADataFetcher>)dataFetcher
configureCell:(void (^)(id obj, UITableViewCell *cell, NSIndexPath *indexPath, UITableView *tableView))configureCell;

{
  self = [super init];
  if (self) {
    _tableView      = tableView;
    _cellIdentifier = cellIdentifier;
    _cellClass      = cellClass;
    [tableView registerClass:cellClass forCellReuseIdentifier:cellIdentifier];
    _datafetcher = dataFetcher;

    __weak typeof(self) weakSelf = self;
    [dataFetcher fetchSuccess:^{
      dispatch_async(dispatch_get_main_queue(), ^{
        typeof(self) strongSelf = weakSelf;
        if (strongSelf) {
          [strongSelf setObjectsFromArray:[strongSelf.datafetcher objects]];
          [strongSelf.tableView reloadData];

        }
        });
        }];

        _configureCell  = configureCell;
        _sectionObjects = @[];
      }
      return self;
    }


    - (void (^)(id, UITableViewCell *, NSIndexPath *, UITableView*))cellConfigurator
    {
      return self.configureCell;
    }

    - (void)setObjectsFromArray:(NSArray *)array
    {
      self.sectionObjects = array;
    }

    - (void)addObjectsFromArray:(NSArray *)array
    {
      self.sectionObjects = [self.sectionObjects arrayByAddingObjectsFromArray:array];
    }

    - (NSInteger)numberOfSectionsInTableView:(UITableView *)tableView
    {
      return 0;
    }

    - (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section
    {
      return self.sectionObjects.count;
    }

    -(CGFloat)tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath
    {
      if (self.cellHeight) {
        return self.cellHeight(indexPath, self.sectionObjects[indexPath.row]);
      }
      return 44.0;
    }

    - (NSString *)tableView:(UITableView *)tableView titleForHeaderInSection:(NSInteger)section
    {
      if (self.sectionHeaderTitle)
          return self.sectionHeaderTitle(section);
      return nil;
    }

    - (NSString *)tableView:(UITableView *)tableView titleForFooterInSection:(NSInteger)section
    {
      if (self.sectionFooterTitle)
          return self.sectionFooterTitle(section);
      return nil;
    }

    - (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath
    {
      UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:self.cellIdentifier forIndexPath:indexPath];
      return cell;
    }

    -(void)tableView:(UITableView *)tableView willDisplayCell:(UITableViewCell *)cell forRowAtIndexPath:(NSIndexPath *)indexPath
    {
      [self cellConfigurator](self.sectionObjects[indexPath.row], cell, indexPath, tableView);
    }


    - (void)setSectionObjects:(NSArray *)sectionObjects
    {
      _sectionObjects = sectionObjects;
      [self.tableView reloadData];
    }

    - (UITableViewCellEditingStyle)tableView:(UITableView *)tableView editingStyleForRowAtIndexPath:(NSIndexPath *)indexPath
    {
      return UITableViewCellEditingStyleNone;
    }

    @end

{% endhighlight %}

From an architectual point of view we are done.

But the fun justs starts:

Let's say we want to have a section where I can select 1 to 4 rows.


{% highlight objc %}
#import "OFASectionDataSource.h"

@interface OFAMinMaxSelectionSectionDataSource : OFASectionDataSource

@property (nonatomic, assign, readonly) NSUInteger  min;
@property (nonatomic, assign, readonly) NSUInteger  max;


- (instancetype)initWithTableView:(UITableView *)tableView
                        cellClass:(Class)cellClass
                   cellIdentifier:(NSString *)cellIdentifier
                 minimumSelection:(NSUInteger)min
                 maximumSelection:(NSUInteger)max
                      dataFetcher:(id<OFADataFetcher>)dataFetcher
                    configureCell:(void (^)(id object, UITableViewCell *cell, NSIndexPath *indexPath, UITableView *tableView))configureCell;
@end
{% endhighlight %}

{% highlight objc %}
#import "OFAMinMaxSelectionSectionDataSource.h"

@interface OFAMinMaxSelectionSectionDataSource ()
@property (nonatomic, assign) NSUInteger          min;
@property (nonatomic, assign) NSUInteger          max;
@property (nonatomic, strong) NSMutableOrderedSet *selectedIndexSets;
@end

@implementation OFAMinMaxSelectionSectionDataSource
- (instancetype)initWithTableView:(UITableView *)tableView
cellClass:(Class)cellClass
cellIdentifier:(NSString *)cellIdentifier
minimumSelection:(NSUInteger)min
maximumSelection:(NSUInteger)max
dataFetcher:(id<OFADataFetcher>)dataFetcher
configureCell:(void (^)(id object, UITableViewCell *cell, NSIndexPath *indexPath, UITableView *tableView))configureCell
{
  self = [super initWithTableView:tableView cellClass:cellClass cellIdentifier:cellIdentifier dataFetcher:dataFetcher configureCell:configureCell];
  if (self) {
    _min               = min;
    _max               = max;
    _selectedIndexSets = [[NSMutableOrderedSet alloc] init];
    tableView.allowsSelectionDuringEditing = YES;
  }
  return self;
}

- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath
{
  UITableViewCell *cell = [super tableView:tableView cellForRowAtIndexPath:indexPath];
  cell.editing        = NO;
  cell.selectionStyle = UITableViewCellSeparatorStyleNone;
  return cell;
}

- (void)selectRowAtIndexPath:(NSIndexPath *)indexPath
{
  UITableViewCell *cell = [self.tableView cellForRowAtIndexPath:indexPath];
  if ([self.selectedIndexSets containsObject:indexPath] && [self.selectedIndexSets count] > self.min) {
    [self.selectedIndexSets removeObject:indexPath];
    cell.accessoryType = UITableViewCellAccessoryNone;
    } else {
      [self.selectedIndexSets addObject:indexPath];
      cell.accessoryType = UITableViewCellAccessoryCheckmark;
      if ([self.selectedIndexSets count] > self.max) {
        NSIndexPath *firstIndexPath = [self.selectedIndexSets firstObject];
        cell               = [self.tableView cellForRowAtIndexPath:firstIndexPath];
        cell.accessoryType = UITableViewCellAccessoryNone;
        [self.selectedIndexSets removeObject:firstIndexPath];
        [self.tableView deselectRowAtIndexPath:firstIndexPath animated:NO];
      }
    }
    [self.tableView deselectRowAtIndexPath:indexPath animated:NO];

  }

  - (NSArray *)selectedObjects
  {
    NSMutableIndexSet *indexSets = [NSMutableIndexSet indexSet];
    [self.selectedIndexSets enumerateObjectsUsingBlock:^(NSIndexPath *indexPath, NSUInteger idx, BOOL *stop) {
      [indexSets addIndex:indexPath.row];
      }];

      return [self.sectionObjects objectsAtIndexes:indexSets];
    }

    @end
{% endhighlight %}

And we want the data source to fetch a file from the file system and break up it's lines

{% highlight objc %}
#import <Foundation/Foundation.h>
@import OFADelegateDataSource;

@interface FileDataFetcher : NSObject <OFADataFetcher>
@property (nonatomic, strong, readonly) NSArray *objects;

-(instancetype)initWithFilePath:(NSString *)path;
@end
{% endhighlight %}

{% highlight objc %}
#import "FileDataFetcher.h"


@interface FileDataFetcher ()
@property (nonatomic, strong, readwrite) NSArray *objects;
@property (nonatomic, copy) NSString *path;
@property (nonatomic, copy) void(^success)(void);
@end


@implementation FileDataFetcher

- (instancetype)initWithFilePath:(NSString *)path
{
  self = [super init];
  if (self) {
    _path = path;
  }
  return self;
}

-(void)fetchSuccess:(void (^)(void))success
{
  self.success = success;
  NSError *error;
  NSString *string = [NSString stringWithContentsOfFile:self.path encoding:NSUTF8StringEncoding error:&error];
  if (!string) {
    [self fetchingDataFaildWithError:error onDataFetcher:self];
    } else {
      [self fetchedData:[string componentsSeparatedByString:@"\n"]
      onDataFetcher:self];
      self.success();
    }
  }

  -(void)fetchedData:(id)obj onDataFetcher:(id<OFADataFetcher>)dataFetcher
  {
    self.objects = (NSArray *)obj;
  }

  -(void)fetchingDataFaildWithError:(NSError *)error onDataFetcher:(id<OFADataFetcher>)dataFetcher
  {
    NSLog(@"%@", [error  localizedDescription]);
  }

  @end
{% endhighlight %}


And now we stick it together. Note that the data source also implements the delegate,
but isnt it the responibility of the VC to manage the cells? Yes it is. Just for simplicity
I let my data source act a proxy so that I can send back row's index path and the
corresponding object the VC  by using a block


In VC's `viewDidLoad:`

{% highlight objc %}
FileDataFetcher *lineDataFetcher = [[FileDataFetcher alloc] initWithFilePath:[[NSBundle mainBundle] pathForResource:@"File2" ofType:@"txt"]];


UIFont *font = [UIFont systemFontOfSize:20];
OFASectionDataSource *lineDataSource = [[OFAMinMaxSelectionSectionDataSource alloc] initWithTableView:self.tableView
                                                                                            cellClass:[UITableViewCell class]
                                                                                      cellIdentifier:@"line"
                                                                                    minimumSelection:1
                                                                                    maximumSelection:2
                                                                                         dataFetcher:lineDataFetcher
                                                                                       configureCell:
^(NSString *object, UITableViewCell *cell, NSIndexPath *indexPath, UITableView *tableView)
{
  cell.textLabel.text = object;
  cell.textLabel.font= font;
  cell.textLabel.numberOfLines = 0;
  cell.textLabel.lineBreakMode = NSLineBreakByWordWrapping;
  [cell.textLabel sizeToFit];
  }];


  self.dataSource = ({
    OFASectionedTableViewDataSource *ds= [[OFASectionedTableViewDataSource alloc] initWithTableView:self.tableView];
    [ds addSectionDataSource:lineDataSource];
    ds;
  });
{% endhighlight %}

That's all.

How about exchanging the text with photos form flickr?

{% highlight objc %}
#import "FlickerPhotoFetcher.h"
#import "FlickrKit.h"

#import "flickerCredentials.h"

@interface FlickerPhotoFetcher ()
@property (nonatomic, strong) NSMutableArray *photos;
@property (nonatomic, copy) void (^success)(void);
@end

@implementation FlickerPhotoFetcher

+(void)initialize
{
  [[FlickrKit sharedFlickrKit] initializeWithAPIKey:FLICKR_KEY
  sharedSecret:FLICKR_SECRET];
}

- (instancetype)init
{
  self = [super init];
  if (self) {
    self.photos = [@[] mutableCopy];
  }
  return self;
}

-(NSArray *)objects
{
  @synchronized(self.photos){
    return [self.photos copy];
  }
}


-(void)fetchSuccess:(void (^)(void))success
{
  self.success = success;
  FlickrKit *fk = [FlickrKit sharedFlickrKit];
  FKFlickrInterestingnessGetList *interesting = [[FKFlickrInterestingnessGetList alloc] init];
  [fk call:interesting completion:^(NSDictionary *response, NSError *error) {
    // Note this is not the main thread!
    if (response) {
      NSMutableArray *photoURLs = [NSMutableArray array];
      for (NSDictionary *photoData in [response valueForKeyPath:@"photos.photo"]) {
        NSURL *url = [fk photoURLForSize: [[UIScreen mainScreen] scale] > 1 ? FKPhotoSizeMedium640 : FKPhotoSizeSmall320 fromPhotoDictionary:photoData];
        [photoURLs addObject:url];
      }
      [self fetchPhotos:photoURLs];
    }
  }];

}

-(void)fetchingDataFaildWithError:(NSError *)error onDataFetcher:(id<OFADataFetcher>)dataFetcher
{

}

-(void)fetchedData:(id)obj onDataFetcher:(id<OFADataFetcher>)dataFetcher
{
  dispatch_async(dispatch_get_main_queue(), ^{
    self.success();
  });
}

-(void)fetchPhotos:(NSArray *)photpURLs
{
  NSOperationQueue *queue = [[NSOperationQueue alloc] init];
  __weak typeof(self) weakSelf = self;

  [[photpURLs subarrayWithRange:NSMakeRange(0, 10)] enumerateObjectsUsingBlock:^(NSURL *url, NSUInteger idx, BOOL *stop) {
    [queue addOperationWithBlock: ^ {
      typeof(weakSelf) strongSelf = weakSelf;
      if(strongSelf){
        NSError *error = nil;
        NSData *data = [NSData dataWithContentsOfURL:url options:0 error:&error];
        UIImage *image = nil;
        if (data){
          image = [UIImage imageWithData:data];
        }
        if (image) {
          @synchronized(self.photos){
            [self.photos addObject:image];
            [self fetchedData:self.photos onDataFetcher:self];
          }
        }

      }
    }];
  }];

}

@end
{% endhighlight %}

{% highlight objc %}
OFASectionDataSource *flickrSectionDataSource = [[OFAMinMaxSelectionSectionDataSource alloc] initWithTableView:self.tableView
                                                                                                     cellClass:[UITableViewCell class]
                                                                                                cellIdentifier:@"thirdSectionCell"
                                                                                              minimumSelection:1
                                                                                              maximumSelection:2
                                                                                                   dataFetcher:flickrPhotoFetcher
                                                                                                 configureCell:^(UIImage *image, UITableViewCell *cell, NSIndexPath *indexPath, UITableView *tableView){
  cell.backgroundView  = ({
    UIImageView *iv = [[UIImageView alloc] initWithFrame:cell.bounds];
    iv.image =image;
    iv.contentMode = UIViewContentModeScaleAspectFill;
    iv.center = cell.center;
    iv;
  });
  cell.clipsToBounds = YES;
}];
[flickrSectionDataSource setSectionHeaderTitle:^NSString *(NSUInteger section) {
  return @"Flickr";
}];

flickrSectionDataSource.cellHeight = ^(NSIndexPath *ip, UIImage *obj){
  CGFloat factor = obj.size.width / self.view.frame.size.width;
  return obj.size.height / factor;
};

{% endhighlight %}


[apple]: http://developer.apple.com
[intentions]: http://chris.eidhof.nl/posts/intentions.html
