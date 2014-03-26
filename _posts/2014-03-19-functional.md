---
layout: post

title: First impressions of ReactiveCocoa
subtitle: "And how functionals help you produce readable code without going all-in with FRP"

excerpt: "I had been looking forward to learn more about this framework ever since I first heard about it. The talk was inspiring, and I bought the book the next day, but now we are two weeks later and I'm still not sure what to really think of it..."

author:
  name: Hannes Verlinde
  twitter: hverlind
  bio: iOS Developer at In the Pocket
  image: avatar.png
---

Earlier this month I was in Amsterdam to talk at [mdevcon](http://mdevcon.com). Right before I went up on stage, I watched [Ash Furrow](http://ashfurrow.com)'s introduction to [ReactiveCocoa](https://github.com/ReactiveCocoa/ReactiveCocoa). I had been looking forward to learn more about this framework ever since I first heard about it. The talk was inspiring, and I bought [the book](https://leanpub.com/iosfrp) the next day, but now we are two weeks later and I'm still not sure what to really think of it.

To be clear, I have always been a fan of functional programming. I spent most of my last year at university writing an XML editor in Haskell, using [Functional Reactive Animation](http://conal.net/fran/) for the UI. And although I have never used a purely functional programming language in my professional life as a software developer, I have often experienced how applying functional concepts in imperative programming languages produces code that is more easy to read, understand and maintain.

Back in my C++ days, I was taught to liberally use `boost::bind` and friends. Later on, as a .NET developer, I loved to refactor C# code using LINQ operators. And of course, in Objective-C we now have blocks, so I tend to use `enumerateObjectsUsingBlock`, `sortedArrayUsingComparator` and `filteredArrayUsingPredicate` whenever I can.

Now I think we can all agree that blocks are not the end-all solution to writing declarative code in Objective-C. In fact, the thing I missed most when switching from C# was the ability to chain higher-order functions to filter, map and fold collections with lazy evaluation, like you can with the `Where`, `Select` and `Aggregate` operators from LINQ.

Fortunately, Objective-C libraries exist that offer exactly this functionality. [RXCollections](https://github.com/robrix/RXCollections) is one such library, and it is used in Ash Furrow's book to dip the readers' feet into the FRP water. A similar library is [Underscore.m](http://underscorem.org) (inspired by [underscore.js](http://underscorejs.org)), which can be used like this to calculate the longest common number prefix in a set of strings:

{% highlight objc %}

    NSArray *inputStrings = @[ @"67034X", @"67039XXX", @"XX", @"6704", @"67004", @"670X" ];

    NSCharacterSet *nonDigits = [[NSCharacterSet decimalDigitCharacterSet] invertedSet];

    NSString *commonPrefix = Underscore.array(inputStrings)
        .map(^(NSString *value) {
           return [value stringByTrimmingCharactersInSet:nonDigits];
        })
        .filter(^BOOL(NSString *value) {
           return [value length] > 0;
        })
        .reduce(nil, ^(NSString *accumulator, NSString *value) {
           return accumulator ? [accumulator commonPrefixWithString:value options:0] : value;
        }); // yields @"670”

{% endhighlight %}

This code is powerful and elegant at the same time, and it is a typical example of how using functionals in an imperative context can lead to clearer code. In fact you can accomplish the exact same thing in ReactiveCocoa with an `RACSequence`:

{% highlight objc %}

    NSString *commonPrefix = [[[inputStrings.rac_sequence
        map:^(NSString *value) {
           return [value stringByTrimmingCharactersInSet:nonDigits];
        }]
        filter:^BOOL(NSString *value) {
           return [value length] > 0;
        }]
        foldLeftWithStart:nil reduce:^(NSString *accumulator, NSString *value) {
           return accumulator ? [accumulator commonPrefixWithString:value options:0] : value;
        }];

{% endhighlight %}

But of course ReactiveCocoa is a lot more ambitious than this. Introducing Functional Reactive Programming in your code requires a real paradigm shift and forces you to rethink what programs, inputs and outputs really are. I do like the premise of FRP, and some of the ReactiveCocoa examples (e.g. chaining dependent network requests) look very convincing to me, but I am still wondering to what extent this can be applied in real life projects.

For starters, it seems that quite a bit of boilerplate code is needed to lift the behavior of existing classes to the FRP signal space. For common UI components like buttons and switches, this functionality is provided through built in categories, e.g. there is an `RACSignalSupport` category on `UITextField` that defines an `rac_textSignal`. If you want to use other classes with ReactiveCocoa, you will need to do the lifting yourself.

Since I've been working a lot with iBeacons lately, I thought it might be a good idea to try and rewrite some of that code using ReactiveCocoa. After all, we are constantly talking about beacon *signals*, so I expected this to be a good match for FRP. In an imperative context, you request an instance of the `CLLocationManager` class to start ranging beacons in a certain region, and you observe the results by implementing the corresponding delegate method. Using ReactiveCocoa, this is the code I came up with to turn the observed beacon data into an `RACSignal`:

{% highlight objc %}

    RACRangeSignal *rangeSignal = 
        [[[self rac_signalForSelector:@selector(locationManager:didRangeBeacons:inRegion:)
                         fromProtocol:@protocol(CLLocationManagerDelegate)]
        reduceEach:^(CLLocationManager *manager, NSArray *beacons, CLRegion *region) {
            return [beacons.rac_sequence signal];
        }] concat];

{% endhighlight %}

Now I might be missing some subtle ReactiveCocoa shortcuts that would allow me to write this in a more elegant way, and maybe this isn't such a good use case for FRP after all, but an example like this one won't exactly help to convince my co-workers (or even myself) to start using ReactiveCocoa in large scale projects.

Don't get me wrong. I think ReactiveCocoa is pretty cool. I certainly like it better than [KVO and bindings](http://blog.metaobject.com/2014/03/the-siren-call-of-kvo-and-cocoa-bindings.html). But it's also a pretty big step. Maybe I just need a bit more time and practice. Or maybe I should start small by applying it to very restricted subsets of projects. And in the meantime, I'll keep trying to convince my colleagues to adopt block based syntax and higher order functionals.
