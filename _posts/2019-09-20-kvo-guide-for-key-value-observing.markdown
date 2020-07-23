---
layout: post
title: "Guide to KVO in Swift 5 with code examples"
date: 2019-09-20 17:30:00 +0600
description: "Everything you need to know about Key-Value Observing in Swift and Objective-C"
tags: [ios,swift,kvo,observevalueforkeypath,nskeyvalueobservation]
redirect_from:
  - kvo-guide-for-key-value-coding
comments: true
sharing: true
published: true
img: kvo_001.jpg
---

## TL;DR

For the **KVO code example** in Swift jump stright to the [KVO in Swift](https://nalexn.github.io/kvo-guide-for-key-value-observing/#kvo_swift) section.

## The concept

**KVO**, which stands for Key-Value Observing, is one of the techniques for observing the program state changes available in Objective-C and Swift.

The concept is simple: when we have an object with some instance variables, KVO allows other objects to establish surveillance on changes for any of those instance variables.

KVO is a practical example of the [Observer pattern](https://en.wikipedia.org/wiki/Observer_pattern). What makes Objective-C (and Obj-C bridged Swift) unique is that every instance variable that you add to the class becomes observable through KVO right away! (There are exceptions to this rule, I'll talk about them in the article).

But in the majority of other programming languages, such a tool doesn't come out of the box - you usually need to write additional code in the variable's setter to notify the observers about the value changes.

> Swift has inherited KVO from Objective-C, so for a full picture you need to understand how KVO works in Objective-C.

## KVO in Objective-C

Consider we have a class named `Person` with properties `name` and `age`

```objc
@interface Person: NSObject
 
@property (nonatomic, strong) NSString *name;
@property (nonatomic, assign) NSInteger age;
 
@end
```

The objects of this class now are able to communicate the changes of the properties through KVO, but with no additional code - this feature comes for free!

So the only thing we need to do is to start the observation in another class:

```objc
@implementation SomeOtherClass

- (void)observeChanges:(Person *)person {
    [person addObserver:self
             forKeyPath:@"age"
                options:NSKeyValueObservingOptionNew
                context:nil];
}

- (void)observeValueForKeyPath:(NSString *)keyPath
                      ofObject:(id)object
                        change:(NSDictionary<NSKeyValueChangeKey,id> *)change
                       context:(void *)context {
    if ([keyPath isEqualToString:@"age"]) {
        NSNumber *ageNumber = change[NSKeyValueChangeNewKey];
        NSInteger age = [ageNumber integerValue];
        NSLog(@"New age is: %@", age);
    }
}

@end
```

**That's it!** Now every time the `age` property changes on the `Person` we'll have `New age is: ...` printed to the log from the observer's side.

As you can see, there are two methods involved in KVO communication.

The first is `addObserver:forKeyPath:options:context:`, which can be called on any `NSObject`, including `Person`. This method attaches the observer to an object.

The second is `observeValueForKeyPath:ofObject:change:context:` which is another standard method in `NSObject` that we **have to override** in our observer's class. This method is used for handling the observation notifications.

There is a third method, `removeObserver:forKeyPath:context:`, which allows you to stop the observation. It's important to unsubscribe from notifications if the observed object outlives the observer. So the subscription just has to be removed in the observer's `dealloc` method.

Now, let's talk about the parameters of the methods used in KVO.

**The method we used for attaching the observer is declared in `NSObject`**

```objc
- (void)addObserver:(NSObject *)observer 
         forKeyPath:(NSString *)keyPath
            options:(NSKeyValueObservingOptions)options
            context:(nullable void *)context;
```

- **observer** is the object that will be receiving the change notifications. Usually, you provide `self` in this parameter, as the `addObserver:` is called from inside own instance method.
- **keyPath** is a string parameter that in simplest case is just the *name of the property* you want to observe. If the property references a complex object hierarchy it can be a set of property names for digging into that hierarchy: `"person.father.age"`
- **options** is an `enum` that allows for customizing what information is delivered with notification and when it should be sent. Available options are `NSKeyValueObservingOptionNew` and `NSKeyValueObservingOptionOld`, which control whether to include the most recent and the previous values respectively. There is also `NSKeyValueObservingOptionInitial` for triggering the notification right after the subscription, and `NSKeyValueObservingOptionPrior` for diffing the changes in a collection, such as instertions of deletions in `NSArray`.
- **context** is a reference to object of an arbitrary class, which can be helpful for identifying the subscription in certain complex use cases, such as when working with CoreData. In most other cases you simply provide `nil` here.

**The method we used for handling the update notifications**

```objc
- (void)observeValueForKeyPath:(NSString *)keyPath
                      ofObject:(id)object
                        change:(NSDictionary<NSKeyValueChangeKey,id> *)change
                       context:(void *)context
```
- **keyPath** is the same string value we provided when attaching the observer. You may ask why it is provided here as well. The reason is that we may be observing multiple properties at once, so this parameter can be used to distinguish the notifications for one property from another.
- **object** is the observed object. Since we can observe changes on more than one object, this parameter allows us to identify who's property has changed.
- **change** is the dictionary with information about the changed value. Based on the `NSKeyValueObservingOptions` we provided upon subscription, this dictionary may contain the current value under key `NSKeyValueChangeNewKey`, previous value for `NSKeyValueChangeOldKey`, and the "diff" information when observing changes in a collection: `NSKeyValueChangeIndexesKey` and `NSKeyValueChangeKindKey`
- **context** is the reference provided upon subscription. Again, used for proper observation identification and in most cases can be ignored.

### When KVO does not work

Even though KVO looks like magic, there is nothing extraordinary behind it. In fact, you can have direct access to its internals, which are hidden by default.

The trick is how Objective-C generates setter for properties. When you declare a property like

```objc
@property (nonatomic, assign) NSInteger age;
```

The factual setter generated by Objective-C is equivalent to the following:

```objc
- (void)setAge:(NSInteger)age {
    [self willChangeValueForKey:@"age"];
    _age = age;
    [self didChangeValueForKey:@"age"];
}
```

And if you explicitly define the setter without calling these `willChangeValueForKey` and `didChangeValueForKey`

```objc
- (void)setAge:(NSInteger)age {
    _age = age;
}
```

... the KVO will stop working for this property.

So basically, these two methods `willChangeValueForKey` and `didChangeValueForKey` allow KVO to deliver the updates to the subscribers, and the developer can opt-out by omitting those calls from the setter.

It is important to understand that every `@property` synthesized by Objective-C adds a hidden instance variable with `_` prefix.

For example, `@property NSInteger age;` generates an instance variable with the name `_age` that can be accessed just like the property:

```objc
self.age = 25;
self._age = 25;
```

The difference is that `self.age = 25;` triggers setter `setAge:`, while `self._age = 25;` changes the stored variable directly.

This means that even if the KVO is enabled for the `age` property, the KVO communication will work correctly for `self.age = 25;` and won't deliver an update for `self._age = 25;`

Another way to break free from KVO is to not use `@property` in the first place, but instead store the instance variable in the anonymous category of the class:

```objc
@interface Person () {
    NSInteger _privateVariable;
}
@end
```

For such variables, Objective-C does not generate setter and getter, thus not enabling KVO.

## <a name="kvo_swift">[KVO in Swift](#kvo_swift)</a>

Swift has inherited the support for the KVO from Objective-C, but unlike the latter, KVO is disabled in Swift classes by default.

Objective-C classes used in Swift keep KVO enabled, but for a Swift class we need to set the base class to `NSObject` plus add `@objc dynamic` attributes to the variables:

```swift
class Person: NSObject {
    @objc dynamic var age: Int
    @objc dynamic var name: String
}
```

There are two APIs available in Swift for Key-Value Observing: the old one, which came from Objective-C, and the new one, which is more flexible, safe and Swift-friendly.

Let's start with the **new one**:

```swift
class PersonObserver {

    var kvoToken: NSKeyValueObservation?
    
    func observe(person: Person) {
        kvoToken = person.observe(\.age, options: .new) { (person, change) in
            guard let age = change.new else { return }
            print("New age is: \(age)")
        }
    }
    
    deinit {
        kvoToken?.invalidate()
    }
}
```

As you can see, the new API is using a closure callback for delivering the change notification right in the place where the subscription started.

This is more convenient and safe because we no longer need to check the `keyPath`, `object` or `context`, - no other notifications are delivered in that closure, just the one we've subscribed on.

There is a new way for managing the observation lifetime - the act of subscribing returns a token of type `NSKeyValueObservation` which has to be stored somewhere, for example, in an instance variable of the observer class.

Later on, we can call `invalidate()` on that token to stop the observation, like in the `deinit` method above.

The final change is related to the keyPath. String was error-prone because when you rename a variable the compiler won't be able to tell you that the keyPath now leads to nowhere. Instead, this new API is using Swift's special type for keyPath, which allows the compiler to verify the path is valid.

The `options` parameter has just the same set of options as in Objective-C. If you need to provide more than one option, you just bundle them in an array: `options: [.new, .old]`

The old API is also available, although it maintained all its disadvantages, so I encourage you to use the new API instead.

Here is the **old one**:

```swift
class PersonObserver: NSObject {
    
    func observe(person: Person) {
        person.addObserver(self, forKeyPath: "age",
                           options: .new, context: nil)
    }
    
    override func observeValue(forKeyPath keyPath: String?,
                               of object: Any?,
                               change: [NSKeyValueChangeKey : Any]?,
                               context: UnsafeMutableRawPointer?) {
        if keyPath == "age",
           let age = change?[.newKey] {
             print("New age is: \(age)")
        }
    }
}
```

The old API **requires** the observer to be an `NSObject` descendant as well. We also need to verify the `keyPath`, `object`, and `context`, since other notifications are also delivered in this method, just like in Objective-C.

## KVO alternatives

There are quite a few other techniques in modern iOS development serving the same purpose: propagation of the state changes. And I have to say that KVO is the one that is used **least often** because alternatives often outpace it in convenience and versatility. In fact, I wrote a series of articles where I covered **all** the available tools for state change propagation and thoroughly described **the pros and cons** of each. Here are quick references:

* [delegate](https://nalexn.github.io/callbacks-part-1-delegation-notificationcenter-kvo/#delegate)
* [NotificationCenter](https://nalexn.github.io/callbacks-part-1-delegation-notificationcenter-kvo/#NotificationCenter)
* [KVO](https://nalexn.github.io/callbacks-part-1-delegation-notificationcenter-kvo/#KeyValueObserving)
* [Closure](https://nalexn.github.io/callbacks-part-2-closure-target-action-responder-chain/#Closure_Block)
* [Target-Action](https://nalexn.github.io/callbacks-part-2-closure-target-action-responder-chain/#Target-Action)
* [Responder Chain](https://nalexn.github.io/callbacks-part-2-closure-target-action-responder-chain/#Responder_chain)
* [Promise](https://nalexn.github.io/callbacks-part-3-promise-event-stream/#Promise)
* [Event](https://nalexn.github.io/callbacks-part-3-promise-event-stream/#Event)
* [Stream of values](https://nalexn.github.io/callbacks-part-3-promise-event-stream/#Stream)

The final post in that series of articles is the [**ultimate guide**](https://nalexn.github.io/state-management-guide-ios/) where I describe the practical situations where **one tool suits better than the others**.

Finally, with the release of Apple's **Combine** framework, KVO now stands no chances to maintain its original popularity, but it is still important to know how it works!