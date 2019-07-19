# Concurrency

Swift offers a number of concurrency options varying in their simplicity and flexibility (for example `OperationQueue` is very simple to use but provides no way to remove a queued operation).

## Methods

### OperationQueue

At it's simplest, [OperationQueue][4] provides a mechanism for queueing up items of work to be performed with a controllable degree of concurrency.

```swift
let queue = OperationQueue()

queue.addOperation {
    // Code that does some significant work
}

queue.waitUntilAllOperationsAreFinished()
```

Subclassing [`Operation`][5] allows greater control and enables data to be passed into the operation prior to starting it (generally via a custom initializer), extracted from it after completion (via getter methods) and provides visibility of `.isCancelled` for interrupting an already started operation. The block-based `.addOperation` doesn't allow the individual operation to be cancelled or for the operation itself to have visibility of `.isCancelled` in the event that the whole queue was cancelled.

### DispatchGroup

A [DispatchGroup][2] provides a way to perform a set of tasks and call a handler once they are all complete, or wait synchronously for them to complete. It doesn't do anything for you relating to threading or scheduling, or even have any strong concept of tasks. What it does do is let you schedule some code to run when all tasks have completed:

```swift
let group = DispatchGroup()

group.notify(queue: .main) {
    print("All download tasks have completed")
}

for url in urls {
    group.enter()
    URLSession.shared.downloadTask(with: url, completionHandler: { (tempURL, response, error) in
        // Handle the response/error
        ...
        // Signal that we have finished
        group.leave()
    }).resume()
}

```

## Supporting classes

Various classes exist to support concurrency, perhaps to provide signalling or synchronisation.

### DispatchSemaphore

A [DispatchSemaphore][1] has an initial value (often zero, depending on the desired behaviour) and can be incremented (`signal()`) or decremented (`wait()`), optionally with a timeout. Logically, the various forms of `wait()` block until a `signal()` if decrementing would take the value negative.



## Examples

[DownloadSequentially][2] uses `OperationQueue` to perform a number of downloads sequentially, but since `URLSession` download tasks happen asynchronously, `DispatchSemaphore` is employed to ensure a download only starts when it's predecessor finishes. While it may seem odd to be using a concurrency mechanism to avoid concurrency(!), that code was written as a response to [an article advocating a far more involved approach][3], so see it more as an example of semaphore usage in asynchronous code (and of course, increasing `.maxConcurrentOperationCount` will allow multiple simultaneous downloads, although `URLSessionConfiguration.httpMaximumConnectionsPerHost` is the simplest solution in this specific case).

[1]: https://developer.apple.com/documentation/dispatch/dispatchsemaphore
[2]: https://gist.github.com/azureblue75/9b20e0159b35c6037add476f2700132d
[3]: https://fluffy.es/download-files-sequentially/
[4]: https://developer.apple.com/documentation/foundation/operationqueue
[5]: https://developer.apple.com/documentation/foundation/operation