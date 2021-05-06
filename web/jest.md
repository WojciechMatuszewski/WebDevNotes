# Random `jest` bits

## `mockName` property on a mock

This one is something that is often overlook, but may help you debug failing tests.
While we mock, we often do not think about how your test output will look like if the tests fail. This is a mistake! The more information (useful) information you can provide while your test fails, the better the chances you will fix it in time.

This is where `mockName` on `jest.fn` comes in.

When you create a `spy` you often end up writing something like this:

```js
const spy = jest.fn(); // maybe with mockImplementation
```

While this is completely valid way of using `jest.fn` when the assertion on that spy fails, the output will look similar to this

```js
expect(jest.fn()).not.toHaveBeenCalledWith(...expected);
```

Not that good right? When you have multiple mocks, this can hinder your ability to debug failing test.

You can add `mockName` to the `jest.fn` so that the output will look similar to this

```js
// declaration
const spy = jest.fn().mockName("json");
// failing output
expect(json).not.toHaveBeenCalledWith(...expected);
```

Much better!

## Snapshots

Snapshots **can be useful** but only **when used correctly**. There are many traps you can fall into while using snapshots for your tests, but where snapshots shine are **error messages**.
In such cases snapshots can be really helpful. **Use snapshots for error messages, otherwise avoid them**.

### Snapshots with dynamic values

Imagine this: you are trying to delete an item from a database, and the `id` that your provided does not exist. The error message might contain that `id`. Within tests, you most likely generate that `id` at random, so whenever you try to create an `inlineSnapshot` your tests will fail.

The solution here is pretty simple. Just replace the `id` with something static

```js
const id = "1"; // from the item
const idlessMessage = error.message.replace(id, "SOME_ID");
expect(idlessMessage).toMatchInlineSnapshot();
```

## Mocks on object which only have getters

This usually happens when you have a file which re-exports methods from other files, a barrel module if you will.

```js
export * from "./foo";
```

This is the crux of the issue since this file will be transpiled to an object with only `get` property defined. **Without the `set` property \_jest_s `spyOn` does not work**.
To combat this, mock the entire module, probably using `jest.requireActual` along the way.

```js
jest.mock("module", () => ({
  ...jest.requireActual("module"),
  method: jest.fn(),
}));
```

## Mocking ES6 classes

First of all, remember that the `class` syntax is just a syntactic sugar. In the end, everything lands on the `.prototype`.

Sometimes, you will need to mock a property of a given class. You will most likely not want to explicitly create that class, if you do, the mock will not work (as you are mocking on a different instance than the code you are testing).
What you should do in this situation, is to assign mocks for a given method. You can do it by assigning to a `.prototype` directly.

```ts
SomeClassImport.prototype.someMethod = jest.fn();
```

A good example would be `@aws-sdk/lib-dynamodb` (the v3 version of aws-sdk for Node.js).
The `DocumentClient` is created using the `from` method

```ts
const client = DynamoDBDocumentClient.from(new DynamoDBClient({}));
```

While testing, you will probably need to mock a given method, the `client` exposes.
You can do so, by assigning the mock directly to a given `.prototype` property. You do not have to create instance of `DynamoDBDocumentClient`.

```ts
const mockFn = jest.fn().mockName("sendMock");
DynamoDBDocumentClient.prototype.send = mockFn;
```

## Async timers

A very useful video as a refresher before reading this section
https://www.youtube.com/watch?v=8eHInw9_U8k

So it's 2k20 when writing this and we still are having problems with mocking timers inside async callbacks.

```js
const wait = () => new Promise((resolve) => setTimeout(resolve, 3000));
```

The function above would be easy to test, the whole deal is to make sure you call the `advanceTimersByTime` or `clock.tickAsync` **AFTER** the _promise_ callback has been invoked.

### `useFakeTimers('modern')` hangs my promises

When you are testing asynchronous code which rely on timers, you have probably copied this snipped from StackOverflow

```js
const flushPromises = () => new Promise((r) => setImmediate(r));
```

Here I want to point out that there is nothing wrong with this snipped, but it is misleading.
The callback that you specified within the `new Promise` call is called **synchronously** (the pattern that you see here is called _revealing constructor pattern_). You probably wanted to schedule a microtask which would call `setImmediate` did you?

You probably did not. To properly schedule a callback to be executed in the _microtask queue_ you need to put it inside the `.then` block

```js
const microTask = (doWork) => Promise.resolve().then(doWork);
```

**Or even better, use the `queueMicrotask` API available in almost every browser and in Node.js**

Either way, let's talk about `modern` implementation of `fakeTimers` API

#### Why is my hack pending

If you use the `flushPromises` function along with `jest.useFakeTimers('modern')`, your test function will time out.

```js
// within a test/it block
jest.useFakeTimers("modern");

await flushPromises(); // this will never complete!

// stuff
```

This is because **the `modern` implementation of timers seem to mock ALL the timer-related functions**. This **also includes `setImmediate`**.
So your function will never resolve since you cannot flush the `setImmediate` because you yielded out of the function when you used `await`. What a boomer.

So what do you do?

1. You could switch to `legacy` implementation of timers, but they might not be supported anymore at the time you are reading this

2. Use methods exposed by the `timers` package

The method no. 1 is self explanatory, so let us focus on the no. 2

#### The `timers` package

