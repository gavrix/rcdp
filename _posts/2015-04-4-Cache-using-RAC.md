---
layout: post
title: "Cache using RAC"
category: post
author: octogavrix
tag: Patterns
---

One the patterns built using RAC which shows dramatic increase in clarity, simplicity and flexibility I came across while developing with RAC is caching. Let's take a look at how to build cache using RAC and why I find it so super cool


## The cache

Let's first define what is cache. [Wiki] defines cache as 

>  a cache (/ˈkæʃ/ kash) is a component that transparently stores data so that future requests for that data can be served faster.

So this component is effectively a layer between the party interested in data and actual data source. It assumes an interface for requesting data, the request is proxied to the actual datasource if the data requested hasn't been yet fetched or computed and stored somewhere for easier access.

More complicated cache could have 'drop cache' strategy — support for dropping data that's been cached based on a certain criteria like maximum cache size, cache TTL (or expiry date), free memory availability, as well as an explicit 'drop cache' functionality exposed through a method or function.


## Imperative way of building cache

An obvious way to build cache with Cocoa is of course using `NSDictionary` for storing cached data. There is a more suitable API for this `NSCache`, which supports values evicting out-of-the-box. They values are evicted is not obvious though, and there a lot of complainеs related (see good notes on [NSHipster].

Naive implementation could look like this:

``` objc
- (void)getData:(id)inputParam forKey:(NSString *)key withCompletionBlock:(void (^)(id data, NSError *error)) {
	//look up cached
	if (_cache[key]) {
		completionBlock(_cache[key], nil);
	}
	else {
		//get data
		[self _actualGetDataRequest:inputParam withCompletionBlock:^(id result, NSError *error) {
			//store cache
			if (!error) {
				_cache[key] = result;
			}
			completion(result, error);
		}];
	}

}
```

Seems legit, but this code has obvious drawbacks:
- it's not re-entrant safe, meaning that, when asked for data with the same key not waiting for first one to complete will result in two requests, and second request will overwrite data in cache written by first one.
- can't be canceled
- doesn't support cache drop based on criteria.

Of course all of this issues could be fixed. The fix requires making the code more complex in non-obvious manner. For example, to make it re-entrant safe you need to track all blocks for the requests with the same key and make signal request and in completionBlock call other outer completionBlocks. To fix cancellation you need to re-factor the code to use NSOperations instead of block-based methods. In the end, this cache is likely going to be a monster.

## Cache using RAC

Now let's take a look at how RAC can simplify this.

First, we wrap actual data request into a signal

```objc
RACSignal *dataSignal = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
	id operation = _actualGetDataRequest(inputParam, ^(id data, NSError *error) {
		if (error) {
			[subscriber sendError:error];
		}
		else {
			[subscriber sendNext:asyncActionResult];
			[subscriber sendCompleted];
		}
	});
	return [RACDisposable disposableWithBlock:^{
		[operation cancel];
	}];
}];
```

Now, the trick is to use `replayLast` or `replayLazily` signal operation, which basically creates buffer and hold in it cached value! Then we put that signal into local cache storage:

```objc
_cache[key] = dataSignal.replayLast;
```

Now, have two options on how to return result. We can return this signal itself, so that caller of this API could connect to it and extract data when it's ready. Or we can do this ourselves right here. Here's what final code could look like:

```objc
- (void)getData:(id)inputParam forKey:(NSString *)key withCompletionBlock:(void (^)(id data, NSError *error)) {
	if (!_cache[key]) {
		RACSignal *dataSignal = [RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
			id operation = _actualGetDataRequest(inputParam, ^(id data, NSError *error) {
				if (error) {
					[subscriber sendError:error];
				}
				else {
					[subscriber sendNext:asyncActionResult];
					[subscriber sendCompleted];
				}
			});
			return [RACDisposable disposableWithBlock:^{
				[operation cancel];
			}];
		}];
		_cache[key] = dataSignal.replayLast;
	}
	if (completionBlock) {
		RACSignal *cachedSignal = _cache[key];
		[cachedSignal subscribeNext:^(id data){
			completionBlock:(data, nil);
		} error: ^(NSError *error) {
			completioBlock:(nil, error);
			}]
	}	

}
```

[Wiki]:http://en.wikipedia.org/wiki/Cache_(computing)
[NSHipster]:http://nshipster.com/nscache/