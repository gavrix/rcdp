---
layout: post
title: "Loadable resource ViewModel"
category: post
author: octogavrix
tag: Patterns
---

When [discussing]({% post_url 2014-03-28-ViewModel %}) benefits of ViewModel pattern I mentioned that ViewModel can be easily compound from more granular ViewModels. In other words, we can break complex ViewModel into combination, or structure of simpler ViewModels. Let's build a very simple ViewModel responsible for managing single loadable resource, e.g. resource which has to be computed somehow (often something downloaded or fetched from the internet, that's why I used term 'loadable').

## Architecture

First, let's define high-level design of such a ViewModel. It will take some sort of model as an input, create computational signal (signal, subscribing to each somputation occurs and result passed into). This ViewModel will have property holding result and computational signal needs to be connected to it. Also, this ViewModel has to handle errors occuring in computational signal and provide `loading` property, as a bouns.

## Implementation

Let's make an initial draft.

``` objc

@interface SRGLoadbleViewModel : NSObject
@property (nonatomic) id model;
@property (nonatomic) id result;
@property (nonatomic) NSError *lastError;
@property (nonatomic) BOOL loading;
@end

```

Notice, we provided a `model` property with undefined type `id`. Our ViewModel is supposed to abstract away handling results of computation but it should contain specifics of computation itself. If think a little more, we don't even need `model` property here, because computation may not depend on this input. We said that computation is wrapped into a signal, so subclasses have to provide this signal. And a particular subclass will define how computation depends on a model (if there is such a dependency). Now question is, how subclasses have to provide computation signal? Naive approach would be by using overloadable methods, but I will go the other way here. 

We provide a property holding computation signal:

``` objc

@interface SRGLoadableViewModel : NSObject

@property (nonatomic) BOOL loading;
@property (nonatomic) NSError *lastError;

@property (nonatomic) id result;
@property (nonatomic) RACSignal *computationSignal;

@end


```

Now, base class, `SRGLoadableViewModel`, can create a signal of computation signals by observing `computationSignal` property:

```objc

RACSignal *loadableSignal = [[RACObserve(self, computationSignal)
                              map: ^id (RACSignal *signal) {
    return signal ?: RACSignal.empty;
}] map: ^id (RACSignal *signal) {
    return signal.replayLazily;
}].replayLast;


```

What's that `replayLazily` and `replayLast` mean? Remember I said we wrap computation into signal? Here's example how subclass of our ViewModel may implement that:

```objc

RAC(self, computationSignal) = [RACObserve(self, subclassOwnModel) map:^id(id model) {
	return model ? [RACSignal createSignal: ^RACDisposable *(id < RACSubscriber > subscriber) {
		//do computation
		[subscriber sendNext: <* computation result*>];
		[subscriber sendCompleted];
		return nil;
	}] : nil;
}];

```

So, `computationSignal` contains signal which performs computation _every time_ this is subscribed to. That means, base ViewModel needs to ensure that `computationSignal` is only subscribed to once and result of computation is shared between any interested components. That's why `loadableSignal` proxies the actual computation signal through `replayLazily`. `replayLast` at the end ensure that only one copy of that proxy is created. Note, we could have flatten signal at this point and apply `replayLast` once, but holding the actual computation signal inside provides some useful benefits: we could be able to wrap internal signal, which we'll example of when handling error and producing `loading` flag.


Let's take a look at how to flatten `laodableSignal` (passing along another signal performing actual computation).

```objc

RAC(self, result) = [loadableSignal map:^id(RACSignal *signal) {
	return [signal catchTo:[RACSignal return:nil]];
}].switchToLatest;

```

Note, we catch any error occuring while computation and return `nil` as result for that computation. Otherwise, signal bound to `result` property would complete and next computationSignal will not be handled.

## Error handling

As you noticed, I declared a property called `lastError`. This property will hold the last occurred error in computationSignal. Extracting error from stream of computationSignals is pretty straightforward:

```objc

RAC(self, lastError) = [loadableSignal map:^id(RACSignal *signal) {
	return [[signal.ignoreValues.materialize filter:^BOOL(RACEvent *event) {
		return event.eventType == RACEventTypeError;
	}] map:^id(RACEvent *event) {
		return event.error;
	}];
}].switchToLatest;

```

Here we transform each computationSignal into signal returning nothing but error should error occur in that computationSignal. I used here method `materialize` to turn `error` signal event into regular value. Such signal will complete after sending error value, but since it is wrapped into super signal flattened by `switchToLatest` resulting signal will never complete, and will be passing errors only.

## Loading flag

As a nice bonus, I added property called `loading`. This is supposed to be BOOL flag indicating where computation is in process (computation may be asynchronous and time consuming operation). Idea behind it's implementation is this: for each computation signal coming from `loadableSignal` we create new one passing `YES` immediately, then waiting for computationSignal to complete and then passing 'NO'.

```objc
RAC(self, loading) = [loadableSignal map:^id(RACSignal *signal) {
	return [RACSignal concat:@[
							   [RACSignal return:@YES],
							   [signal.ignoreValues catchTo:RACSignal.empty],
							   [RACSignal return:@NO]
							   ]];
}].switchToLatest;

```

Note, we catch possible error occurring while computation here as well as when binding `result` property with the exception that we catch error to empty signal — we can simple ignore that error. We handle errors in different place, here we want to ignore any errors as resulting signal will error (and complete) as well, which is unwanted, since next computationSignal may complete successfully.


I placed complete code for this `SRGLoadbleViewModel` on a [gist](https://gist.github.com/gavrix/1e09f68871519818401d)


