---
layout: "post"
title:  "ReactiveCocoa初识"
date:   2016-08-01 21:55:40
categories: [杂项]
tags: [小试牛刀]
---
几个月前看了一点RAC的介绍，感觉很强大但也很难入门，这种编程思想值得去学些下，整理下RAC的资源。

官方介绍
>ReactiveCocoa 受`函数响应式编程`激发。不同于使用可变的变量替换和就地修改，RAC提供Signals（被表示为**RACSignal**）来捕获当前值和将来值。
通过链接（chaining），组合（combining）和对Signals做出反应（reacting），我们不必频繁地观察并更新值，而是声明式编写软件。
比如，文本域可以绑定到最新的时间，当它变化时，不需用额外的代码来观察时间每秒钟更新文本域。它类似KVO，但是用blocks替代了重写 **-observeValueForKeyPath:ofObject:change:context:**。
Signals也可以表示异步操作。这极大地简化了异步编码，网络方面的代码。
RAC一个重要的优点就是它提供了单独的、统一的方法来处理异步行为，包括委托方法，回调blocks，target-action机制，通知和KVO。

这还有一个简单的栗子：

```objc
// When self.username changes, logs the new name to the console. //
// RACObserve(self, username) creates a new RACSignal that sends the current
// value of self.username, then the new value whenever it changes.
// -subscribeNext: will execute the block whenever the signal sends a value.

[RACObserve(self, username) subscribeNext:^(NSString *newName) { NSLog(@"%@", newName); }];
```

但是不同于KVO通知，signals可以链接在一起操作：

```objc
// Only logs names that starts with "j". // // -filter returns a new RACSignal that only sends a new value when its block
// returns YES.
[[RACObserve(self, username) filter:^(NSString *newName) {
	 return [newName hasPrefix:@"j"]; }]
subscribeNext:^(NSString *newName) {
	NSLog(@"%@", newName);
	}];
```

Signals也可以被用于导出状态。不必观察属性然后设置其他属性来响应这个属性新的值，RAC可以依照signals和操作来表达属性：

```objc
RAC(self, createEnabled) = [RACSignalcombineLatest:@[ RACObserve(self, password), RACObserve(self, passwordConfirmation) ] reduce:^(NSString *password, NSString *passwordConfirm) { 	
	        return@([passwordConfirmisEqualToString:password]);
	}];
```

Signals可以建立在任意值随时间的流动上，不仅仅是KVO。比如，它们也能表示按钮被按下：

```objc
self.button.rac_command = [[RACCommand alloc] initWithSignalBlock:^(id _) {
	NSLog(@"button was pressed!"); return [RACSignal empty];
}];
```

或者异步网络操作：

```objc
self.loginCommand = [[RACCommand alloc] initWithSignalBlock:^(id sender) {
// The hypothetical -logIn method returns a signal that sends a value when
// the network request finishes.
	 return [client logIn];
}];
// -executionSignals returns a signal that includes the signals returned from
// the above block, one for each time the command is executed.
[self.loginCommand.executionSignals subscribeNext:^(RACSignal *loginSignal) {
// Log a message whenever we log in successfully. 	[loginSignal subscribeCompleted:^{
 		NSLog(@"Logged in successfully!");
 	}];
 }];

 // Executes the login command when the button is pressed. self.loginButton.rac_command = self.loginCommand;
```

Signals也能表示定时器，其他UI事件，或者任何其他随时间而改变的东西。
通过链接和转换这些Signals，可以为异步操作建立更加复杂的行为。在一组操作完成后，后续工作能容易地被触发：

```objc
[[RACSignal merge:@[ [client fetchUserRepos], [client fetchOrgRepos] ]] subscribeCompleted:^{
	NSLog(@"They're both done!");
}];
```
Signals可以被链接起来按顺序地执行异步操作，而不用嵌套回调blocks。这类似futures and promises是如何经常使用的：

