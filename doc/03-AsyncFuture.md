# Async Tasks and Futures

## Introduction

Running threads directly from `std::thread` does have some drawbacks, particularly because there's no direct way to get the return value of the function the thread executed (which in scientific computing it often what we really want). Using messages or shared objects may not be the safest or easiest way of doing this. However, the standard library offers some more convenient ways to manage this simple case.

## Async Tasks and Futures 

An `async` task is very like launching a thread - it has the same syntax, giving the name of the function to call and any arguments it should take. However, the return value from an `async` task is a special class template, a `future`, in which the return value will
only be placed once the task has executed and returned.

So, we just launch a thread as an `std::async` object and assign the return value to the `std::future` templated class. E.g.,

```cpp
#include <future>

double some_expensive_operation(std::vector<thing>& stuff) {
    // Do expensive things to stuff
    ...

    return result;
}

int main() {
	auto my_stuff = initialise_stuff();

    std::future<double> res = std::async(some_expensive_operation, std::ref(my_stuff));

    // do some other things

    double the_answer = res.get();
    ...
```

You can see that the value of the variable is accessed with the `get()` method of the `future`.

A future also has the `wait()` method, which will block the calling thread until the result of the asynchronous execution is available, but it doesn't read the result. (There are also `wait_for()` and `wait_until()` which will wait until the result is ready, but not wait indefinitely).

Note that after a future's result has been retrieved further calls to `get()` or `wait()` will thrown an `std::future_error`. If your program could potentially retrieve the future result like this then use the `valid()` method to test before you do so.

### Launch Parameter

If you run an `async` task, you might be surprised that it doesn't run in the background -- it might well run only when the future's get method is called. This is because the implementation can define when it runs an asynchronous task, either immediately or deferred. However, if the first parameter passed to the invocation of the task is an `std::launch` parameter, this can be controlled manually.

In particular `std::launch::async` says that the asynchronous thread should execute immediately, where as `std::launch::deferred` will delay execution until either `wait()` or `get()` is called on the future. The default value is `std::launch::async | std::launch::deferred`, which is why the implementation can choose what happens. In particular, gcc4.8 and earlier will not run a task asychronously unless explicitly asked too - later versions of gcc behave more sensibly.

Obviously using `std::launch::async` could always be used if you know there's no point in deferring execution, however, you should beware of breaking programs in a single threaded environment, so use that option with caution. An advantage of `std::launch::deferred` is that if it turns out that the async task isn't needed then it just won't be run at all when the future goes out of scope.

### Exceptions

If the async task results in an unhandled exception then the call to `get()` or `wait()` will raise that exception again in the thread owning the future. Thus exception handling for async tasks should be identical to sychronous execution.

## Packaged Tasks and Promises

There are other ways to set the values associated with a `future`. One is to setup a `packaged_task` that allows a task to be tied to a `future` _before_ it is sent off for dispatch - this is useful if you have a task queue, but need to make sure you can get a handle on the result of the function before it is sent to the queue, which a `packaged_task` allows you to do with its `get_future()` method.

For a packaged task, instead of passing arguments directly to the function (as with `async`), construct with just the callable entity. Then, later on, invoke the call method of the `packaged_task` itself (with any arguments) to start the task's execution:

```cpp
	double my_func(double) {
	    ...
    }
    
    // Construct the packaged task for function my_func
    pt = std::packaged_task(my_func);
    
    ...
    
    // Associate the packaged task's shared state with a future
    // (Note, get_future() can only be called once)
    std::future<double> the_answer = pt.get_future();
    
    // Run the packaged task with this argument - also can be in a different thread
    pt(3.1415)
    
    ...
    
    try{
        double d = the_answer.get();
    } catch (...) {
        // Handle any exception
    }
``` 

# Exercises

1. Write a small program that executes some numerical calculation using an `std::async` task to return the value as an `std::future`. (You might use the classic monte-carlo determination of pi, picking a random `x` and `y` in [0, 1] and seeing if the point is within a circle of radius 1. The ratio of points in the circle to the total number of points is pi/4.)
    1. Does the task run asynchronously without an `std::launch` parameter?
    2. Use the `std::launch` parameter to control execution and force the task to run asynchronously.
    3. Can you demonstrate that a task launched with `deferred` where the future is not interrogated actually never runs?

2. Generalise the above exercise and use `std::async` to fill all the cores on your machine.
    1. You can use `std::thread::hardware_concurrency` to allow the program to estimate the correct number of threads to launch.
