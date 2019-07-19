# Notifications

Notifications come in two distinct flavours:

 * System notifications - use the `.userInfo` `[String:Any]` dictionary property to convey information
 * Custom notifications - *could* use `.userInfo`, but often simply attach an instance of a custom type to the `.object` property, although this does go against the expectation that it be a record of the poster of the notification.

Here we'll explore notifications using the `.object` approach first, before going on to cover a technique for handling notifications that make use of `.userInfo`, regardless of their origin.
 
## .object-as-payload notifications 

The default way to post and be notified of a custom notification might look something like this:

```swift
struct MyNotification {
    let status: Int
    let message: String
}

// Register a block for handling notifications. Don't forget to
// remove the observer when we are deallocated though!

let observer = NotificationCenter.default.addObserver(forName: "MyNotificationDidUpdate") { note in
    guard let msg = note.object as? MyNotification else {
        // We received a notification with the right name, but an unexpected
        // object type was attached to it
        return
    }
    
    // Handle msg
}

// Post
let notification = MyNotification(status: 0, message: "All good")

NotificationCenter.default.post("MyNotificationDidUpdate", object:  notification)
```

There's a couple of things about that which aren't ideal:

  * we need to remember to unregister ourselves as an observer when we go out of scope, otherwise NotificationCenter will try to deliver notifications to something that doesn't exist.
  * we've got string-based identifiers in two places that need to match.

To fix the first of those, we'll borrow the [self un-registering token idea][1] from episode 27 of [Swift Talk][2]:

```swift
class Token {
    let token: NSObjectProtocol
    let center: NotificationCenter
    init(token: NSObjectProtocol, center: NotificationCenter) {
        self.token = token
        self.center = center
    }

    deinit {
        center.removeObserver(token)
    }
}
```
  
We could use a constant for the identifier, but should we have to? We already know the type of object we want to send and receive, so maybe it can do the work for us? Let's define a protocol for types that we wish to send to conform to and extend `NotificationCenter` with functions to post and observe those types:

```swift
protocol Postable {
    static var name: Notification.Name { get }
}

extension NotificationCenter {
    func post<T: Postable>(_ notification: T) {
        post(name: T.name, object: notification)
    }
}

extension NotificationCenter {
    func addObserver<N: Postable>(using block: @escaping (N) -> ()) -> Token {
        return Token(token: addObserver(forName: N.name, object: nil, queue: nil, using: { note in
            block(note.object as! N)
        }), center: self)
    }
}
```

With that in place, sending and receiving our previous notification becomes:

```swift
extension MyNotification {
    static let name = Notification.Name("MyNotificationDidUpdate")
}

var token = NotificationCenter.default.addObserver { (notification: MyNotification) in
    print("Received '\(notification.message) with status \(notification.status)")
}

let notification = MyNotification(status: 1, message: "Download complete")

// Closure above will receive the notification
NotificationCenter.default.post(notification)

// Cause the token to auto-unregister itself
token = nil

// Nobody will receive this notification
NotificationCenter.default.post(notification)
```

## .userInfo-based notifications

At the call-site, we'd like to be shielded from the `[String:Any]` dictionary that arrives as part of a system notification, and we'd like to make use of a self-unregistering observer token, as before. In order to receive such a notification, that translation from `[String: Any]` does need to happen somewhere, so we define a protocol which requires conforming types to be initializable from such a dictionary:

```swift
protocol ReceivableNotification {
    static var name: Notification.Name { get }
    init(notification: Notification)
}
```

For example, to receive the (very simple) `PlaygroundPageNeedsIndefiniteExecutionDidChangeNotification`, we would write:

```swift
struct PlaygroundPageNeedsIndefiniteExecution {
    let needsIndefiniteExecution: Bool
}

extension PlaygroundPageNeedsIndefiniteExecution: ReceivableNotification {
    static let name = Notification.Name("PlaygroundPageNeedsIndefiniteExecutionDidChangeNotification")

    init(notification note: Notification) {
        needsIndefiniteExecution = note.userInfo!["PlaygroundPageNeedsIndefiniteExecution"] as! Bool
    }
}
```

To register as an observer which receives the result of initializing our `struct` from the notification:

```swift
extension NotificationCenter {
    func addObserver<N: ReceivableNotification>(using block: @escaping (N) -> ()) -> Token {
        return Token(token: addObserver(forName: N.name, object: nil, queue: nil, using: { note in
            block(N.init(notification: note))
        }), center: self)
    }
}
```

Usage is then:

```swift
let token = NotificationCenter.default.addObserver { (note:PlaygroundPageNeedsIndefiniteExecution) in
    print("Indefinite now: \(note.needsIndefiniteExecution)")
}
```

There is definitely scope for improving this approach as it currently has a couple of weaknesses relating to the (ab)use of `.object`.

 * The sender of a System notification is inaccessible
 * We can't set the sender (`.object`)
 
## Custom notifications, improved version

This approach aims to:

 * facilitate the convenient, type-safe sending and receiving of custom notifications
 * preserve the option of reporting the sending object
 * not require sub-classing NSNotification

To achieve that, we wrap the custom notification type in a struct along with the sender:

```swift
struct NotificationWrapper<T> {
    let sender: Any?
    let notification: T
}
```

We define a protocol for custom notification types to adopt and extend `NotificationCenter` to make posting and observing these custom notifications whilst transparently wrapping and unwrapping them:

```swift
protocol CustomNotification {
    static var notificationName: Notification.Name { get }
}

extension NotificationCenter {
    func post<T: CustomNotification>(notification: T, sender: Any? = nil) {
        let wrapped = NotificationWrapper(sender: sender, notification: notification)
        post(name: T.notificationName, object: wrapped)
    }

    func addObserver<T: CustomNotification>(using block: @escaping (Any?, T) -> ()) -> Token {
        return Token(token: addObserver(forName: T.notificationName, object: nil, queue: nil, using: { note in
            guard let wrapped = note.object as? NotificationWrapper<T> else { return }
            block(wrapped.sender, wrapped.notification)
        }), center: self)
    }
}
```

With that in place, we can define and send and receive our custom notifications and the associated sender:

```swift
struct MyNotification {
    let status: Int
    let message: String
}

extension MyNotification: CustomNotification {
    static let notificationName = Notification.Name("MyNotificationDidHappen")
}

var token = NotificationCenter.default.addObserver { (sender: Any?, notification: MyNotification) in
    print("\(sender) says \(notification.status) \(notification.message)")
}

let note = MyNotification(status: 123, message: "Looking good")

NotificationCenter.default.post(notification: note, sender: self)
```

[1]: https://talk.objc.io/episodes/S01E27-typed-notifications-part-1
[2]: https://talk.objc.io

