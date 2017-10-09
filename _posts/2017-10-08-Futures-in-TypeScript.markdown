---
layout: post
title:  "Futures in TypeScript"
date:   2017-10-08 00:00:00 +0000
categories: typescript functional
---

# Futures in TypeScript

In functional programming Promises are not the norm for async communications. The datatype **Future** is far more wide spread. This will be my try to implement a basic Future in TypeScript.

## What Differs Futures from Promises

In all honesty not very much differs from promises. The main thing is that a Promise is evaluated when it is created while a Future is evaluated when you explicitly call `fork`. Further they adhere to the monadic interface, something that Promises does not do. 

{% highlight ts %}

new Promise((res) => { setTimeout(1000, res, 'Bananas') })
.then(x => `${x} are great!`)
.then(x => FunctionReturningPromise(x)))
.then(console.log, console.error);

{% endhighlight %}

This is what it would look like using Futures

{% highlight ts %}

Future((rej, res) => { setTimeout(1000, res, 'Bananas') })
.map(x => `${x} are great!`)
.chain(x => FunctionReturningFuture(x))
.fork(console.error, console.log)

{% endhighlight %}

The promise version uses `then` for everything but the Future version have three distinct functions.
* Map - Converts the data from one form to another.
* Chain - This takes a future and flattens it, absorbing it into the Future chain. Just like `flatMap` the Promise version also does this but it is far from obvious and is handeled automatically.
* Fork - Explicitly evaluate the future.

A Future cannot be evaluated if the continuations (the functions provided to `fork`) are not provided. A Promise will run even if no continuations exists. Also when a Future is created you have the option to call `fork` as many times as you like. Every new fork will result in another evaluation. This is not possible using Promises.

## A Simple Example

Let's get started! 

{% highlight ts %}

const simpleFuture = f =>  ({
        fork: g => f(g)})

{% endhighlight %}

This creates a higher order function that takes the function `f` as a parameter returning a new `simpleFuture` with a `fork` function on it. Calling the fork function with a handler will result in the future being evaluated. This requires that the function `f` is a function that takes one callback parameter (hard to know without types). Something like this:

{% highlight ts %}

const simpleGetHttp = callback => {
    const rnd = Math.floor(Math.random() * 5000 + 1000)
    setTimeout(() => callback(`hello ${rnd}`), rnd) 
}

const myFuture = simpleFuture(simpleGetHttp)
myFuture.fork(console.log)
//=> hello 4745

{% endhighlight %}

Wow great! What is happening. `simpleGetHttp` is called and `console.log` is passed as the `res` parameter. When the timeout is finished `console.log` will be called with the result. Pretty much what we want.  

### Adding Types

This is typescript after all. Let's add some types. 

{% highlight ts %}

interface ISimpleFuture<T> {
    fork<T>(res:(T)=> void) : void
}

const simpleFuture = <T>(f:(res:(T) => void) => void) : ISimpleFuture<T> =>  ({
        fork: g => f(g)
    })

const simpleGetHttp = (callback:(string) => void) => {
    const rnd = Math.floor(Math.random() * 5000 + 1000)
    setTimeout(() => callback(`hello ${rnd}`), rnd) 
}

{% endhighlight %}

### Adding Map

