---
layout: post
title: "-flattenMap: vs </br>-map: + -switchToLatest:"
category: post
author: octogavrix
tag: Framework
---

As readers with some [FRP] knowledges (specifically, knowledge of [Monad] concept) may already know that `-flattenMap:` is a central piece driving the whole signals chaining mechanism. However, based on my own experience with [ReactiveCocoa], I found that it's not that useful and eventually combination of `-map:` and `-switchToLatest` fits better most of the time. Let's take a closer look at what these operations are and what is the difference.

## `-flattenMap:`

I mentioned that `-flattenMap:` is a central gluing component between monads, but how does it apply to [ReactiveCocoa] and it's signals? Well, signals, or speaking more generally, `RACStream` [is a Monad] and `-flattenMap:` is that piece that turns it into Monad.

As understood from it's name, `-flattenMap:` maps each object from the signal onto another signal, hence creating new signal which is signal of signals. Then this signal of signals is flattened, i.e. turned into signal of objects contained in nested signals. A confusion may come when you take a look at `-map:` method implementation. Apparently, [it's implemented _through_](https://github.com/ReactiveCocoa/ReactiveCocoa/blob/bcb3d9a680adcea12927a74dea9216b0e40d96b5/ReactiveCocoaFramework/ReactiveCocoa/RACStream.m#L89) `-flattenMap:`. But when you recall that `RACStream` [is a Monad] it actually makes sense: all the connections between two Monads (e.g. between two streams) must be expressed through `-flattenMap:`. To be precise, that's not true for all operations from `RACSignal` (for example, [`-deliverOn:`](https://github.com/ReactiveCocoa/ReactiveCocoa/blob/bcb3d9a680adcea12927a74dea9216b0e40d96b5/ReactiveCocoaFramework/ReactiveCocoa/RACSignal%2BOperations.m#L995)), but that's framework implementation details.

Now, consider following example

```objc
RACSignal *sourceSignal = [RACSignal return: @0];

RACSignal *(^makeSignal)(NSNumber *start) = ^RACSignal *(NSNumber *start) {
	__block int step = start.intValue;
	return [[[RACSignal interval:1.0 onScheduler:RACScheduler.mainThreadScheduler] map:^id(id value) {
		return @(step++);
	}] take:3];
};

RACSignal *resultSignal = [sourceSignal flattenMap:^RACStream *(NSNumber *value) {
	return makeSignal(value);
}];

[resultSignal subscribeNext:^(NSNumber *x) {
	NSLog(@"%@", x);
}];

```

In this example for each item from `sourceSignal` we create a new `resultSignal` which outputs 3 sequentially incremented integers starting with one came from `sourceSignal` with 1 second delay. The output for this code will likely be 

```
0
1
2
```

Now, consider sourceSignal is itself defined as a sequence with delay:

```
RACSignal *sourceSignal = makeSignal(@0);
```

I ran that code and here the result:

```
0
1
1
2
2
2
3
3
4
```

As you understand, we created a signal by merging 3 other signals together. 

```
-----0-----1-----2
      \     \     \
1:      -----\(0)--\-(1)----(2)
              \     \
               \     \
2:               -----\-(1)----(2)----(3) 
                       \
                        \
3:                       ---------(2)-----(3)----(4)

      --------(0)----(1)(1)-(2)(2)(2)--(3)(3)----(4)
```

On this diagram the sequence at the bottom is a result of merging 1, 2 and 3 together.

That's how `-flattenMap:` works and there's nothing surprising here. However, we often create signals representing certain state of a model or UI, and in response to that state we perform transformation by mapping events in that signal to something different. For example, when text in a text field changes we perform search request and place result in a label. Search request in an asynchronous operation and we'll wrap it into a signal. If we `-flattenMap:` that signal in response to new search term, and do this for all search terms coming through source signals, we will likely get into situation where multiple search results will be merging into resulting signal. And the worse thing is that the order of that results will not be preserved. In this case we would likely want to use `-switchToLatest`.

## `-switchToLatest`

Combination `-map:` + `-switchToLatest` works very close to `-flattenMap:` with one key difference, signals mapped to incoming events **are not merged**. Instead, they are put in series, meaning once new signal is mapped to incoming event, resulting signal unsubscribes from current one and subscribes to new one. Getting back to example with search term and search request, by using `-map:` and `-switchToLatest` we could map to each new search term a new search request dropping the one created before. The resulting signal will be passing only those search results for the latest search term.
Let's now take a look at the code example we used before and replace `-flattenMap:` with `-map:` + `switchToLatest` combination:

```objc

RACSignal *(^makeSignal)(NSNumber *start) = ^RACSignal *(NSNumber *start) {
	__block int step = start.intValue;
	return [[[RACSignal interval:1.0 onScheduler:RACScheduler.mainThreadScheduler] map:^id(id value) {
		return @(step++);
	}] take:3];
};

RACSignal *sourceSignal = makeSignal(@0);

RACSignal *resultSignal = [[sourceSignal map:^RACStream *(NSNumber *value) {
	return makeSignal(value);
}] switchToLatest];

[resultSignal subscribeNext:^(NSNumber *x) {
	NSLog(@"%@", x);
}];

```

This code will output following:

```
2
3
4
```

Here's the diagram explaining what actually happens:

```
-----0-----1-----2
      \     \     \
1:      -----\     \
              \     \
               \     \
2:               -----\
                       \
                        \
3:                       ------(2)-----(3)----(4)

      -------------------------(2)-----(3)----(4)
```

Each new event from source signal drops the perviously mapped signal and eventually we see the output from only the last one.


While using [ReactiveCocoa] in practice I noticed that most of the time I need to use `-switchToLatest` rather than `-flattenMap:`. You can think of `-flattenMap:` as making _all possible combinations_ which pretty rare use case when programming UI. Anyways, you now know the benefits and drawbacks of both and would be able to choose the correct one for particular use case.




[is a Monad]:http://stackoverflow.com/questions/22284944/what-does-it-mean-that-racstream-represents-a-monad
[FRP]:http://en.wikipedia.org/wiki/Functional_reactive_programming
[Monad]:http://en.wikipedia.org/wiki/Monad_(functional_programming)
[ReactiveCocoa]:https://github.com/ReactiveCocoa/ReactiveCocoa