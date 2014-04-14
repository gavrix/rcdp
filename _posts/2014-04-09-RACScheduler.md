---
layout: post
title: RACScheduler
category: post
author: octogavrix
tag: Framework
---

Before digging into real world [ReactiveCocoa] examples and finally starting fulfilling blog with what it name says: design patterns, it's worth talking about ReactiveCocoa scheduling.

## RACScheduler

All events passed along the [ReactiveCocoa] signals are pushed by special framework component — scheduler, represented as `RACScheduler` class cluster. The reason why it was designed that way is that scheduler concept brings very convenient abstraction over synchronous/asynchronous/delayed events passing as well as scheduled actions cancellation. "Events passing" is performed as block of course and that's what `RACScheduler` operates with: it schedules blocks. Canceling scheduling blocks could be done by disposing `RACDisposable` objects retuned by scheduling methods.

As I said, `RACScheduler` is a class cluster. These are possible concrete schedulers:

### RACImmediateScheduler

Private scheduler used in [ReactiveCocoa] internally and and supports only synchronous scheduling. It simply execute the block right away. Delayed scheduling works by blocking current thread with `-[NSThread sleepUntilDate:]`. Obviously, there's no way to control cancellation of such scheduling, so disposables retuned by this scheduler will do nothing (in fact, scheduling methods of this scheduler return nil).

### RACQueueScheduler

This scheduler uses GCD queues for scheduling blocks. It's functionality is pretty straightforward if you know what GCD does. It's actually a very thin layer over dispatching blocks in GCD queues.

### RACSubscriptionScheduler

Another private scheduler used by framework internally. It forwards scheduling to current thread's scheduler (scheduler can be associated with the thread) if it exists or to default background queue scheduler.


## Interface

Scheduler methods look like this:

```objc 
- (RACDisposable *)schedule:(void (^)(void))block;

- (RACDisposable *)after:(NSDate *)date schedule:(void (^)(void))block;

- (RACDisposable *)afterDelay:(NSTimeInterval)delay schedule:(void (^)(void))block;

- (RACDisposable *)after:(NSDate *)date repeatingEvery:(NSTimeInterval)interval withLeeway:(NSTimeInterval)leeway schedule:(void (^)(void))block;

```

So, scheduling block would look like this:

```objc
RACDisposable *disposable = [[RACScheduler mainThreadScheduler] afterDelay:5.0 schedule:^{
	//do something
}];

//if you changed your mind
[disposable dispose]; // block scheduling cancelled, it won't be executed.

```
[ReactiveCocoa]:https://github.com/ReactiveCocoa/ReactiveCocoa