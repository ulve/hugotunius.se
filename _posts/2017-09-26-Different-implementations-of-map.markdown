---
layout: post
title:  "Different implementations of map"
date:   2017-09-23 12:30:00 +0000
categories: javascript functional
---

The first way is to simply create a new list and mutate it by adding elements to it.

{% highlight js %}

// Mutatation
// map :: (a -> b) -> [a] -> [b]
const map = (m, l) => {
    let ret = []
    for (var i=0; i < l.length; i++) {
        ret.push(m(l[i]))
    }
    return ret
}

map(x => x + 1, [1, 2, 3])
//=> [2, 2, 4]

{% endhighlight %}

Another way is to solve it with recursion.

{% highlight js %}

// Recursion
// rmap :: (a -> b) -> [a] -> [b]? -> [b]
const rmap = (m, [h, ...t], acc = []) =>
    // If head contains an element recursively call rmap with the tail.
    // If head is empty return the accumulator
    h ? rmap(m, t, [...acc, m(h)]) : acc 

rmap(x => x + 1, [4, 5, 6])
//=> [5, 6, 7]

{% endhighlight %}

The third way is to use a catamorphism.

{% highlight js %}

// Catamorphism
// cmap :: (a -> b) -> [a] -> [b]
const cmap = (m, l) => 
    // reduce the array. Apply the current elemet to the function
    // add that to the accumulator array and continue
    l.reduce((acc, h) => [...acc, m(h)], [])

cmap(x => x + 1, [7, 8, 9])
//=> [8, 9, 10]
{% endhighlight %}