This is all great but we need a map function too (in reality we need much more but let's start with map)

Fist the interface

{% highlight ts %}

interface ISimpleFuture<T> {
    fork<T>(res:(T)=> void) : void
    map<U>(a:(T) => U) : ISimpleFuture<U>
}

{% endhighlight %}

map will be a generic function taking function parameter that takes a value of type `T` and mapping that to a value of type `U`. Then this value will be wrapped in a new `ISimpleFuture` of type `U`. What would the implementation for this interface look like?

{% highlight ts %}

const simpleFuture = <T>(f:(res:(T) => void) => void) : ISimpleFuture<T> =>  ({
        fork: g => f(g),
        map: g => simpleFuture(k => f(l => k(g(l))))
    })

{% endhighlight %}

That is a bit more complicated. For starters  is a function converting a value from `T` to `U` (parsing a string to a number for example). What is returned is a new `simpleFuture` with a whole lot of wrapping going on. Let's take it for a spin

{% highlight ts %}

const myFuture : ISimpleFuture<string> = simpleFuture(getHttp)
const myMappedFuture : ISimpleFuture<number> = myFuture.map<number>(x => x.match(/\d+/g)[0])
myMappedFuture.fork(console.log)
//=> 5237

{% endhighlight %}

That works. Sheer luck with all that wrapping. To complete the monadic interface a couple of other functions such as `chain` and `apply` but the basics are done.

## With Both Reject and Resolve

Let's try to do a version that supports both `resolve` and `reject`. This is a bit more complicated but not much

let's start with the interface. And for convenience sake will do `of` and `reject`. `of` creates and resolves a Future in one go while `reject` creates and rejects it.

{% highlight ts %}

interface IFuture<T, U> {
    of(U): IFuture<any, U>;
    reject(T) : IFuture<T, any>;
    fork<T, U>(a:(T) => void, b:(U) => void) : void;
}

{% endhighlight %}

Since we have both a reject and a resolve function the generic interface needs two type variables. The `reject` handler can be passed an `error` object while the `resolve` can be passed a string.

This is a sample implementation

{% highlight ts %}

const future = <T, U>(f:(rej:(T) => void, res:(U) => void) => void): IFuture<T, U> => ({
        fork: (rej, res) => f(rej, res),
        of: x => future((_, res) => res(x)),
        reject: x => future((rej, _) => rej(x)),
    })

{% endhighlight %}

First the function expected to be passed to the constructor now takes two arguments one function that takes a `T`, the reject type and a function that takes `U` the resolve type. In essence the whole thing works as before just using two variables.

The `of` function takes a `U` and  returns a new future that calls the `resolve` function with that value. `reject` works the same way but calls the reject function. As simple as that.

Let's test it.

{% highlight ts %}

const getHttp = (rej:(string) => void, res: (string) => void) => {
    const rnd = Math.floor(Math.random() * 1000 + 1000)
    setTimeout(() => (rnd % 2 == 0) ? 
                        rej(`Failed after  ${rnd} ms`) : 
                        res(`Succeeded after ${rnd} ms`) , rnd) 
}

const withreject : IFuture<string, string> = future2(getHttp)
withreject.fork(x => console.error('error: ' + x), x => console.log('success: ' + x))
//=> error: Failed after 1345 ms
// or
//=> success: Succeeded after 1122 ms

{% endhighlight %}

### Adding map

This one needs a `map` function as well.

{% highlight ts %}

interface IFuture<T, U> {
   of(U): IFuture<any, U>;
   reject(T) : IFuture<T, any>;
   map<V>(a:(U) => V): IFuture<T, V>;
   fork<T, U>(a:(T) => void, b:(U) => void) : void;
}

{% endhighlight %}

this leads to the implementation

{% highlight ts %}

const future = <T, U>(f:(a:(T) => void, b:(U) => void) => void): IFuture<T, U> => ({
        fork: (rej, res) => f(rej, res),
        of: x => future((_, res) => res(x)),
        reject: x => future((rej, _) => rej(x)),
        map: <V>(g) : IFuture<T, V> => future((rej, res) => f(a => rej(a), b => res(g(b))))
    })

{% endhighlight %}

If you managed to understand the simple version this will pretty much be the same. The only thing worth mentioning is that `map` will only map success values. Better implementations of Future will have a special `bimap` function that can map both rejections and resolved values.

Does our `of` and `reject` functions work. Note the dummy function that is passed. It is somethig like a Identity function. Makes this simplistic implementation work

{% highlight ts %}

const resolved = future(x => x).of('Hello').map(x => x + ' world!')
resolved.fork(console.log, console.log)
//=> Hello world
const rejected = future(x => x).reject('Goodbye').map(x => x + ' cruel world!')
rejected.fork(console.log, console.log)
//=> Goodbye

{% endhighlight %}

## Conclussion

There is no mystery behind the Future data type. It is pretty much the same thing as a Promise. The upside is that it is lazy and I at least thinks it is easier to work with with sane names for the methods and less magic then promises. 

Whatever you do do not use any of this code instead take a look at the following libraries.

[Ramda-Fantasy](https://github.com/ramda/ramda-fantasy)
[Fluture](https://github.com/fluture-js/Fluture)
[FolkTale](https://github.com/origamitower/folktale)