```
[[[[client logInUser] flattenMap:^(User *user) {
// Return a signal that loads cached messages for the 	user. return [client loadCachedMessagesForUser:user];
}] flattenMap:^(NSArray *messages) {
// Return a signal that fetches any remaining messages.
	return [client fetchMessagesAfterMessage:messages.lastObject];
}] subscribeNext:^(NSArray *newMessages) {
	NSLog(@"New messages: %@", newMessages);
	} completed:^{
	 NSLog(@"Fetched all messages.");
}];
```

RAC甚至使绑定到异步操作结果更加容易：

```objc
RAC(self.imageView, image) = [[[[client fetchUserWithUsername:@"joshaber"] deliverOn:[RACScheduler scheduler]] map:^(User *user) {
// Download the avatar (this is done on a background queue).
	return [[NSImage alloc] initWithContentsOfURL:user.avatarURL];
	}]
	// Now the assignment will be done on the main thread.
	deliverOn:RACScheduler.mainThreadScheduler];
```

上面示范了RAC能做什么，但它没示范RAC为何这么强大。用README的篇幅的例子很难赞美RAC，但是它让编程有更加简化的状态，更少的饮用，更好的代码位置和更好的表达意图。

###RAC中的类和方法
#####RACSignal和RACStream
RAC的核心是Signal，对应的类为RACSignal，它其实是一个事件源，Signal会给它的订阅者（subscribers）发送一连串的事件。有三种事件：next，error和completed。Signal可以在error或completed事件发出前发出任意多的next事件。
RACSignal有很多方法用于订阅事件，查看RACSignal (Subscription)类别可以看到所有的订阅事件的方法，每个方法都会将类型为(void (^)(id x))的block作为参数，当事件发生时block中的代码会执行，例如**subscribeNext:**方法会传入一个block作为参数，当Signal的next事件发出后，block会接收到事件并执行。
RAC为UIKit添加了很多类别来让我们可以订阅UI组件的事件，比如UITextField (RACSignalSupport)中的rac_textSignal会在文本域内容变化时发出next事件。
事件包含的内容可以是类型，只要是对象就行，如果是一些数字，布尔值等字面量，可以用@()语法装箱成NSNumber。
RACSignal是RACStream的子类，RACStream是一个抽象类，描述了值的流动，列举一下它比较常用的操作（Operations类别）：

**filter:** 对RACStream中的事件内容进行过滤，返回一个过滤事件内容后的instancetype

**map:** 会将事件中的数据转换成你想要的数据，返回一个转换事件内容后的instancetype

**flattenMap:** 在map的基础上使其flatten，也就是当Signal嵌套（一个Signal的事件是另一个Signal）的时候，会将内部Signal的事件传递给外部Signal

**distinctUntilChanged** 比较数值流中当前值和上一个值，如果不同，就返回当前值，简单理解为“流”的值有变化时反馈变化的值，求异存同。
PS：instancetype是程序运行时对象的类型，有可能为RACStream，也可以为其子类RACSignal。正是因为这些操作事件的方法都会返回事件源对象相同的类型，事件可以被一连串的被这些方法修改，过滤等，这就形成了管道，管道中传递着事件，包含着value。
RACSignal还有一些方法是对Signal做操作的，在RACSignal (Operations)类别中有详细的描述，比较常用的如下：
**combineLatest:reduce:** 将一组Signal发出的最新的事件合并成一个Signal，每当这组Signal发出新事件时，reduce的block会执行，将所有新事件的值合并成一个值，并当做合并后Signal的事件发出去。这个方法会返回合并后的Signal。

PS：关于reduce的block中参数，其实是与combineLatest中数组元素一一对应的.

**doNext:** 这个向Signal管道上添加添加副作用。它并不会改变事件，参数block也没有返回值，它返回一个执行了block的Signal，这样block中的副作用就被注入到了以前的Signal。

