---
layout: post
title:  "Maybe TypeScript"
date:   2017-09-24 12:30:00 +0000
categories: typescript functional
---

Vi gör en egen Maybeklass. Så här brukar dom se ut

Först en klass som heter just och kan ta vilket värde som helst och hålla det i en liten låda. Observera att den har en property som heter kind och är av typen "just". Det är lite konstigt. Det är en string-literal. En variabel som bara kan innehålla värdet "just".

Sen gör vi en klass som heter Nothing. Den är innehåller inget. Det enda den har som är intressant är en string-literal och en map som inte gör något.

{% highlight ts %}

class Just<T> { // Generisk så den kan innehålla just av vad som helst
  kind: "just"; // okej skulle testa det här tror man måste ha det
  value: T; // vi håller värdet som man stoppar in här
  constructor(a: T) {
    this.value = a; // och när vi constructar den så stoppar vi in värdet
    this.kind = "just"; // och sätter typen. obs!
  }
  public map(f: (a:any) => any): Just<T> {
    // sen gör vi så att man kan anropa map på den
    return new Just(f(this.value)); // det här är en mer generisk map än den som finns på arrayer.
    // den vet hur man går in i den här klassen och manipulerar värdet vi har i vår låda
  }
}

class Nothing {
  kind: "nothing"; // än en gång
  constructor() {
    this.kind = "nothing";
  }
  public map(f: (a:any) => any): Nothing {
    return new Nothing(); // men här är tricket. Vi gör så att man kan mappa över den.
  }
}

{% endhighlight %}


Och så gör vi en tagged union av det hela. Det här innebär att vi har en typ som heter maybe som kan vara antingen Nothing eller Just det här kan man göra med alla möjliga typer. vill man ha en funktion som tar antingen en sträng eller ett nummer så gör man type `MinUnion = string | number` då går det inte skicka in arrayer i funktioner som tar just en MinUnion

{% highlight ts %}
type Maybe<T> = Nothing | Just<T>;
{% endhighlight %}


Nu kan man göra saker som

{% highlight ts %}
let a = new Just(3);
let b = a.map(x => x + 3);
//=> Just(6)

// eller
let c = new Nothing();
let d = c.map(x => x + 3);
// Här har vi ju inget värde? Fungerar det att låtsas att vi har ett och bara anropa en funktion sådär? Jodå!
// Nothing()

{% endhighlight %}

Problemet nu är ju att vi aldrig har vårt värde. Vi har bara värdet fast i vår lilla låda. Men bli inte rädd! Vi gör en funtion för att plocka värdet ur en Maybe! Här får vi ange ett värde ifall det inte finns något.

{% highlight ts %}
function FromMaybeWithDefault<T>(v: Maybe<T>, defaultValue: T): T {
  switch (v.kind) { // Här kollar vi typen
    case "just":
      return v.value; // är det en just så plockar vi värdet ur justen
    case "nothing":
      return defaultValue; // är det en nothing så returnerar vi defaultvärdet
    // skulle vi nu missa att köra en case t.ex. skippa nothing så kommer typescript att varna. så smart är den
  }
}

console.log(FromMaybeWithDefault(b, 0));
// 6
console.log(FromMaybeWithDefault(d, 0));
// 0

{% endhighlight %}

Vill man då använda det i verkligheten är något i stil med det här bra. Den tar ett värde och är det null eller undefined så blir det nothing annars stoppar den värdet i vår låda

{% highlight ts %}
function MaybeOf<T>(a: any): Maybe<T> {
  if (a === undefined || a === null) return new Nothing();
  else return new Just(a);
}
{% endhighlight %}

Här har vi en ful funktion. Den returnerar ibland null och ibland siffror. Jag har i alla fall varit nog snäll och berättat det men det behövs ju inte ens! Dåligt!
{% highlight ts %}
const FunktionSomKanReturneraNull = (a: number): null | number => {
  if (a % 2 == 0) return null;
  else return a;
};

// Här är en funktion som multiplicerar med 10 skickar man in null här blir det inte roligt!
const FunktionSomtarEnIntOchMultiplicerarMedTio = (a: number) => a * 10;

// okej en av dom här borde då vara null men vi behöver inte kolla om något
// bara att köra på som om det inte fanns någon morgondag
let e = MaybeOf(FunktionSomKanReturneraNull(7))
  .map(x => x + 3) // lägg till tre
  .map(FunktionSomtarEnIntOchMultiplicerarMedTio); // anropa en funktion som multiplicerar med tio
let f = MaybeOf(FunktionSomKanReturneraNull(8))
  .map(x => x + 3)
  .map(FunktionSomtarEnIntOchMultiplicerarMedTio);
// det här andra exemplet skulle ju kräva if-satser och nästlade grejer. Fult och långt i onödan. Nu slipper vi det!
// Typernas magi!

console.log(FromMaybeWithDefault(e, 0));
// 100
console.log(FromMaybeWithDefault(f, 0));
// 0
{% endhighlight %}