The `timers` package contains all the timer-related functions that are available to you globally.
In our context, that package is of significance because **jest does not mock timers exposed by that package, jest ony mocks timers that live on the global object**.

So in our case, we can import the non-mocked version of `setImmediate` which will actually be called (since it's not mocked).
We can then `runAllTimers` within the callback of the non-mocked `setImmediate`

#### The solution

So bringing all the information you've read so far together. The solution to the issue

```js
import { setImmediate } from "timers";

// within a test/it block
setImmediate(() => jest.runAllTimers()); // actually runs, uses the non-mocked version of the `setImmediate` API

// stuff
```

There you have it. How to stay on the more feature-rich implementation of timers and still be able to test async code that relies on timers.

### MSW delay timers

But what if you have more complex example, like a webserver with a delay (you have to use the native `http` module because `msw` does not support timer mocks - it uses the `timer` module, LOL!)

```js
const { createServer } = require("http");
const server = createServer(async (_, res) => {
  await new Promise((resolve) => {
    setTimeout(() => {
      resolve();
    }, 400);
  });

  res.write(JSON.stringify({ message: "success" }));
  res.end();
});
```

So with this example, I was not able to make sure the functions that run timers run after the promise callback. You might overcome this using the _legacy_ timers from jest and the `setImmediate` trick.

This is quite bad.

#### `msw` workaround

So we learned that the `msw` package is using the `timers` package so that the `setTimeout` used to make `ctx.delay` possible is not mocked.
But what if you really want to make it work? Well, just mock the `timers` package yourself.

```js
jest.mock("timers", () => ({
  setTimeout: setTimeout,
}));
```

This will make it so that the `timers.setTimeout` actually points to the global one. This will allow `sinon` or `jest` to mock those.

```js
const FakeTimers = require("@sinonjs/fake-timers");
const clock = FakeTimers.install();

jest.mock("timers", () => ({
  setTimeout: setTimeout,
}));
```

And now you can use `await clock.tickAsync` in your tests. This is **actually, IMO, the best solution to the problem described above**.
One thing that is quite bad about this whole situation is that we are relying on implementation details. If the author changes the implementation of `ctx.delay` we are doomed.

Nevertheless having some test is better than having no test at all.

### Flushing without timers

So let us say you have a situation where you are invoking something in a _fire-and-forget_ manner.

```js
function getData() {
  void callService(URL).catch(console.log);
}
```

How would you test this while also making sure there are no leaks in your tests? (the `callService` promise sticking around after your test case finished)

The solution here is to make sure that the `callService` promise is _flushed_ before making any assertions. Since we do not have access to the promise itself (it's not returned from the `getData` function) we need to leverage our knowledge of the event loop to get the job done.

#### The `setImmediate` way

```js
test("it does not throw even if the `callService` rejects", async () => {
  getDataMock.mockRejectedValueOnce(new Error("boom"));

  const result = getData();

  await new Promise((r) => setImmediate(r));

  // if the `getData` throws, the test would not pass
  expect(result).toEqual(undefined);
});
```

The `microtask` and `nextTick` functions are processed in-between every event loop phase. What happens here:

1. The callback for the promise inside the `getData` is put on the `microtask` queue

2. The callback for the inline promise is put on the `microtask` queue

3. The control is yielded back to the event loop

4. First `microtask` callback is fired. This will be our `catch`. The error is logged

5. Second `microtask` callback is fired. This will be our inline `setImmediate` call. The `immediate` queue is populated

6. The event loop moves forwards to the `immediate` phase. The inline inline promise is resolved

7. The control is yielded back to the test function

8. Assertion is made, we are 100% certain that it was made after the `getData` resolved

#### The `Promise.resolve` way

I would not recommend this method. This is a mental shortcut which is indirect and can create confusion. Having said that I think that it is worth mentioning

```js
test("it does not throw even if the `callService` rejects", async () => {
  getDataMock.mockRejectedValueOnce(new Error("boom"));

  const result = getData();

  await Promise.resolve();

  // if the `getData` throws, the test would not pass
  expect(result).toEqual(undefined);
});
```

1. The callback for the promise inside the `getData` is put on the `microtask` queue

2. The control is yielded back to the event loop

3. First `microtask` callback is fired. This will be our `catch`. The error is logged

4. The control is yielded back to the test function

5. Assertion is made

As you can see we are not using the `setImmediate` semantics to ensure that the assertion is made after the `getData` promise is resolved. While there seem to be _less noise_ in the code itself, the reader of the code does not have a point of reference - that being `setImmediate` and it's documentation in the previous example

### Flushing with timers

You should favour the _modern_ implementation of timers moving forward. With modern timers you can manipulate the `Date` object and other, previously unreachable timers.

I'm going to be using only the `setImmediate` way of flushing timers. This is because of the reasons I mentioned above.

So the situation here is that we have a promise which is resolved when some timer ticks to 0. An example

```js
test("waiting", async () => {
  jest.useFakeTimers("modern");

  const waiter = new Promise((r) => {
    setTimeout(() => r("DATA"), 100000);
  });

  jest.runAllTimers();

  await expect(waiter).resolves.toEqual("DATA");
});
```

As you can see, there is not much to it. All you have to do is to let the callbacks be allocated to proper queues in the event loop phases and then call the `jest` timers API.

The `Promise` callback it put inside the `microtask` queue which is checked before moving to a new _phase_. Inside that callback we are pushing a new callback into the _timers_ queue.
