---
layout: post
title: ViewModel
categories: [post]
author: octogavrix
---


This is the first topic in this blog, and since ViewModel is really a center component in [MVVM] pattern, which in turn is recommended to be used along with FRP, I decided to start this blog series with it. Why is MVVM so recommended for FRP? Well, these are my thoughts.

### What is ViewModel? 

In short, it's a piece of data representation logic. Or piece of functionality associated with the data in any way. You rarely represent the data backing your app to the user _exactly_ the way this data is stored, most likely you somehow transform it first. Say, you store a date in one of machine formats, but show to the user human-friendly formatted string. That's pretty much what ViewModel is for: it takes model (or any data assumed as model) and _transforms_ it, _denormalizes_ it, to exact form to fill in the UI. ViewModel actually holds not only transformations but also _actions_ associated with the model. We'll see how actions on model can be stored in ViewModel later.

### Why ViewModel?

So, why ViewModel is so convenient to use with FRP and ReactiveCocoa? Well, because data _transformations_ I mentioned are easily described as signal pipelines in declarative manner. Here's a very simple example:

```objc
@property (nonatomic) NSString *dateString;
@property (nonatomic) Model *model;
...
RAC(self, dateString) = [RACObserve(self, model.date) map:^id (NSDate *date) {
	NSDateFormatter *dateFormatter = [[NSDateFormatter alloc] init];
	[dateFormatter setDateFormat:@"yyyy-MM-dd"];

	return [dateFormatter stringFromDate:date];
}];
```

In this example, whenever ``model` property of our ViewModel object, or model's `date` property, changes - `dateString` is _automatically_ populated with corresponding value. We created an invisible connection between them. That's why it's called reactive: two properties are now in relationships so that changes to one of them reactively affects the other one.

### Actions in ViewModel

I mentioned that we may even incorporate _actions_ in ViewModel using ReactiveCocoa's features: by creating signal that perform those actions upon subscription and sends results to a subscriber. I'll cover this case in more details a special post. There is also a convenient component `RACCommand` which basically manages creation of signals and subscribing to them, handles execution in serial and concurrent mode as well as `enabled` state.

### Decoupling and compounding ViewModels.

It's a very important feature of any piece of software - to be easily decoupled. Most of the time particular ViewModel serving a whole screen of the app does more than one atomic thing. Now imaging that another ViewModel serving another view has something in common with this one. In this scenario ViewModels can be refactored so that common functionality is moved into a third special ViewModel. This let us have more granular and less spaghetti dependent code. How to reuse common sub-ViewMode? Simply make it as property of another ViewModel and connect inputs and outputs accordingly:

```objc
@property (nonatomic) SubViewModel *subViewModel;
@property (nonatomic) Model *inputModel;
@property (nonatomic) NSString *outputValue;

- (instancetype)init {
	self = [super init];
	if (self) {
		//tune subViewModel
		RAC(self, subViewModel.model) = RACObserve(self, inputModel);

		//tune output
		RAC(self, outputValue) = RACObserve(self, subViewModel.itsOutputValue);
	}
	return self;
}

```


### Testability

No matter whether you practice TDD, or simply cover your code with integrational tests, having code covered is a must these days. Apparently, ViewModels are very convenient pattern to cover with tests, since it has strongly formalized inputs and outputs. Tests have straightforward pattern: set arbitrary inputs of your choice and check outputs for expected values.


ViewModels, by having use of FRP, particular RectiveCocoa, make it clearer and more straightforward how to implement functionality needed. It doesn't mean you have to write less code — no, computers won't do your job. But it definitely makes it clearer. The clearer you understand what you need to do, the less mistakes you make. And that is pretty much the main advantage FRP and why MVVM is so damn good: it allows you to make less mistakes.

[MVVM]:http://en.wikipedia.org/wiki/Model_View_ViewModel