**then:** 当一个订阅者被发送了completed事件后，then:方法才会执行，订阅者会订阅then:方法返回的Signal，这个Signal是在block中返回的。这样优雅的实现了从一个Signal到另一个Signal的订阅。

**deliverOn:** 参数为RACScheduler类的对象scheduler，这个方法会返回一个Signal，它的所有事件都会传递给scheduler参数所表示的线程，而以前管道上的副作用还会在以前的线程上。这个方法主要是切换线程。

**subscribeOn:** 功能跟**deliverOn:**相同，但是它也会将副作用也切换到制定线程中。

**throttle:** 它接收一个时间间隔interval作为参数，如果Signal发出的next事件之后interval时间内不再发出next事件，那么它返回的Signal会将这个next事件发出。也就是说，这个方法会将发送比较频繁的next事件舍弃，只保留一段“静默”时间之前的那个next事件，这个方法常用于处理输入框等信号（用户打字很快），因为它只保留用户最后输入的文字并返回一个新的Signal，将最后的文字作为next事件参数发出。

**and、or、not** NSNumber中Bool的与、或、非操作，将Signal发出的事件内容转化。
还可以根据方法（SEL类型）来创建Signal，每当该方法被调用时，Signal都会将此方法被传入的参数打包成RACTuple元组类型来发送next事件给它的接受者。**rac_signalForSelector:**和**rac_signalForSelector:fromProtocol:**这两个方法都能通过指定的方法来创建Signal。
######RACSubscriber
RACSubscriber是一个协议，包含了向订阅者发送事件的方法。

```objc
[RACSignal createSignal:^RACDisposable *(id<RACSubscriber> subscriber) {
	[subscriber sendNext:@(YES)];
	[subscriber sendCompleted]; return nil;
}];
```

上面工厂方法用于创建一个Signal，当Signal被订阅时，**createSignal:**的参数block中的内容被执行。block的参数是一个实现**RACSubscriber**协议的对象，然后向这个订阅者发送了next事件（内容为NSNumber类型的@YES值）和completed事件。

PS：除此之外RACSubscriber还有**sendError:**和**didSubscribeWithDisposable:**两个方法。

#####RACDisposable
你会发现RACSignal (Subscription)类别中所有方法的返回值类型都是RACDisposable，它的dispose方法可以让我们手动移除订阅者。举个栗子：

```objc
RACSignal *backgroundColorSignal = [self.searchText.rac_textSignal map:^id(NSString *text) {
	 return [self isValidSearchText:text] ? [UIColor whiteColor] : [UIColor yellowColor];
}];
RACDisposable *subscription = [backgroundColorSignal subscribeNext:^(UIColor *color) {
	 self.searchText.backgroundColor = color;
}];
// at some point in the future ...
[subscription dispose];
```

当管道（好吧比较短）的订阅者全部被移除后，管道中的代码不会执行，包括三种事件参数block中的代码和诸如doNext:等副作用的block。可以简单理解为，当管道中的Signal没人订阅，它的事件就不会发出了。

#####RACCommand
**RACCommand** 通常用来表示某个Action的执行，比如点击Button。
#####RACScheduler
类似于GCD中的序列，是管理线程的类，负责RAC中让信号发出的事件华丽丽的在线程中穿梭，尤其是想更新UI必须在主线程中的时候，可以让事件直接从其他线程跳到主线程。此外RACScheduler也有优先级、延时等GCD中的特性。
#####常用宏定义
**RAC()** 可以将Signal发出事件的值赋值给某个对象的某个属性，其参数为对象名和属性名

**RACObserve()** 参数为对象名和属性名，新建一个Signal并对对象的属性的值进行观察，当值变化时Signal会发出事件.

参考资料：

[nshipster-Reactive​Cocoa][1]

[ReactiveCocoa与Functional Reactive Programming][2]

[1]: http://nshipster.cn/reactivecocoa/
[2]: http://limboy.me/ios/2013/06/19/frp-reactivecocoa.html
