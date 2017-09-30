---
layout: post
title:  "Fun With Catamorphisms"
date:   2017-09-30 00:00:00 +0000
categories: typescript functional
---

The word **cata** (ancient Greek: κατά _“down from”_) + **morphism** (ancient Greek: μορφή _“form, shape”_) is used to describe a way to collapse a recursive structure into a new value based on its structure. `fold` defined for a list gives you a hint of what a catamorphisms is able to do. It is a very powerful thing to define for any structure since all other functions can be defined in terms of it. The closes equivalent in Object Oriented programming is perhaps the [visitor pattern](https://en.wikipedia.org/wiki/Visitor_pattern).

To illustrate a basic catamorphism we have to define a recursive datastructure. This one is basically shamelessly stolen from [fsharpforfunandprofit](https://fsharpforfunandprofit.com/posts/recursive-types-and-folds) (a great resource for every one interested in functional programming).

## The Gift data structure

{% highlight ts %}
interface Book {
  kind: "book";
  price: number;
  title: string;
}

interface Chocolate {
  kind: "chocolate";
  taste: string;
  price: number;
}

interface Wrapping {
  kind: "wrapping";
  pattern: string;
}

interface Wrapped {
  kind: "wrapped";
  wrapping: Wrapping;
  contains: Gift;
}

interface Boxed {
  kind: "boxed";
  contains: Gift;
}

type Gift = Book | Chocolate | Wrapped | Boxed;
{% endhighlight %}

We are using `tagged unions` to describe the datatype `Gift`. A gift can either be a `Book` or a `Chocolate`. Where things get a bit more complicated is we add the types `Boxed` and `Wrapped`. A boxed book is still a gift and a wrapped, boxed book still serve as a great birthday present. One flaw here is that an empty box is still considered a gift but that’s OK for this purpose.

## The Test Data

{% highlight ts %}
let book1: Gift = {
  kind: "book",
  title: "The Life of a Lumberjack",
  price: 123
}; 

let chocolate1: Gift = {
  kind: "chocolate",
  price: 22,
  taste: "Strawberry"
}; 

let wrapping1: Wrapping = { kind: "wrapping", pattern: "diamonds" };

// Data constructor for wrapped gifts
const wrapGift = (g: Gift, w: Wrapping): Gift => ({
  kind: "wrapped",
  wrapping: w,
  contains: g
});

// Data constructor for boxed gifts
const boxGift = (g: Gift): Gift => ({ kind: "boxed", contains: g });

let wrapped1: Gift = wrapGift(book1, wrapping1); 
let boxed1: Gift = boxGift(chocolate1); 
let wrapped2: Gift = wrapGift(boxed1, wrapping1); 
{% endhighlight %}

`boxGift` is a function that takes a gift and “puts it in a box”. `wrapGift` takes a gift and a wrapping and returns the newly wrapped gift. This leaves us with a couple of varying gifts.

## Life Without Catamorphisms

The first thing we want to do is create a function that takes a gift and tells us whats inside. A form of pretty print for the gift data type. 

{% highlight ts %}
  const whatsInside = (g: Gift): string => {
    switch (g.kind) {
      case "chocolate":
        return `Delicious ${g.taste} chocolate`;
      case "book":
        return `An interesting book named ${g.title}`;
      case "wrapped":
        return whatsInside(g.contains) + ` wrapped in ${g.wrapping.pattern}`;
      case "boxed":
        return whatsInside(g.contains) + " boxed";
    }
  };  
{% endhighlight %}

Here we use the `tagged union` to switch on the kind. In case of the both leave-nodes (chocolate and book) we just return a string that informs us of the taste or title.
The containers (wrapping and boxing) adds the style of paper or just the information that the gift is inside a box. Then it recursively calls `whatsInside` to keep on moving to the center of the gift.

{% highlight ts %}
console.log(whatsInside(book1));
//=> An interesting book named The Life of a Lumberjack​​​​​
console.log(whatsInside(chocolate1));
//=> ​​​​​Delicious Strawberry chocolate​​​​​
console.log(whatsInside(wrapped1));
//=>​ ​​​​​An interesting book named The Life of a Lumberjack wrapped in diamonds​​​​​
console.log(whatsInside(wrapped2));
//=> ​​​​​Delicious Strawberry chocolate boxed wrapped in diamonds​​​​​
console.log(whatsInside(boxed1));
//=> ​​​​​Delicious Strawberry chocolate boxed​​​​​
{% endhighlight %}

Seems to be working great! Lets do another one! This time we want to know the complete cost of the gift. This will take the price of the gift and add the price for the box or the wrapping paper. There are not defined in the data structure and is added in this function instead.

{% highlight ts %}
const totalCost = (g: Gift): number => {
  switch (g.kind) {
    case "chocolate":
      return g.price;
    case "book":
      return g.price;
    case "wrapped":
      return 1 + totalCost(g.contains);
    case "boxed":
      return 5 + totalCost(g.contains);
  }
};
{% endhighlight %}

Works the same way as `whatsInside` the difference is the return value. This returns a `number` while `whatsInside` returns a string. 

{% highlight ts %}
console.log(`$ ${totalCost(book1)}`);
//=> $ 123
console.log(`$ ${totalCost(chocolate1)}`);
//=> $ 22
console.log(`$ ${totalCost(wrapped1)}`);
//=> $ 124
console.log(`$ ${totalCost(wrapped2)}`);
//=> $ 28
console.log(`$ ${totalCost(boxed1)}`);
//=> $ 27
{% endhighlight %}

And it seems like it´s adding correctly. Great! But I get this nagging feeling that since the two functions are pretty much the same there is some form of reuse we are missing. 

## Here Comes the Catamorphisms

This is where the catamorphisms enter the stage. A catamorphisms is a general way of “collapsing” our gift data structure or generally any recursive data structure. A catamorphism does this by recursion from the bottom up. There are other ways of doing this collapsing but this is a nice and easy way to do it. 

How do we create a catamorphism then? There are just a few steps. If we follow them we will be fine.

Step one is to create a function that that takes four functions. One for `Book`, `Chocolate`, `Box` and `Wrapped` plus the `Gift` we want to traverse. 

{% highlight ts %} 
const cataGift = <T>(
  fBook: (a: Book) => T,
  fChocolate: (a: Chocolate) => T,
  fWrapped: (a: T, w: Wrapping) => T,
  fBoxed: (a: T) => T,
  g: Gift
): T 
{% endhighlight %}

This is our function signature. `cataGift` is generic and parameterized by the type `T`. The first parameter called fBook is the function that will handle book. It takes a function that takes a `Book` and returns a `T`. `fChocolate` behaves the exact same way.

`fWrapped` is a bit more complicated. It takes a function that takes a `T` and a `Wrapping` and then returns a `T`. This is because the catamorphism will recurse when it encounters a wrapping and the result from the recursion will be `a`

Finally our function will take a `Gift` and return a `T`. The complete function will look something like this.

{% highlight ts %}
const cataGift = <T>(
  fBook: (a: Book) => T,
  fChocolate: (a: Chocolate) => T,
  fWrapped: (a: T, p: Wrapping) => T,
  fBoxed: (a: T) => T,
  g: Gift
): T => {
  switch (g.kind) {
    case "chocolate":
      return fChocolate(g);
    case "book":
      return fBook(g);
    case "wrapped":
      return fWrapped(
        cataGift(fBook, fChocolate, fWrapped, fBoxed, g.contains),
        g.wrapping);
    case "boxed":
      return fBoxed(cataGift(fBook, fChocolate, fWrapped, fBoxed, g.contains));
  }
};
{% endhighlight %}

If the gift it `Chocolate` or `Book` we will call the provided function on that value. Pretty straight forward. If the `Gift` is  wrapped we will call the cataGift on the contained gift recursevly. When that finally returns we well call `fWrapped` with the result from the recursion and also the wrapping. A boxed gift will just recurse and then call `fBoxed` with the result.

## whatsInside with cataGift

Lets reimplement `whatsInsde`using our new fancy catamorphism.

{% highlight ts %}
const whatsInsideC = (g: Gift): string =>
  cataGift(
    b => `An interesting book named ${b.title}`,
    b => `Delicious ${b.taste} chocolate`,
    (c, p) => c + ` wrapped in ${p.pattern}...`,
    b => b + " boxed" + b,
    g);
{% endhighlight %}

we are using lambdas here and I hope they are self explanatory. This is the result:

{% highlight ts %}
console.log(whatsInsideC(book1));
//=> An interesting book named The Life of a Lumberjack​​​​​
console.log(whatsInsideC(chocolate1));
//=> ​​​​​Delicious Strawberry chocolate​​​​​
console.log(whatsInsideC(wrapped1));
//=>​ ​​​​​An interesting book named The Life of a Lumberjack wrapped in diamonds​​​​​
console.log(whatsInsideC(wrapped2));
//=> ​​​​​Delicious Strawberry chocolate boxed wrapped in diamonds​​​​​
console.log(whatsInsideC(boxed1));
//=> ​​​​​Delicious Strawberry chocolate boxed​​​​​
{% endhighlight %}

Seems alright!

## totalCost with cataGift

How about `totalCost` how do we recreate that using `cataGift`. In case of `Book` and `Chocolate` just return the price. In the recursive cases, add our cost to the recursive case and return that. Something like this.

{% highlight ts %}
const totalCostC = (g: Gift): number =>
  cataGift(
    b => b.price,
    b => b.price,
    (c, p) => c + 1,
    b => b + 5,
    g);
{% endhighlight %}

and that will yield

{% highlight ts %}
console.log(`$ ${totalCostC(book1)}`);
//=> $ 123
console.log(`$ ${totalCostC(chocolate1)}`);
//=> $ 22
console.log(`$ ${totalCostC(wrapped1)}`);
//=> $ 124
console.log(`$ ${totalCostC(wrapped2)}`);
//=> $ 28
console.log(`$ ${totalCostC(boxed1)}`);
//=> $ 27
{% endhighlight %} 

Super! It all works out!

## Unwrapper

One nice thing with catamorphisms is how easy it is to convert from one structure to another. If we want to remove the packaging just use `cataGift`!

{% highlight ts %}
// I'm well aware that this is not the Identity. But close enough. 
// It takes any number of parameters and returns the first one. In our case it will work
const Id = (x, ...y) => x;

const unwrapper = (gift: Gift): Gift =>
    cataGift(
      Id, // do nothing
      Id,
      Id, // This will throw away the wrapping
      boxGift, // this will reboxThe gift
      gift
    );   
{% endhighlight %}

Ok. I created a function called `Id`. It's not really the Identity but it will work in our case. It just takes any number of parameters and returns the first one. That makes it work both with our recursive cases and our simple cases.

Our function takes a `Gift` and returns a `Gift`. The simple cases just returns the same `Book` or `Chocolate`. The wrapped case will just return the fist parameter. in our case that is the contents of the wrapped datatype. Great! That will remove the wrapping! When we hit our boxed case we cant easily just return the same box. If we check our function signature `fBoxed: (a: T) => T` it takes a `T` and returns a `T`. In our case the `T` is a `Gift` and the gift in question is the contents. So we need to re-box our content. Luckily we already have a function that does exactly that. `boxGift` our data constructor for boxes. Just use that

The result:

{% highlight ts %}
console.log(unwrapper(wrapped1));
//=> ​​​​​{ kind: 'book', title: 'The Life of a Lumberjack', price: 123 }​​​​​
console.log(unwrapper(wrapped2));
//=> { kind: 'boxed',​​​​​
//   ​​​​​  contains: { kind: 'chocolate', price: 22, taste: 'Strawberry' } }​​​​​
console.log(unwrapper(chocolate1));
//=> ​​​​​{ kind: 'chocolate', price: 22, taste: 'Strawberry' }​​​​​
console.log(unwrapper(boxed1));
//=> ​​​​​{ kind: 'boxed',​​​​​
​​​​​//    contains: { kind: 'chocolate', price: 22, taste: 'Strawberry' } }​​​​​
{% endhighlight %}

Seems ok. And its easy to do an unboxer too

{% highlight ts %}
const unboxer = (gift: Gift): Gift =>
    cataGift(
      Id,
      Id,
      wrapGift, // rewrap
      Id,
      gift);
{% endhighlight %}

and a function that tastes all chocolate? Is that possible? Well yes!

{% highlight ts %}
const nibble = c => ({
    kind: "chocolate",
    taste: "half eaten " + c.taste,
    price: c.price / 2
  });

const nibbler = (gift: Gift): Gift =>
    cataGift(Id, nibble, wrapGift, boxGift, gift); 

console.log(nibbler(wrapped2));
//=> ​​​​​{ kind: 'wrapped',​​​​​
​​​​​//  wrapping: { kind: 'wrapping', pattern: 'diamonds' },​​​​​
​​​​​//  contains: ​​​​​
​​​​​//   { kind: 'boxed',​​​​​
​​​​​//     contains: { kind: 'chocolate', taste: 'half eaten Strawberry', price: 11 } } }​​​​​
{% endhighlight %}

Great! And look! They compose

{% highlight ts %}
nibbler(unboxer(unwrapper(wrapped1))); 
nibbler(unboxer(unwrapper(wrapped2))); 
unboxer(unwrapper(chocolate1)); 
unboxer(unwrapper(boxed1)); 
{% endhighlight %}

In any order!

{% highlight ts %}  
nibbler(unboxer(unwrapper(wrapped2))); 
unboxer(nibbler(unwrapper(wrapped2))); 
unboxer(unwrapper(nibbler(wrapped2))); 
{% endhighlight %} 

Lets make a list of all gifts
{% highlight ts %}
let gifts: [Gift] = [book1, chocolate1, wrapped1, boxed1, wrapped2]; /*?*/
{% endhighlight %}

Then we can sum the total costs of all the gifts using a ordinary reduce
{% highlight ts %}  
gifts.reduce(
    (a, x) => a + cataGift(b => b.price, b => b.price, Id, Id, x),
    0); 
{% endhighlight %}

And it is easy to create a cata that lists all the contents, discarding andy packageing

{% highlight ts %}    
gifts.map(x =>
    cataGift(
      b => `An interesting book named ${b.title}`,
      b => `Delicious ${b.taste} chocolate`,
      Id,
      Id,
      x)); 
{% endhighlight %}
One great case for catamorphisms is reuse. We have used the same catamorphism for several different uses instead of creating new tailored functions for everything. That's great!

There are some drawbacks. TypeScript is not great for recursion. This can be solved using folds (perhaps something to write about). And depending on your use case the types can become quite hairy. TypeScript is not the optimal langue for doing this kind of work but as I showed here doable and not that complicated.

