# Task

Like a Promise with better error handling and some other upgrades.

> made (and best used) with [TypeScript](http://www.typescriptlang.org/) ♥️

### Usage
**Install** the library in your project.
```bash
npm install @ts-task/task
```

**Use** it in your code preeeeety much how you would use a Promise.

```typescript
import { Task } from '@ts-task/task';

// Create it with a resolver
const task1 = new Task((resolve, reject) => {
    setTimeout(
        _ => resolve('Hello'),
        2000
    )
})

// Or with a constructor
const task2 = Task.resolve('world');

Task.all([task1, task2])
    // Transform the eventual value
    .map(([msg1, msg2]) => `${msg1} ${msg2}!!!`)
    // And then do something with it
    .fork(
        err => console.error('Buu', err), // Errors come first!
        msg => console.log('Yeay', msg)
    );
```

### Why

Promises are great! so why do we need a replacement? or when should I use `Task`?

Good question, I'm happy you asked. Both promises and tasks are great if you are working with *single asynchronous values*, meaning **one** *value or event* that maybe ocurring now or in some future. For example getting results from a database query, or making an AJAX request to a server. If you need *multiple asynchronous values* you should check out [RxJs](https://github.com/ReactiveX/RxJS).

The differences between Task and Promise (or why should you choose task) are:

* Task have [better error handling](#better-error-handling), so you'll have less bugs
* Task are [`pipe`able](#pipe-operator), so they are easier to extend
* Task has more [specific semantics](#specific-semantics), so it will be easier to know what you are doing
* Task are Lazy, so it's easier to create retry logic

### Better Error Handling
If you ever used Promises in TypeScript you may have noticed that it's only typed on success a.k.a `Promise<T>`. For example in the following code

```typescript
const somePromise: Promise<string>;

somePromise
  // Transform the eventual value
  .then(x => `${x}!!!`)
  // And then do something with it
  .then(
    value => /* value is of type string */,
    err   => /* err is of type any */
)
```

We know `value` is of the expected type `T` but we don't know **any**thing about `err`. The main reason is that when we transform our promises, the callbacks we pass to `then` can throw **any**thing, and in TypeScript the exceptions are not typed. We could manually define the error type and say it's a `Promise<string, Error>` but it would be a lie because we can't avoid exceptions and if they happen we can't know their types, and because `any & Error = any` we can't be more specific.

Thats a boomer because we make all this trouble with static typings to have more confidence on how we program and we are left wide open when things go south.


So we can't forbid a function from throwing but we can wrap exceptions inside an `UncaughtError` object, and with that decision alone we can type `Task<T, E>` and let TypeScript help us infer and manipulate errors 🎉.

For example

```typescript
const task1 = Task.resolve(1);
// task1 is of type Task<number, never>, which makes sense as there is no way resolve can fail

const task2 = task1.map(n => '' + n)
// task2 is of type Task<string, UncaughtError>, because we have converted the success value
// and we don't know if the inner function throws an error or not
```

You can also add, remove and transform your error logic and the type inference will get you a long way.

For example is you use the `caseError` function from [@ts-task/utils](https://github.com/ts-task/utils) you could do something like this

```typescript
// Assuming we have defined a getUser function somewhere
declare function getUser(id: number): Task<User, DbError | UserNotFound>;

const user = getUser(100)
  .catch(
    caseError(
      // If the error meets this condition
      isInstanceOf(UserNotFound),
      // Handle it with this callback
      err => Task.resolve(new Guest())
    )
  )
// user will have type Task<User | Guest, DbError | UncaughtError> because caseError only handles
// the UserNotFound error (removing it from the errors) and resolves it to a new type of answer (Guest)
// and theres always the possibility one of those functions throws, so we have to take UncaughtError
// in account
```

Another use case could be to only retry an http request if the error was 502, or 504 which may happen on a timely basis but don't retry if the error was 401 or a rate limit as the expected result is the same.

Another feature of Task that will help you with error handling is that when you fork the error callback comes first. So whenever you want to use the eventual value, you first need to decide what you do with the error. You can always ignore it or `console.log` it, but you need to make a conscious decision.

```typescript
Task
  .resolve('Hello!')
  .fork(
      err => console.error('Buu', err), // Errors come first!
      msg => console.log('Yeay', msg)
  );
```

This is great for defensive programming 🛡 as all possible paths will need to be defined.

> If you only type on success you're missing half the fun

### Pipe operator
When the [pipe operator](https://github.com/tc39/proposal-pipeline-operator) lands to JavaScript (currently in stage 1) we will be able to write code like this

```javascript
const task = Task.resolve(1)
    |> map(n => '' + n)
    |> chain(getUser)
    |> retryWhen(DbError)
    |> catchError(caseError(UserNotFound, err => Task.resolve(null)))
```

which has the advantage of using custom methods without having to modify the prototype of `Task`. But because we cannot
wait until the operator makes it to the standard we added a `pipe` method to Task, inspired by [RxJs pipeable operators](https://github.com/ReactiveX/rxjs/blob/master/doc/pipeable-operators.md).

So the previous code would look like

```javascript
const task = Task.resolve(1).pipe(
      map(n => '' + n)
    , chain(getUser)
    , retryWhen(DbError)
    , catchError(caseError(UserNotFound, err => Task.resolve(null)))
)
```

which is not that bad. All that is required is that the functions passed pipe to have the following signature
`Task<T1, E1> => Task<T2, E2>`

We have a list of experimental operators in `src/operators/experimental.ts`. Use them with discresion as they might change without notice. All other operators will follow the semantic release convension.

### Specific semantics

Promises API is quite simple by design, it has a `then` method that can be used for 3 different purposes, in contrast *Task* has a different method for each usage.

* `Promise.then` can be used to **transform** an eventual value, with *task* you should use `map`.
* `Promise.then` can be used to **chain** sequential async operations, with *task* you should use `chain`.
* `Promise.then` can be used to **do** something once you have the result, with *task* you should use `fork`.

As stated in *Lord of the Promises*
> One `method` to rule them all, One `method` to find them,
> One `method` to bring them all and in the darkness **bind** them

The nice thing about having one method should be simplicity (less methods to remember), but trying to put the different use cases in the same method can cause some confusions that we'll explain in this section.

For example, when we want to **do** something with an eventual value, we need to know if the Promise *succeeds* or *fails*. Thats why the `then` method accepts two arguments, the *onSuccess* and *onError* callbacks (in that order).

If it's used in the middle of a Promise chain, it can cause some confusion

```javascript
// Don't do this
somePromise
  .then(x => foo(x))
  .then(
      y   => bar(y),
      err => handleError(err)
    )
  .then(z => baz(z))
```

a common doubt arises with `handleError`, does it catch errors on *foo* or in *bar*?. The answer is the first option, thats why it's recommended to write an explicit `catch` instead.

```javascript
// Instead do this
somePromise
  .then(x => foo(x))
  .catch(err => handleError(err))
  .then(y => bar(y))
  .then(z => baz(z))
```

But just the fact that you can write the previous code can be misleading.

The second argument of a `then` method should only be used in the last step of a Promise chain, when we are **do**ing something with the result.

```javascript
somePromise
  .then(...)
  .then(...)
  .then(
    html => render(html),
    err => openErrorModal(err)
  );
```

But because the second parameter is optional, it's fairly easy to end up with an unexpected result. For example in the following code

```javascript
somePromise
  .then(...)
  .then(...)
  .then(
    html => render(html)
  );
```

if `somePromise` fails there is no handler, so depending on the environment you could get a silent error or an `Uncaught Promise Rejection` that may be difficult to trace.

When using task, if you want to **do** something with your eventual result you have to use `fork` as the last step. That method is the only one that doesn't return a new *task*, so it's impossible to use it in the middle of the chain. Even more, `fork` handles errors in the first callback, so it's impossible to have an `Uncaught Promise Rejection`.

```javascript
someTask
  .map(...)
  .chain(...)
  .fork(
      err => openErrorModal(err),
      html => render(html)
  )
```

And because *Tasks* are lazy, if you don't call `fork` nothing happens, so the library forces you to use best practices.

The difference between `map` and `chain` is a little more subtle. Promise (and Tasks) **transform**ations are useful when you don't care about the eventual value itself, rather something that can be synchronously computed from that value. For example, you could fetch a document and only care about how many words the document has.

```javascript
fetch('http://task-manifesto.org')
  .then(doc => countWords(doc))
  .then(n => alert(`The document has ${n} words`));

fetchTask('http://task-manifesto.org')
  .map(doc => countWords(doc))
  .fork(
    noop,
    n => alert(`The document has ${n} words`)
  );
```

where we assume `countWords` is a function that receives a `String` and returns a `Number`.

In contrast, **chain**ing *Promises* and *Tasks* are useful when you need the result of a previous async operation in order to make the next one. For example the first request may return a JSON object with an url to make the next request.

```javascript
fetch('http://some-rest.api')
  .then(obj => analyzeResponseAndGetTheUrl(obj))
  .then(url => fetch(url));
  .then(obj => alert(`Some data ${obj.foo}`));

fetchTask('http://some-rest.api')
  .map(obj => analyzeResponseAndGetTheUrl(obj))
  .chain(url => fetchTask(url));
  .fork(
    noop,
    obj => alert(`Some data ${obj.foo}`)
  );

```

In this example we are synchronously **transform**ing the first response using the `analyzeResponseAndGetTheUrl` function that receives an `Object` and returns a `String` and then **chain**ing the eventual transformation with the second call to `fetch`.

This should give you an intuition that whenever you see a `map` you are not adding more time to your computation, but when you are using `chain` you are most likely are.


## Credits

Initialized with [@alexjoverm](https://twitter.com/alexjoverm)'s [TypeScript Library Starter](https://github.com/alexjoverm/typescript-library-starter) under the Acamica Labs initiative and made with :heart: by

<!-- ALL-CONTRIBUTORS-LIST:START - Do not remove or modify this section -->
<!-- prettier-ignore -->
| [<img src="https://avatars0.githubusercontent.com/u/2634059?v=4" width="100px;"/><br /><sub><b>Hernan Rajchert</b></sub>](https://github.com/hrajchert)<br />[💻](https://github.com/hrajchert/@ts-task/task/commits?author=hrajchert "Code") [🎨](#design-hrajchert "Design") [📖](https://github.com/hrajchert/@ts-task/task/commits?author=hrajchert "Documentation") [💡](#example-hrajchert "Examples") [⚠️](https://github.com/hrajchert/@ts-task/task/commits?author=hrajchert "Tests") | [<img src="https://avatars1.githubusercontent.com/u/1573956?v=4" width="100px;"/><br /><sub><b>Gonzalo Gluzman</b></sub>](https://github.com/dggluz)<br />[💻](https://github.com/hrajchert/@ts-task/task/commits?author=dggluz "Code") [🤔](#ideas-dggluz "Ideas, Planning, & Feedback") [⚠️](https://github.com/hrajchert/@ts-task/task/commits?author=dggluz "Tests") |
| :---: | :---: |
<!-- ALL-CONTRIBUTORS-LIST:END -->

This project follows the [all-contributors](https://github.com/kentcdodds/all-contributors) specification. Contributions of any kind welcome!
