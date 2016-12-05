# koroutine
Small, lightweight coroutine scheduler for node.js based on ES6 generators

##Table of Contents

- [Install](#install)
- [Introduction](#introduction)
- [Sequential Async Calls Example](#sequential-async-calls-example)
- [Parallel Async Calls Example](#parallel-async-calls-example)
- [Koroutine Library Object Methods](#koroutine-library-object-methods)
- [Coroutine Context Methods](#coroutine-context-methods)

## Install

```sh
$ npm install koroutine
```

## Introduction

This is a 100% javascript implementation of coroutine scheduler based on ES6 generators. It can be used 
to replace callback spaghetti async code with simpler, sequential looking code that is still 100% async.
You can either make async calls one after other or fire bunch of async calls in parallel and wait for all
of them to complete before retrieving results/errors for each of the calls.

## Sequential async calls example

Use `Koroutine.run(generator_funtion, timeout, argument_1, argument_2, ...)` to run any ES6 generator function 
as a coroutine. It runs your generator function with `this` bound to the coroutine context while passing in all the
arguments passed to run() after the second parameter `timeout` as function arguments. You can then pass `this.resume` as a 
callback to any async function you may want to call from inside of the `generator_function`. `resume` follows Node's callback 
convention - i.e. first parameter is error followed by results or data parameters. If the async function returns an error, it 
is thrown as an exception inside the generator function body as shown below.

The second parameter `timeout` is the maximum amount of time in milliseconds that the coroutine is allowed to run. If it 
runs for more than timeout milliseconds, an exception with cause = "timedout" is thrown inside the generator function. 
timeout=0 means no maximum limit (infinite timeout).

```js
const ko = require('koroutine');

function dummyAsyncSuccessCall(input, callback, delay) {
    setTimeout(function() {
        const result = input+"-ok";
        callback(null, result);
    }, delay);
}

function dummyAsyncErrorCall(input, callback, delay) {
    setTimeout(function() {
        callback(new Error(input+"-error"));
    }, delay);
}


function* exampleKoroutine(input1, input2) {
    try {
        const result1 = yield dummyAsyncSuccessCall(input1, this.resume, 1000);
        console.log(result1)
        yield dummyAsyncErrorCall(input2, this.resume, 2000);
    } catch (e) {
        console.log(e);
    }
}

ko.run(exampleKoroutine, 0, "myinput1", "myinput2");
```

## Parallel async calls example

You can also fire multiple async calls in parallel using Koroutine and then wait for all of them to complete. To do this, get 
`future` (function) object by calling `this.future()` for each of the async calls and pass that as a callback in place of 
`this.resume`. You can then wait for all of them to complete by calling `yield* koroutine.join(future1, future2, ...)`. It 
returns when all the calls have completed - either succesfully or by returning error. Upon return, each future object either 
has its `data` member set to result - in case of a succesfull call - or its `error` member set to the error returned by the corresponding call.

```js
function* exampleKoroutine(input1, input2) {
    const future1 = this.future();
    dummyAsyncSuccessCall(input1, future1, 1000);
    const future2 = this.future();
    dummyAsyncErrorCall(input2, future2, 1000);
    const numErrors = yield* ko.join(future1, future2);
    console.log(future1.data);
    console.log(future2.error);
}
```
## Koroutine Library Object Methods

### run(generator, timeout, ...rest)
Runs generator function as a coroutine. 

  * __this__ is bound to coroutine context object (see below) inside generator function.  
  * __timeout__ is maximum number of milliseconds coroutine is allowed to run. If it runs for more than that exception is thrown inside generator function with cause = 'timedout'.   
  * __...rest__  are rest of the arguments that are passed in to generator function as function arguments.  

### *join(...futures)
Non-blocking wait till all the async operations represented by futures passed are complete. On completion each future either has `future.data` set to the result of the call (in case of success) or `future.error` set to the error returned by the call.

## Coroutine Context Methods

### this.resume
Callback you can pass to any async calls you want to make from inside the generator function. `this` is bound to current coroutine inside generator function. Resume follows Node js callback convention where first parameter is an error followed by one or more result parameters

### this.future()
Returns a future function object which can be passed as a callback to any async call you wish to make. Futures can be used as callbacks in place of this.resume when you want to make multiple async calls in paralell and then wait for all of them to finish at a single join point inside your code. See [Parallel Async Calls Example](#parallel-async-calls-example) above

### this.sleep(ms)
Non-blocking sleep for `ms` number of milliseconds

### this.defer()
Gives up CPU voluntarily. The coroutine will be resumed on the next event loop turn. Similar to `setImmediate()` or Thread.yield() in pthread library.

### this.cancel()
Cancel the coroutine from outside the the running coroutine. Causes an exception to be thrown inside the canceled coroutine with cause = "canceled"

### cancel
