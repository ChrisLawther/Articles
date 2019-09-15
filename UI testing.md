UI Testing
==========

If we want to perform tests, potentially many of them, on a ViewController that is deep within our application, we could save time and reduce test complexity by being able to control at launch time which ViewController is initially displayed and what data it is presenting.

NOTE: This approach works best if you **don't** have a main interface storyboard configured and are instead defining your app structure in code, perhaps using the Coordinator pattern. Otherwise, you will have some initial UI that doesn't belong to the `UIViewController` of interest and your special-case startup logic will have to replace that unwanted UI before your tests can proceed.	

To get started, we need to pass arguments to the application from our tests. Rather than modify the default `override func setup() {...}` which takes no arguments, we'll lay the foundations for something more flexible:

```swift
class MyAppUITests: XCTestCase {
	var app: XCUIApplication!

	private func launchApp() {
		app = XCUIApplication()
		app.launchArguments.append("--uitesting")
		app.launch()
	}
}
```

NOTE: This does mean we need to remember to call `launchApp` from each test though...
	
In our `AppDelegate` we need to be able to detect that we should not perform conventional app launch, but instead present whichever view is being asked for, perhaps resulting in an `AppDelegate` something like this:

```swift
class AppDelegate: UIResponder, UIApplicationDelegate {
	var window: UIWindow?

	var rootCoordinator: Coordinator!

	func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
		// Override point for customization after application launch.

		window = UIWindow()
		window?.makeKeyAndVisible()

		#if DEBUG
		if CommandLine.arguments.contains("--uitesting") {
			rootCoordinator = TestingCoordinator(window: window!)
			return true
		}
		#endif

		rootCoordinator = MyApp(window: window!)

		return true
	}	
}
```

Our `Testing` object needs to understand how to process additional command line arguments specifying which `UIViewController` to display. Making use of the fact that command line arguments which specify values (i.e. `-key value`) are included in  `UserDefaults`, we can easily determine what is being requested.

NOTE: Checking for the presence of arguments using `CommandLine.arguments.contains("...")` gives full visibility of the parameter exactly as passed and requires manual parsing of any key/value combinations. Using `UserDefaults.standard.string(forKey: someKey)`, a leading `-` will have been stripped off the parameter in order to generate the corresponding key. In other words, the value passed with `-vc` can be accessed via the key `vc`.

Let's pass an additional argument requesting a specific initial `UIViewController` by modifying `launchApp`:

```swift
func launchApp(withArgs args: [String] = []) {
	app = XCUIApplication()
	app.launchArguments.append("--uiTesting")
	app.launchArguments.append(contentsOf: args)
	app.launch()
}

func testSubscriptionVC() {
	launchApp(withArgs: ["-vc", "subscriptions"])
	
	// Inspect view
	// XCTAssertEqual(...)
}
```	

In our `TestingCoordinator` we implement some logic to respect the request:

```swift
init(window: UIWindow) {
	let vcName = UserDefaults.standard.string(forKey: "vc")
	
	let vc: UIViewController
	
	switch vcName {
		case "subscriptions":
			vc = SubscriptionsViewController.fromStoryboard()
		case "options":
			vc = OptionsViewController.fromStoryboard()    		
		default:
			fatalError("Unexpected VC '\(vcName)' was requested"
	}
	
	window.rootViewController = vc
}
```

NOTE: `.fromStoryboard()` is from an extension `StoryboardInitializable`, described in [Loading views and view controllers][1]

[1]: "Loading%20views%20and%20view%20controllers.md"

Passing data at the commandline
-------------------------------

OK, so we've got a way for our UI tests to request a particular view, without the overhead of each test having to navigate to it. In order to perform meaningful tests, we also need to pass data for that view. In order to reliably pass data via the command line, we need a robust, string-based encoding of our data. JSON may at first seem like a good candidate, but that introduces problems with escaping quotes and spaces, so we choose base64 instead - no spaces, no special characters - perfect!

Let's begin by adding an extension on `Encodable` to conveneiently give us the base64 representation of any object which adopts it:
	
```swift
public extension Encodable {
	func encoded() -> Data? {
		return try? JSONEncoder().encode(self)
	}

	func base64Encoded() -> String? {
		return encoded()?.base64EncodedString()
	}
}
```

Along similar lines, we also want a convenient way to turn a base64 string back into the object we expect it to represent:

```swift
public extension Data {
	func decoded<T: Decodable>() -> T? {
		return try? JSONDecoder().decode(T.self, from: self)
	}
}

public extension String {
	func decoded<T: Decodable>() -> T? {
		return Data(base64Encoded: self)?.decoded()
	}
}
```

As our UI tests run as a separate process to the app which we are testing, the best way to make our models visible to it is simply to add them to the testing target:

![UI test target membership][2]

[2]: ./uiTestingTargetMembership.png

With those in place we can pass model data from our UI-test function to our app at launch:

```swift
func testShowsCorrectDetails() {
	let model = SomeModel(name: modelName)
	let base64 = try! model.base64Encoded()

	launchApp(withArgs: ["-vc", "someModelDetail", "-model", base64])

	XCTAssertEqual(app.staticTexts["modelName"].label, model.name)
}
```

Also, we expand our `TestingCoordinator` implementation of `init` to apply the specified data to the requested view:

```swift
init(window: UIWindow) {
	let vcName = UserDefaults.standard.string(forKey: "vc")
	
	let vc: UIViewController
	
	do {
		switch vcName {
			case "someModelDetail":
				vc = ModelDetailViewController.fromStoryboard()
				guard let modelString = UserDefaults.standard.string(forKey: "model") else {
					fatalError("No parameter passed for 'model'")				
				}
				guard let model: SomeModel = try? modelString.decoded() else {
					fatalError("Couldn't decode expected model")
 				}
				vc.model = model
			
			case "options":
				vc = OptionsViewController.fromStoryboard()    		
			default:
				fatalError("Unexpected VC '\(vcName)' was requested")
		}
	} catch {
		fatalError("")
	}
			
	window.rootViewController = vc
}
```

NOTE: `TestingCoordinator`, as currently written, is ripe for refactoring. Adding more view controllers and models is quickly going to yield a lot of very similar code. `fatalError` is chosen over attempting to handle errors as any error encountered is a coding error and must be addressed via a code change.
