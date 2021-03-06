# Operacjas

[![Swift][swift-badge]][swift-url]
[![Build Status][travis-badge]][travis-url]
[![Platform][platform-badge]][platform-url]
[![Latest0.4][version-0.6-badge]][releases-url]

**Operacjas** is an open-source implementation of concepts from [Advanced NSOperations][anso-url] talk.

> The `NSOperation` class is an abstract class you use to encapsulate the code and data associated with a single task.

`Operacja` is an `NSOperation` subclass which adds some very powerful concepts to it and extends the definition of readiness.

- `0.0.x` versions contains code directly from Apple's [sample project](https://developer.apple.com/sample-code/wwdc/2015/downloads/Advanced-NSOperations.zip).
- `0.2.x` and later versions contains community improvements.

We recommend you to use the newest "community version" (`0.5.0` at the time).

#### Note

There are aslo other implementations of "Advanced NSOperations". You can also see [danthorpe/Operations][danthorpe/Operations], which soon will become **ProcedureKit** ([#343](https://github.com/danthorpe/Operations/issues/343)). It has a lot more possibilities, and many predefined operations, conditions, observers and so on. However, we have different goals. **AdvancedOperations/Operations** aims to have a pretty small codebase which will perform as a ground for your own projects, features and new cool ideas, wherever **ProcedureKit** is mostly all-in-one solution. Also, **Operacjas** is still in the "flow state" and things are moving fast, and **ProcedureKit** already has stable releases. So, basically, use the project which is closer to you (you can also consider using so-called "Apple versions" of **Operacjas**).

Also take a look at [PSOperations](https://github.com/pluralsight/PSOperations).

## Usage

*DISCLAIMER*: **Operacjas** are un-swifty as hell, with all these subclassing and reference semantics everywhere. But the goal of **Operacjas** is not to make `NSOperation` "swifty", but to make it more powerful *using* Swift. Operations are still a very great concept that can dramatically simplify the structure of your app, they are system-aware and they *just work*.

Before reading the **Usage** section, please go watch [Advanced NSOperations](https://developer.apple.com/videos/play/wwdc2015/226/) talk from Apple, it will help you to understand what's going on, especially if you're new to `NSOperation` and `NSOperationQueue`.

### Operations
Operations are abstract distinctive pieces of work. Each operation must accomplish some task, and the only thing it cares about is that task. **Operacjas** introduces `Operacja` - an `NSOperation` subclass which add some new concepts and redefines readyness state, and `OperationQueue` - `NSOperationQueue` subclass which supports this concepts.

##### Creation of `Operacja`
The best way to create an operation is to subclass `Operacja`. Unlike `NSOperation`, here you only need to override new `execute()` method.

```swift
// WARNING! The operation below is absolutely useless
class LogOperation<T> : Operacja {
    
    private let value: T
    init(value: T) {
        self.value = value
    }
    
    override func execute() {
        guard !cancelled else {
            finish()
            return
        }
        print(value)
        finish()
    }
    
}
```

*TIP*: Better check `cancelled` property in a `guard` statement (not `if cancelled { ...`), because when the operation is `cancelled`, you need to *both* `finish()` and `return`, and sometimes that's easy to forget. `guard` guarantees that you're gonna return from the function.

Then you just add your operation to the queue:

```swift
let queue = OperacjaQueue()
let logOperation = LogOperation(value: "Test")
queue.addOperation(logOperation)
```

You can also use `BlockOperacja` and create operation simply from the closure, like this:

```swift
let operation = BlockOperacja {
    self.performSegueWithIdentifier("showEarthquake", sender: nil)
}
operationQueue.addOperation(operation)
```

There are also other customisation points along with `execute()`. For example, you can override `finished(_:)` method that is called when your operation is going to finish. It's a good place to do any clean-up. Think of it as operation-level `defer`.

##### Dependencies
Operations can depend on another operations. Adding dependencies is simple:

```swift
let first = OperationA()
let second = OperationB()
second.addDependency(first)
```

That means that `second` operation will not start before `first` operation enters it's `finished` state. Dependencies are also queue-independent, i.e. operations from different queues can depend on each other.

If your operation is depending on `Operacja` (not just `NSOperation`), you can also make this:

```swift
let first = OperationA()
let second = OperationB()
second.addDependency(first, expectSuccess: true)
```

That just means that `second` will be cancelled if `first` will finish with errors. In some cases that might be useful.

*WARNING*: If the operation depends on itself, it's never gonna be executed, so don't do that. It may sound like an obvious thing, but seriously - always keep that in mind. If your operation queue is stalled, you've probably deadlocked yourself somewhere. Also, if some operation A depends on B, and B depends on A - your app is deadlocked again.

##### Vital operations

Vital operations are operations which execution is super important for its queue. That means that vital operation **blocks** an execution of new operations until all vital operations are finished. You can make operation vital easily:

```swift
let important = ImportantOperation()
queue.addOperation(important, vital: true)
```

Or, if you want to treat operation from the different queue as vital, you can actually do that too!

```swift
let superImportant = SuperImportantOperation()
firstQueue.addOperation(superImportant)

let other = SomeOperation()
// Queue dependency! Whooo!
secondQueue.addDependency(superImportant)
secondQueue.addOperation(other)
// `other` won't be executed until `superImportant` is finished.

```

### Operation observing

You can observe operation lifecycle by assigning one or more *observers* to it. *Observer* is an implementor of `OperationObserver` protocol:

```swift
// Example observer
final class LogObserver: OperacjaObserver {
    
    func operationDidStart(operation: Operacja) {
        debugPrint("\(operation) did start")
    }
    
    func operation(operation: Operacja, didProduceOperation newOperation: NSOperation) {
        debugPrint("\(operation) did produce \(newOperation)")
    }
    
    func operationDidFinish(operation: Operacja, errors: [ErrorType]) {
        if errors.isEmpty {
            debugPrint("\(operation) did finish succesfully")
        } else {
            debugPrint("\(operation) did failed with \(errors)")
        }
    }
    
}
```

```swift
let logger = LogObserver()
let operation = OperationA()
operation.addObserver(logger)
queue.addOperation(operation)
```

It's a good practice to implement `OperacjaObserver` directly if you want your observer to be reusable. However, if you only want to observe individual `Operacja`, consider using `.observe(_:)` method on an `Operacja` object, which makes observing very easy:

```swift
let myOperation = MyOperation()
myOperation.observe {
    $0.didStart {
        // operation did start
    }
    $0.didProduceAnotherOperation { produced in
        // operation did produce another operation
    }
    $0.didSuccess {
        // operation did finish successfuly
    }
    $0.didFail { errors in
        // operation did fail
    }
}
```
That creates a new observer and automatically assigns it to `myOperation`.

Instead of using `didSuccess` and `didFail`, you can also use `didFinishWithErrors`, which is gonna be notified when operation finishes, no matter successfuly or not. Also keep in mind that if you specify `didFinishWithErrors`, `didSuccess` and `didFail` will be ignored. In most cases, using `didSuccess` and `didFail` is the best option.

### Operation conditions
You can solve pretty complex problems with `NSOperation`s and dependencies, but `OperacjaCondition` takes that even further, allowing you to create very sophisticated workflows. You can create and assign any number of conditions to an `Operacja` object. Conditions ensure you that some operation will be executed *only* if condition was satisfied. Take these situations as examples:

 - ~~Download file only if server is reachable~~ (see below)
 - Perform request only if user is logged in
 - Try to get user's location only if permission to do so is granted

Basically, your condition can do two things. First, it can *generate dependency* for operation. Second, it can *evalute condition*, meaning it can check if the condition is satisfied or not.

For example, some `LoggedInCondition` can generate `LoginOperation` which will present a login view if the user is not logged in. After `LoginOperation` is completed (i.e. user logs in or cancel), you can evaluate a condition to actually check if the user is logged in, and pass that result to determine whether your initial operation needs to be executed. Let's look at some code:

```swift
struct LoggedInCondition: OperacjaCondition {
    
    // let's assume that we have some `LoginController` that controls the current login status
    let loginController: LoginController
    
    init(loginController: LoginController) {
        self.loginController = loginController
    }
    
    func dependencyForOperation(operation: Operacja) -> NSOperation? {
    	// The actual implementation of `LoginOperation` is not the point here.
        return LoginOperation(loginController: self.loginController)
    }
    
    // It's a good practice to always explain what is wrong with condition result
    enum Error: ErrorType {
        case NotLoggedIn
    }
    
    func evaluateForOperation(operation: Operacja, completion: OperacjaConditionResult -> Void) {
    	// Here we checks whether the user is logged in and call the correspondent completion
        if loginController.loggedIn {
            completion(.Satisfied)
        } else {
            completion(.Failed(error: Error.NotLoggedIn))
        }
    }
}
```

```swift
let requestOperation = GetUserInfoOperation()
let loggedIn = LoggedInCondition(loginController: loginController)
requestOperation.addCondition(loggedIn)
```

One important note here: `evaluateForOperation(_:completion:)` gets called *after* the generated operation is executed. Actually, operation returned from here is going to be added as a dependency for initial operation and assigned to an operation queue *before* initial operation, and initial operation is evaluating conditions only after all it's dependencies are executed.

For example, let's take next situation. We have operation **A** to which we assign two conditions - **Bc** and **Cc**. These conditions generate one operation each - **Bo** and **Co**. So the workflow is going to look like this:

> **Bo** execution -> **Co** execution -> **A** evaluates it's conditions (**Bc** and **Cc**) -> **A** execution

Of course, there are situations when you don't need to generate dependencies. In this cases you can just return `nil` in `dependencyForOperation(_:)`. For example, this is how Apple's `PassbookCondition` looks:

```swift
import PassKit

/// A condition for verifying that Passbook exists and is accessible.
struct PassbookCondition: OperacjaCondition {
    
    static let name = "Passbook"
    static let isMutuallyExclusive = false
    
    init() { }
    
    func dependencyForOperation(operation: Operacja) -> NSOperation? {
        /*
            There's nothing you can do to make Passbook available if it's not
            on your device.
        */
        return nil
    }
    
    enum Error: ErrorType {
    	case PassLibraryIsNotAvailable
    }
    
    func evaluateForOperation(operation: Operacja, completion: OperacjaConditionResult -> Void) {
        if PKPassLibrary.isPassLibraryAvailable() {
            completion(.Satisfied)
        }
        else {
            completion(.Failed(error: Error.PassLibraryIsNotAvailable))
        }
    }
}
```

Think of it this way: you generate dependency in sutiations where you *can* influence the result of condition evaluation. `LoginCondition` is a good example. You generate `LoginOperation` which checks whether the user is logged in - if he is, it just happily finishes, if he's not, it presents some kind of "login view" to try to satisfy the condition. After the operation is finished, `evaluateForOperation(_:completion:)` comes in and checks if the condition was actually satisfied or not. Once again - if condition is not satisfied, the initial operation will not be even executed, so it will transit to "finished with errors" state.

So, for example:

1. **Download file only if server is reachable** - actually, don't do that. [According to Apple](https://developer.apple.com/library/ios/documentation/NetworkingInternetWeb/Conceptual/NetworkingOverview/WhyNetworkingIsHard/WhyNetworkingIsHard.html#//apple_ref/doc/uid/TP40010220-CH13-SW2):
> The SCNetworkReachability API is not intended for use as a preflight mechanism for determining network connectivity. You determine network connectivity by attempting to connect. If the connection fails, consult the SCNetworkReachability API to help diagnose the cause of the failure.

2. **Perform request only if the user is logged in** - generate some "log in operation", which will try to log in user if he's not already.
3. **Try to get the user's location only if permission to do so is granted** - generate some "location permission operation" which will ask user's permission for location if it's not already granted.

*Operation condition* is very powerful concept which extends a definition for "readiness" and allows you to seamlessly create complex and sophisticated workflows.

### Enqueuing modules

`OperacjaQueueEnqueuingModule` is just a `typealias` for

```swift
(operation: Operacja, queue: OperacjaQueue) -> Void
```

Basically, enqueuing module is a tool for adjusting your `OperacjaQueue`. Each module that you assign to queue will be called when `Operacja` is being enqueued. The most obvious scenario here is logging: you can create some kind of `LogObserver` and assign it to every `Operacja` on the queue:

```swift
let queue = OperacjaQueue()
queue.addEnqueuingModule { operation, queue in
    let logger = LogObserver(queueName: queue.name)
    operation.addObserver(logger)
}
```

Now your logger will automatically observe any `Operacja` on the queue and you will get nice visualized operations flow in your console.

Here is another example: you can create `NetworkObserver` which tracks operations and turns activity indicator on and off when appropriate (this is an example from [Advanced NSOperations][anso-url]). Then you can create `NetworkOperation` protocol, and do this to your queue:

```swift
queue.addEnqueuingModule { operation, _ in
    if operation is NetworkOperation {
        operation.addObserver(NetworkObserver())
    }
}
```

Now every `NetworkOperation` will be treated appropriately, again - automatically!

There're actually a lot of cool things you can do using enqueuing modules - use your creativity! 

### Mutual exclusivity
There are situations when you want to make sure that some kind of operations are not executed *simultaneously*. For example, we don't want two `LoadCoreDataStackOperation` running together, or we don't want one alert to be presented if there are some other alert that is currently presenting. Actually, the solution for this is very simple - if you don't want two operations to be executed simultaneously, you just make one *dependent* on another. **Operacjas** does it for you automatically. All you need to do is simply mark your `Operacja` as being mutually exclusive in some `MutualExclusivityCategory`, which is a simple protocol:

```swift
public protocol MutualExclusivityCategory {
    var categoryIdentifier: String { get }    
}
```

Make sure that your `categoryIdentifier` is *really* unique. And all `enum`s with `String` as a raw value are given automatic conformance to this! So, you just need to:

```swift
enum CoreData: String, MutualExclusivityCategory {
    case LoadStack
}

let loadStackOperation = Operacja()
loadStackOperation.setMutuallyExclusive(inCategory: CoreData.LoadStack)
// Make sure to set exclusivity before enqueueing!
queue.addOperation(loadStackOperation)
```

```swift
// More interesting example:

enum UI {
    case Alert
}

let networkAlert = NetworkUnreachableAlertOperation()
let basicAlert = BasicAlertOperation()

networkAlert.setMutuallyExclusive(inCategory: UI.Alert)
basicAlert.setMutuallyExclusive(inCategory: UI.Alert)

// will be executed one-by-one
queue.addOperations(networkAlert, basicAlert)
```

### What's out of the box?
**Operacjas** have some pretty useful stuff right out of the box:

1. Silent condition (`SilentCondition<T>`) that causes another condition to not enqueue its dependency. If we take our `LoggedInCondition` example, making `SilentCondition<LoggedInCondition>` will only check if the user is already logged in, and if he's not, it will **not** generate `LoginOperation`.
2. No cancelled dependencies condition (`NoCancelledDependencies`) specifies that every operation dependency must have succeeded without cancelling. If any dependency was cancelled, the target operation will be cancelled. Be careful, this will apply only to **cancelled** dependencies, not to the failed ones.
3. No failed dependencies condition (`NoFailedDependencies`) - kind of obvious one. Again, be careful - this will apply only to **failed** dependencies, not to the cancelled one. If you want that kind of behavior too, make sure to call `cancel(with: error)` instead of just `cancel()`.

### Tips and tricks
- If your operation failed, simply call `finish(with error: ErrorType)` method instead of just `finish()` (you can also call conform your `Operacja` to `Fallible`, and use super-sweet convenience method `finish(withError:)`, more on this [here](https://github.com/AdvancedOperations/Operations/pull/24)), that will indicate that even though your operation have entered the `finished` state, it wasn't able to do it's job.
- Of course, you can add dependencies, conditions and observers at initialization.

## Installation
**Operacjas** is available through [Carthage][carthage-url]. To install, just write into your Cartfile:

```ruby
github "AdvancedOperations/Operations" ~> 0.5.0
```

## Contributing
**Operacjas** is in early stage of development and is opened for any ideas. If you want to contribute, you can:

- Propose idea/bugfix in issues
- Make a pull request
- Review any other pull request (very appreciated!), since no change to this project is made without a PR.

Actually, any help is welcomed! Feel free to contact us, ask questions and propose new ideas. If you don't want to raise a public issue, you can reach us at [dreymonde@me.com](mailto:dreymonde@me.com).

One more: if you get a merget PR, regardless of content (typos, code, doc fixes), you will be invited to **AdvancedOperations** organizaton, because we want to make strong and vivid community of Operations-oriented programmers! ✊

## We recommend
- [Advanced NSOperations][anso-url] talk from WWDC 2015
- [MVC-N: Isolating network calls from View Controllers][mvcn-url] with Marcus Zarra 

## License
See [#11](https://github.com/AdvancedOperations/Operations/issues/11)

## Why "Operacjas"?
🇵🇱

[travis-badge]: https://travis-ci.org/dreymonde/Operacjas.svg?branch=master
[travis-url]: https://travis-ci.org/dreymonde/Operacjas
[swift-badge]: https://img.shields.io/badge/Swift-3.0-orange.svg?style=flat
[swift-url]: https://swift.org
[platform-badge]: https://img.shields.io/badge/Platform-OS%20X-lightgray.svg?style=flat
[platform-url]: https://developer.apple.com/swift/
[anso-url]: https://developer.apple.com/videos/play/wwdc2015/226/
[mvcn-url]: https://realm.io/news/slug-marcus-zarra-exploring-mvcn-swift/
[carthage-url]: https://github.com/Carthage/Carthage
[version-0.6-badge]: https://img.shields.io/badge/Operations-0.6-1D4980.svg
[danthorpe/Operations]: https://github.com/danthorpe/Operations
[releases-url]: https://github.com/AdvancedOperations/Operations/releases
