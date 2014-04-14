---
layout: post
title: Auto-updating relative timestamp
category: post
author: octogavrix
tag: Patterns
---

This first very practical example in this blog on how to user [ReactiveCocoa] in real world. The idea is simple: we have a label somewhere in UI showing the date timestamp. First and simplest coming to mind is simply format the date. We saw this example in first post related to [ViewModel]. Like this:

```objc
RAC(self, dateString) = [RACObserve(self, model.date) map:^id (NSDate *date) {
	NSDateFormatter *dateFormatter = [[NSDateFormatter alloc] init];
	[dateFormatter setDateFormat:@"yyyy-MM-dd"];

	return [dateFormatter stringFromDate:date];
}];
```

## Auto-updating label

Now imagine that we want to represent timestamp in relative manner like '5d ago' or '5m ago'. Or '5s ago'. Obviously, there's nothing wrong with days or minutes, but showing seconds is tricky. Once next second ticks, you value '5s ago' becomes invalid, as it's actually '6s ago' now and so on. You need to put this update on timer.

With [ReactiveCocoa] and FRP you recall, that we deal with signals and we can construct signal that sends new value every time current becomes invalid. Here's how I do this:

```objc
RAC(self, timestamp) = [[RACObserve(self, model.timestamp) map:^id(NSDate *value) {
			return [[RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
				NSUInteger days, hours, minutes, seconds;
				absoluteTimeUnitsSinceDate(value, &days, &hours, &minutes, &seconds);
				NSUInteger updateAfter = NSNotFound;
				NSString *currentValue = nil;
				if (days) {
					currentValue = [NSString stringWithFormat:@"%ud", (unsigned int)days];
				}
				else if (hours) {
					currentValue = [NSString stringWithFormat:@"%uh", (unsigned int)hours];
				}
				else if (minutes) {
					currentValue = [NSString stringWithFormat:@"%um", (unsigned int)minutes];
					updateAfter = 60;
				}
				else {
					if (seconds < 5.0) {
						currentValue = @"just now";
					}
					else {
						currentValue = [NSString stringWithFormat:@"%us", (unsigned int)seconds];
					}
					updateAfter = 1;
				}

				[subscriber sendNext:currentValue];
				RACSerialDisposable *disposable = [RACSerialDisposable serialDisposableWithDisposable:nil];
				if (updateAfter != NSNotFound) {
					disposable.disposable = [[RACScheduler currentScheduler] afterDelay:updateAfter schedule:^{
						[subscriber sendCompleted];
					}];
				}
				return disposable;
			}] repeat];
			
		}] switchToLatest];

```
This may me hard to get from the first sight. Let me explain what's going on there. The function `absoluteTimeUnitsSinceDate` returns absolute number of seconds, minutes and hours and days from passed from the given day and current time. The main idea of this stucture is that we create signal that sends current value and when it's time to update it completes by scheduling that code in current scheduler. Once signal completes, we reconnect to it immediately — that's was `repeat` does. Upon reconnection, or resubscribing, signal will calculate and send current value to that new subscription, and depending on the `updateAfter` will schedule another delayed complete-and-update.

At the end of the chain there is `-switchToLatest` call. We will be using this often. Basically, it's one of the ways to flatten signal. In fact, combination `-map:` + `-switchToLatest` and `flattenMap:` are very similar and can be confused. We'll learn what the difference between them is later, in the meantime I'll explain what `switchToLatest` does: it applies to the signal of signals (e.g. to the signal, all the values passed through which are other signals) and once getting new signal, it subscribes it's subscriber to that signal, up until new signals arrives. In other words, it switches it's own subscriber between inner signals. In our case, for each incoming timestamp we generate signal and `switchToLatest` connects to that signal. When new timestamps arrives, `switchToLatest` will abandon existing internal signal, generate new one for new timestamp and subscribe to it.

I assume, relation we coded is happened in `ViewModel`, and thus, `ViewModel` has property `timestamp` and you can then bind UI's label `text` property ViewModel's `timestamp` property:

```objc
RAC(self, label.text) = RACObserve(self, viewModel.timestamp);
```

That's it. It will look like this:

<img style="display:block;margin:auto;" src={{ site.url }}/assets/auto-updating-timestamp.gif />

[ReactiveCocoa]:https://github.com/ReactiveCocoa/ReactiveCocoa
[ViewModel]:{% post_url 2014-03-28-ViewModel %}