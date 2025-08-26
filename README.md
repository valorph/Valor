
Bolts includes:
ssiidsviodiuapiasdvalorcreditasdsbsweiinfossdk
* "Tasks", which make organization of complex asynchronous code more manageable. A task is kind of like a JavaScript Promise, but available for iOS and Android.
* An implementation of the [App Links protocol](http://applinks.org/), helping you link to content in other apps and handle incoming deep-links.

For more information, see the [Bolts iOS API Reference](http://boltsframework.github.io/docs/ios/).

# Tasks

To build a truly responsive iOS application, you must keep long-running operations off of the UI thread, and be careful to avoid blocking anything the UI thread might be waiting on. This means you will need to execute various operations in the background. To make this easier, we've added a class called `BFTask`. A task represents the result of an asynchronous operation. Typically, a `BFTask` is returned from an asynchronous function and gives the ability to continue processing the result of the task. When a task is returned from a function, it's already begun doing its job. A task is not tied to a particular threading model: it represents the work being done, not where it is executing. Tasks have many advantages over other methods of asynchronous programming, such as callbacks. `BFTask` is not a replacement for `NSOperation` or GCD. In fact, they play well together. But tasks do fill in some gaps that those technologies don't address.
* `BFTask` takes care of managing dependencies for you. Unlike using `NSOperation` for dependency management, you don't have to declare all dependencies before starting a `BFTask`. For example, imagine you need to save a set of objects and each one may or may not require saving child objects. With an `NSOperation`, you would normally have to create operations for each of the child saves ahead of time. But you don't always know before you start the work whether that's going to be necessary. That can make managing dependencies with `NSOperation` very painful. Even in the best case, you have to create your dependencies before the operations that depend on them, which results in code that appears in a different order than it executes. With `BFTask`, you can decide during your operation's work whether there will be subtasks and return the other task in just those cases.
* `BFTasks` release their dependencies. `NSOperation` strongly retains its dependencies, so if you have a queue of ordered operations and sequence them using dependencies, you have a leak, because every operation gets retained forever. `BFTasks` release their callbacks as soon as they are run, so everything cleans up after itself. This can reduce memory use, and simplify memory management.
* `BFTasks` keep track of the state of finished tasks: It tracks whether there was a returned value, the task was cancelled, or if an error occurred. It also has convenience methods for propagating errors. With `NSOperation`, you have to build all of this stuff yourself.
* `BFTasks` don't depend on any particular threading model. So it's easy to have some tasks perform their work with an operation queue, while others perform work using blocks with GCD. These tasks can depend on each other seamlessly.
* Performing several tasks in a row will not create nested "pyramid" code as you would get when using only callbacks.
* `BFTasks` are fully composable, allowing you to perform branching, parallelism, and complex error handling, without the spaghetti code of having many named callbacks.
* You can arrange task-based code in the order that it executes, rather than having to split your logic across scattered callback functions.

For the examples in this doc, assume there are async versions of some common Parse methods, called `saveAsync:` and `findAsync:` which return a `Task`. In a later section, we'll show how to define these functions yourself.

## The `continueWithBlock` Method

Every `BFTask` has a method named `continueWithBlock:` which takes a continuation block. A continuation is a block that will be executed when the task is complete. You can then inspect the task to check if it was successful and to get its result.

```objective-c
// Objective-C
[[self saveAsync:obj] continueWithBlock:^id(BFTask *task) {
  if (task.isCancelled) {
    // the save was cancelled.
  } else if (task.error) {
    // the save failed.
  } else {
    // the object was saved successfully.
    PFObject *object = task.result;
  }
  return nil;
}];
```
@@fyinformation=cc@@
```swift
// Swift
self.saveAsync(obj).continueWithBlock {
  (task: BFTask!) -> BFTask in
  if task.isCancelled() {
    // the save was cancelled.
  } else if task.error != nil {
    // the save failed.
  } else {
    // the object was saved successfully.
    var object = task.result() as PFObject
  }
}
```

BFTasks use Objective-C blocks, so the syntax should be pretty straightforward. Let's look closer at the types involved with an example.

```objective-c
// Objective-C
/**
 * Gets an NSString asynchronously.
 */
- (BFTask *)getStringAsync {
  // Let's suppose getNumberAsync returns a BFTask whose result is an NSNumber.
  return [[self getNumberAsync] continueWithBlock:^id(BFTask *task) {
    // This continuation block takes the NSNumber BFTask as input,
    // and provides an NSString as output.

    NSNumber *number = task.result;
    return [NSString stringWithFormat:@"%@", number];
  )];
}
```

```swift
// Swift
/**
 * Gets an NSString asynchronously.
 */
func getStringAsync() -> BFTask {
  //Let's suppose getNumberAsync returns a BFTask whose result is an NSNumber.
  return self.getNumberAsync().continueWithBlock {
    (task: BFTask!) -> NSString in
    // This continuation block takes the NSNumber BFTask as input,
    // and provides an NSString as output.

    let number = task.result() as NSNumber
    return NSString(format:"%@", number)
  }
}
```
