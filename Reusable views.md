# Reusable views

For performance reasons, `UITableView` and `UICollectionView` have a mechanism for reusing cells as the view is scrolled. Cells for rows going out of view are added to a pool of cells available for reusing by other rows as they come into view. Custom cells created programatically or loaded from NIBs can be opted-in to this mechanism by registering them with the `UITableView` in which they will appear. This mechanism exists for both rows and headers/footers:

```swift
// Cells
register(MyTableCell, forCellReuseIdentifier: "myCellIdentifier")

let nib = UINib(nibName: "MyTableCell", bundle: nil)
register(nib, forCellReuseIdentifier: "myCellIdentifier")

// Headers
register(MyHeader, forHeaderFooterViewReuseIdentifier: "myHeaderIdentifier")

let nib = UINib(nibName: "MyHeader", bundle: nil)
register(nib, forHeaderFooterViewReuseIdentifier: "myCellIdentifier")
```

Alarm bells might be ringing at the sight of `String` literals being used as identifiers. You could define a constant and that would provide a degree of safety, but you'd still need to make sure you used the right identifier and the name could only be _like_ the cell class name.

So, what if we could eliminate the identifier from both the registration and reuse of cells? Focussing on programatically generated row cells for now, maybe the call site should look something like this:

```swift
// Registration
tableView.register(MyCustomCell)

// Re-use
let cell = tableView.dequeueReusableCell(MyCustomCell.self, for: indexPath)
```

That's definitely an improvement, but it wouldn't prevent us from trying to dequeue a cell which had never been registered. How about this instead:

```swift
// Registration
let customCellToken = tableView.register(MyCustomCell)

// Re-use
let cell = tableView.dequeueReusableCell(customCellToken, for: indexPath)
```

Now we can be certain that:

  * we'll never(*) be asked to dequeue a cell which hasn't been registered
  * we can force-cast the dequeued cell to the expected type since the identifier/cell class relationship can be trusted

* - somebody *could* create their own token, but that would be deliberately fighting against the API, rather than innocently making a mistake.

As is often the case, this looks like a good case for a `protocol`, with a default implementation which uses the classname as the identifier:

```swift
protocol IdentifiedReusableView {
    static var identifier: String { get }
}

extension IdentifiedReusableView {
    static var identifier: String {
        return String(describing: self)
    }
}
```

Since we want this to work for cells and headers/footers, but with no possibility of using one in place of the other, let's define a protocol for each:

```swift
// Allows us to prevent attempts to request a UITableViewCell in a context expecting a
// UITableViewHeaderFooterView and vice-versa.
protocol IdentifiedCell: UITableViewCell, IdentifiedReusableView { }
protocol IdentifiedHeaderFooter: UITableViewHeaderFooterView, IdentifiedReusableView { }
```

Registration of such cells can then be:

```swift
struct ReuseToken<T> { }

extension UITableView {
    func register<T: IdentifiedCell>(_ cellType: T.Type) -> ReuseToken<T> {
        register(cellType, forCellReuseIdentifier: cellType.identifier)
        return ReuseToken<T>()
    }
}

And dequeueing makes use of the type information captured by the token:

```swift
extension UITableView {
    func dequeueReusableCell<T: IdentifiedCell>(token: ReuseToken<T>, for indexPath: IndexPath) -> T {
        return dequeueReusableCell(withIdentifier: T.identifier, for: indexPath) as! T
    }
}
```

That's all that's required for programmatic cells, but there are other ways of defining cells:

  * Interface Builder NIBs
  * Storyboard prototype cells
  
### NIBs

Since cells defined in NIBs have a similar registration mechanism, we can extend our base protocol to provide the necessary information, again with a default implementation:

```swift
protocol IdentifiedReusableView {
    static var identifier: String { get }
    static var nib: UINib { get }
}

extension IdentifiedReusableView {
    static var nib: UINib {
        return UINib(nibName: String(describing: self), bundle: nil)
    }
}
```

The `UITableView` extension then gains a function to make use of this:

```swift
    func registerNib<T: IdentifiedCell>(for cellType: T.Type) -> ReuseToken<T> {
        register(cellType.nib, forCellReuseIdentifier: cellType.identifier)
        return ReuseToken<T>()
    }
```

Note that the returned token is of the same type we saw before. Dequeueing s the same, regardless of the origin of the cells.

### Storyboard prototypes

Cells designed within the `UITableView` in Interface Builder are an exception that slightly compromises all the nice type-safety and de-duplication we've achieved. We can't know or enforce the cell identifier that the designer used, but we can still use the `IdentifiedCell` protocol to avoid using string literals at the call site:

```swift
extension UITableView {
    func dequeueReusableCell<T: IdentifiedCell>(_: T.Type, for indexPath: IndexPath) -> T {
        // swiftlint:disable:next force_cast
        return dequeueReusableCell(withIdentifier: T.identifier, for: indexPath) as! T
    }
}
```

Unlike for programmatic or NIB-based cells where the identifier is only defined in one place, in this case we have to unavoidably trust the user to specify a matching identifier in Interface Builder. Defaulting to the classname hopefully increases the likelihood that they will be successful.

### Header/Footer views

Header and footer views (implemented as a common type) employ the same reuse mechanism, differing only in slight variations to function names (`dequeueReusableHeaderFooterView` v.s. `dequeueReusableCell`) and parameter names (`forHeaderFooterViewReuseIdentifier` v.s. `forCellReuseIdentifier`), so we can apply the same approach to their registration and dequeueing.

The completed source code can be found [in this gist][1].

[1]: https://gist.github.com/azureblue75/89eb0a1bc6df5d1db9f2fd8416baa27f
