# Optionals and exceptions

Even now, with Swift reaching version 5, there is still some confusion over optionals, leading to a particular pattern of redundant code being written.

Let's establish some foundations for this illustration. Here's a class with a single method and a free function that returns an instance of that class half of the time:

```swift
class Thing {
    func doStuff() {
        print("Hello")
    }
}

func maybeMakeThing() -> Thing? {
    return Bool.random() ? Thing() : nil
}

let optionalThing = maybeMakeThing()
```

So far, so good. Now, what if we wanted to safely call `doStuff` only if we've actually got an instance of `Thing`?

We could check it's not `nil` and force-unwrap it:

```swift
if optionalThing != nil {
    optionalThing!.doStuff()
}
```

Or we could attempt to unwrap it and only call the method if unwrapping succeeds:

```swift
if let actualThing = optionalThing {
    actualThing.doStuff()
}
```

Or we could just do the Swift-y thing and make use of optional chaining the way it's intended:

```swift
optionalThing?.doStuff()
```

I've seen examples from well known tutorial providers which still get that wrong and perpetuate the misunderstanding by advocating either the `nil` check or the unwrapping.
