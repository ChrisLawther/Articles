# Chaining animations

When using `UIView.animate` for any sequence of animations can quickly descend into deeply nested closures. For example, here we perform three animations in sequence:

 1. Rotate a view by 90 degrees over a period of 3 seconds
 2. After a further 1 second delay, scale that view by 2x in both directions, over a period of 2 seconds
 3. Restore the original size and rotation, over a period of 1 second.

```swift
UIView.animate(withDuration: 3, animations: {
    someView.transform = CGAffineTransform(rotationAngle: .pi / 2)
}) {
    UIView.animate(withDuration: 2, delay: 1, animations: {
        someView.transform = someView.transform.scaledBy(x:2, y: 2)
    }) {
        UIView.animate(withDuration: 1, animations: {
            someView.transform = .identity
        }
    }
}
```

I think you'll agree, that's not particularly readable. What if we could describe the chain in a more linear way, and then have the whole sequence run as one? The above might then look something like:

```swift
animate(duration: 1) {
    someView.transform = CGAffineTransform(rotationAngle: .pi / 2)
}.then(duration: 2, delay: 1) {
    someView.transform = someView.transform.scaledBy(x:2, y: 2)
}.then(duration: 1) {
    someView.transform = .identity
}.run()
```

Importantly, the first link in the chain needs to know about the second link in order to run it on completion of the first link (and so on, down the chain). We therefore need to describe each link in the chain in turn, and then transform the complete chain in `.run()`.

```swift
class ChainingAnimator {
    typealias AnimationBlock = () -> Void
    
    var links = [AnimationPhase]()
    
    enum AnimationPhase {
        case simple(duration: TimeInterval,
                    delay:TimeInterval,
                    block: AnimationBlock,
                    options: UIView.AnimationOptions)
        case springy(duration: TimeInterval,
                    delay:TimeInterval,
                    damping: CGFloat,
                    velocity: CGFloat,
                    block: AnimationBlock,
                    options: UIView.AnimationOptions)
    }
}
```

And then we need a way of adding links to the chain, for both the simple and 'springy' cases:

```swift
    // In the body of `class ChainingAnimator`
    
    // 'simple' case
    func then(
        duration: TimeInterval,
        delay: TimeInterval = 0,
        options: UIView.AnimationOptions = [],
        block: @escaping AnimationBlock
    ) -> ChainingAnimator {
        phases.append(
            .simple(duration: duration,
                    delay: delay,
                    block: block,
                    options: options
            )
        )
        return self
    }
    
    // 'springy' case
    func then(
        duration: TimeInterval,
        delay: TimeInterval = 0,
        damping: CGFloat,
        velocity: CGFloat,
        options: UIView.AnimationOptions = [],
        block: @escaping AnimationBlock
    ) -> ChainingAnimator {
        phases.append(
            .springy(duration: duration,
                     delay: delay,
                     damping: damping,
                     velocity: velocity,
                     block: block,
                     options: options
            )
        )
        return self
    }
```

Next, we need a way to invoke the chain as a whole, by walking down the chain from the end back to the start, using each animation (and it's completion) as the completion for the earlier link in the chain. The `run()` function we add to `ChainingAnimator` achieves this and also allows us to provide a closure to run when the whole sequence has ended:

```swift
    func run(then onComplete: @escaping () -> Void = {}) {
        let animationSequence = phases.reversed().reduce(onComplete) { completion, link in
            link.block(withCompletion: completion)
        }

        animationSequence()
    }
```

There's one final thing though. We haven't implemented `.block` on `AnimationPhase` yet. That needs to merge the previously unknown completion with the specification of the current link to produce a runnable closure:

extension ChainingAnimator.AnimationPhase {
    func block(withCompletion completion: @escaping () -> Void) -> () -> Void {
        switch self {

        case .simple(let duration, let delay, let block, let options):
            return {
                UIView.animate(withDuration: duration,
                               delay: delay,
                               options: options,
                               animations: block,
                               completion: { _ in completion() })
            }

        case .springy(let duration, let delay, let damping, let velocity, let block, let options):
            return {
                UIView.animate(withDuration: duration,
                               delay: delay,
                               usingSpringWithDamping: damping,
                               initialSpringVelocity: velocity,
                               options: options,
                               animations: block,
                               completion: { _ in completion() })
            }
        }
    }
}

The completed code is available [as a gist on GitHub][1]

[1]: https://gist.github.com/azureblue75/a0125487e3c2339bb80d513634c29b4e

