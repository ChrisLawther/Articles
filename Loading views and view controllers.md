# Loading views and view controllers

On iOS views and view controllers can be loaded in a variety of ways:

 * A view controller could be defined in a Storyboard using Interface Builder
 * A custom view could have it's own `xib` (`nib`) file
 * A view could be defined entirely in code

## Loading from a Storyboard
 
To load a view controller from a Storyboard (perhaps you like Storyboards for designing views, but also enjoy the flexibility of controlling transitions between view controllers in code, maybe using the coordinator pattern), we can get a reference to the relevant storyboard and ask it to instantiate the view controller we want:

```swift
let storyboard = UIStoryboard(name: "Main", bundle: nil)
let vc = storyboard.ininstantiateViewController(withIdentifier: "MyViewController") as? MyViewController
```

This requires us to have remembered to set a storyboard identifier in Interface Builder. If there are no transitions pointing to the view controller and it does not have an identifier, Xcode will helpfully alert us to it's unreachability.

Now, how could we make that more convenient? Could we describe what we need to know about a view controller in a protocol, with a default implementation that requires no custom conformance?

```swift
protocol StoryboardInitializable {
    static var storyboardName: String { get }
    static var storyboardIdentifier: String { get }
}

extension StoryboardInitializable {
    /// Default implementation, assumes Main.storyboard
    static var storyboardName: String { return "Main" }

    /// Default implentation, expects identifier to match class name
    static var storyboardIdentifier: String { return String(describing: self) }
}
```

Now, a view controller that chooses to conform to `StoryboardInitializable` automatically gains two properties:

 * `storyboardName`, which defaults to "Main" (but can be overridden for larger projects)
 * `storyboardIdentifier`, which default to the classname of the View Controller (and again, can be overridden if they don't match for any reason)

All that is missing now is a way to ask for a conforming view controller class to be loaded:

```swift
extension StoryboardInitializable {
    static func fromStoryboard() -> Self {
        let storyboard = UIStoryboard(name: storyboardName, bundle: nil)
        guard let svc = storyboard.instantiateViewController(withIdentifier: storyboardIdentifier) as? Self else {
            fatalError("No view controller with identifier '\(storyboardIdentifier)' in Storyboard '\(storyboardName)'")
        }
        return svc
    }
}
```

Great. With that in place, we can opt-in to this convenience simply by claiming conformance:

```swift
class MyViewController: UIViewController, StoryboardInitializable {
    ...
}
```

Usage is then:

```swift
let myVc = MyViewController.fromStoryboard()
```

## Loading from NIB

This is already much more straightforward and arguably doesn't need simplifying, but let's see what the same approach yields here.

```swift
protocol NibInitializable {
	static var nibName: String { get }
}

extension NibInitializable {
    /// Default implentation, expects nib file name to match class name
    static var nibName: String { return String(describing: self) }
}

extension NibInitializable {
    static func fromNib() -> Self {
    	guard let view = Bundle.main.loadNibNamed(nibName,
    	                                          owner: nil,
    	                                          options: nil)!.first as? Self else {
    	    fatalError("Couldn't instantiate view from nib \(nibName)")
		}
		return view
    }
}
```

Once again, in most cases opting-in requires simply claiming conformance to the protocol and relying on the default implementation. If that scenario describes all of your use-cases, then `fromNib` can be rewritten as a simpler extension on `UIView`:

```swift
extension UIView {
    class func fromNib<T: UIView>() -> T {
    	guard let view = Bundle.main.loadNibNamed(String(describing: Self),
    	                                          owner: nil,
    	                                          options: nil)!.first as? Self else {
    	    fatalError("Couldn't instantiate view from nib \(nibName)")
		}
		return view
    }
}
```

