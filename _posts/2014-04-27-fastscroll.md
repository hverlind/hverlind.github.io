---
layout: post

title: Asynchronous image loading in fast scrolling table cells 
subtitle: Working around UIImageView+AFNetworking limitations

excerpt: "In the UIImageView+AFNetworking implementation, certain decisions are made to optimize for loading images currently on screen, which is done by cancelling previous requests for offscreen views. This behavior can interfere with cell reuse behavior, which may result in images not being loaded correctly..."

author:
  name: Hannes Verlinde
  twitter: hverlind
  bio: iOS Developer at In the Pocket
  image: avatar.png
---

AFNetworking's UIImageView category must be one of its most popular features. The functionality it offers is so commonly needed, and the implementation details are hidden in such a way that you would almost forget that a third party network library is being used. For instance, consider the following simple example where a batch of random images is loaded in a plain table view. 

{% highlight objc %}

    @implementation TableViewController

    - (NSArray *)imageURLs
    {
        if (!_imageURLs)
        {
            NSMutableArray *imageURLs = [NSMutableArray array];
            
            for (NSInteger index = 0; index < 100; ++index)
            {
                [imageURLs addObject:[NSString stringWithFormat:
                                      @"http://dummyimage.com/88/%06X/%06X&text=%d",
                                      arc4random() % 0xFFFFFF,
                                      arc4random() % 0xFFFFFF,
                                      index + 1]];
            }
            
            _imageURLs = imageURLs;
        }
        
        return _imageURLs;
    }

    - (void)viewDidLoad
    {
        [super viewDidLoad];
        
        [self.tableView registerClass:[UITableViewCell class]
               forCellReuseIdentifier:@"TableViewCell"];
    }

    - (NSInteger)numberOfSectionsInTableView:(UITableView *)tableView
    {
        return 1;
    }

    - (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section
    {
        return [self.imageURLs count];
    }

    - (UITableViewCell *)tableView:(UITableView *)tableView
             cellForRowAtIndexPath:(NSIndexPath *)indexPath
    {
        UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:@"TableViewCell"
                                                                forIndexPath:indexPath];
        
        cell.imageView.image = [UIImage imageNamed:@"placeholder"];
      
        [cell.imageView setImageWithURL:[NSURL URLWithString:self.imageURLs[indexPath.row]]];
        
        return cell;
    }

    @end

{% endhighlight %}

One downside of using this convenient category is that it can lead to undesired artifacts when scrolling fast in a table view. In the UIImageView+AFNetworking implementation, certain decisions are made to optimize for loading images currently on screen, which is done by cancelling previous requests for offscreen views. This behavior can interfere with cell reuse behavior, which may result in images not being loaded correctly, as shown below:

![Images are not loaded correctly](/images/2014042701.gif)

To figure out what can be done about these unwanted side-effects, we switch to a lower level implementation using an `AFHTTPRequestOperationManager` and its `GET:parameters:success:failure` method.

{% highlight objc %}

    - (AFHTTPRequestOperationManager *)operationManager
    {
        if (!_operationManager)
        {
            _operationManager = [[AFHTTPRequestOperationManager alloc] init];
            _operationManager.responseSerializer = [AFImageResponseSerializer serializer];
        };
        
        return _operationManager;
    }

    - (UITableViewCell *)tableView:(UITableView *)tableView
             cellForRowAtIndexPath:(NSIndexPath *)indexPath
    {
        UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:@"TableViewCell"
                                                                forIndexPath:indexPath];
        
        cell.imageView.image = [UIImage imageNamed:@"placeholder"];
      
        [self.operationManager GET:self.imageURLs[indexPath.row]
                        parameters:nil
                           success:^(AFHTTPRequestOperation *operation, id responseObject) {
            cell.imageView.image = responseObject;
        } failure:^(AFHTTPRequestOperation *operation, NSError *error) {
            NSLog(@"Failed with error %@.", error);
        }];
        
        return cell;
    }

{% endhighlight %}

When running this code, it becomes obvious why cancelling requests for offscreen views is not such a bad idea: if the request queue cannot keep up with the speed of scrolling, the images in the reused cells get overwritten multiple times (and if the responses would arrive out of order, we could end up with wrong images in the cells).

![Images get overwritten multiple times](/images/2014042702.gif)

There are two simple approaches to work around this. The first one is to cancel the pending request whenever a cell is reused and a new request is queued:

{% highlight objc %}

    - (UITableViewCell *)tableView:(UITableView *)tableView
             cellForRowAtIndexPath:(NSIndexPath *)indexPath
    {
        UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:@"TableViewCell"
                                                                forIndexPath:indexPath];
        
        cell.imageView.image = [UIImage imageNamed:@"placeholder"];
      
        [cell.imageView.associatedObject cancel];
        
        cell.imageView.associatedObject =
            [self.operationManager GET:self.imageURLs[indexPath.row]
                            parameters:nil
                               success:^(AFHTTPRequestOperation *operation, id responseObject) {
            cell.imageView.image = responseObject;
        } failure:^(AFHTTPRequestOperation *operation, NSError *error) {
            NSLog(@"Failed with error %@.", error);
        }];
        
        return cell;
    }

{% endhighlight %}

The `associatedObject` property is used in the code above to remember the previously queued operation for each reused cell. It is implemented as a category on `NSObject` to avoid subclassing in this simplified example:

{% highlight objc %}

    #import <objc/runtime.h>

    @interface NSObject (Associating)

    @property (nonatomic, retain) id associatedObject;

    @end

    @implementation NSObject (Associating)

    - (id)associatedObject
    {
        return objc_getAssociatedObject(self, @selector(associatedObject));
    }

    - (void)setAssociatedObject:(id)associatedObject
    {
        objc_setAssociatedObject(self,
                                 @selector(associatedObject),
                                 associatedObject,
                                 OBJC_ASSOCIATION_RETAIN_NONATOMIC);
    }

    @end

{% endhighlight %}

Now the result looks a lot better, but the downside is that the images for the cells you quickly scroll past will only get loaded when you scroll back up, because their initial fetch request has been cancelled: 

![Image requests are cancelled](/images/2014042703.gif)

An alternative approach is to let each queued operation finish, but associate URLs with reused cells, and only assign the fetched image if it corresponds to the last loaded URL for that cell:

{% highlight objc %}

    - (UITableViewCell *)tableView:(UITableView *)tableView
             cellForRowAtIndexPath:(NSIndexPath *)indexPath
    {
        UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:@"TableViewCell"
                                                                forIndexPath:indexPath];
        
        cell.imageView.image = [UIImage imageNamed:@"placeholder"];
      
        NSString *url = self.imageURLs[indexPath.row];
        
        cell.imageView.associatedObject = url;
        
        [self.operationManager GET:url
                        parameters:nil
                           success:^(AFHTTPRequestOperation *operation, id responseObject) {
            if ([cell.imageView.associatedObject isEqualToString:url])
            {
                cell.imageView.image = responseObject;
            }
        } failure:^(AFHTTPRequestOperation *operation, NSError *error) {
            NSLog(@"Failed with error %@.", error);
        }];
        
        return cell;
    }

{% endhighlight %}

Now the images for the cells you scroll past will be cached too, with the downside that loading the images for the bottom cells will be noticeably slower:

![Image requests are queued](/images/2014042704.gif)

Of course you can implemented more advanced strategies, e.g. pause rather than cancel requests and resume them when higher priority requests have finished. The main thing to take away is that, when you hit a wall with a convenient high-level API, you can always drop down to a lower level API and implement your own logic on top of it.  