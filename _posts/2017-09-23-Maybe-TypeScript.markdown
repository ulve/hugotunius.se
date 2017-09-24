---
layout: post
title:  "Maybe TypeScript"
date:   2017-09-23 12:30:00 +0000
categories: typescript functional
---

`Maybe` är en datastruktur som är rätt så praktisk att ha. Man använder den när det inte är säkert att det finns någon data men man vill inte göra specialfall som kollar om så är fallet.
I många funktionella språk så används den som returtyp där vanliga språk hade haft möjligheten att returnera `null`

Vad är då `Maybe`. Ja det är en algebraisk datatyp. En sumtyp. I `TypeScript` kallas dom `tagged unions`. Vad är nu det? Det är en datatyp som kan vara två olika datatyper. Tänk en abstrakt basklass och två barknklasser som har den som förälder. Fast bättre. Om du har en funktion som kan returnera två olika typer av värden så är det alltid problem att modellera det. Man kan returnera någon form av datastruktur som innehåller både stärngen och numret

{% highlight ts %}

interface StrOrInt {
  myString: string;
  myInt: number;
}

function Test1(val: bool):StrOrInt {
  if(val) {
    return {myString: "Hello", myInt: null}
  } else {
    return {myString: null, myInt: 7}
  }
}

let a = Test1(true)
if(a.myString !== null) {
  // Gör saker med strängen
} else {
  // Gör saker med siffran
}

{% endhighlight %}

Det är för sådana scenarion som en `tagged-union` är perfekt att använda. Det är ett sätt att fånga t.ex. affärslogik på ett enkelt och bra sätt. Den här metoden kan antingen returnera en ansökan eller ett Felmeddelande till handläggaren

{% highlight ts %}
type Retur = Error | Ansökan

function SkapaÄrende() : Retur {
  // validera
  return new Error();
  // eller 
  return new Ansökan();
}
{% endhighlight %}

Hur det fungerar i detalj tänker jag inte gå in på men det är i alla fall grunden till hur man gör en `Maybe` struktur.

`Maybe` är just en sådan `tagged-union` som kan vara `Just` eller `Nothing`. Just är en generisk datatyp så den kan innehålla vad som helst. Man kan se hela `Maybe` som en låda som man stoppar värden i. Det är oftast så dom förklaras. Sedan finns det funktioner som vet hur man manipulerar det lagrade värdet i lådan.

Så här är en enkel implementation av `Just` och `Nothing`

{% highlight ts %}

// Generisk så den kan innehålla just av vad som helst
class Just<T> { 
  // Det här har med tagged unions att göra
  // Det är en så kallad string literal. En
  // variabel som endast kan innehålla värdet
  // "just". Den används av TypeScript för att
  // få datatyperna att fungera
  kind: "just"; 
  
  // vi håller värdet som man stoppar in här
  value: T;
  
  // och när vi constructar den så stoppar vi in värdet
  constructor(a: T) {
    this.value = a; 
    this.kind = "just"; // och sätter typen. obs!
  }

  // Det här är metoden som används för att göra saker
  // inne i "lådan". Den kallas ofta map. 
  // Det finns anlednignar till det och det
  // är ungefär samma anledningar som att
  // funktionen som vet hur man göra saker 
  // med innehållet i en array också heter 
  // map.
  public map(f: (a:any) => any): Just<T> {
    // Det map gör är att den tar en funktion
    // Sen anropar den funktionen med det lagrade
    // värdet som finns här.
    // Sen returnerar den en ny Just med det nya värdet    
    return new Just(f(this.value)); 
  }
}

// Den andra klassen. Den representerar att värde saknas
class Nothing {
  kind: "nothing"; // än en gång
  constructor() {
    this.kind = "nothing";
  }

  // Vi har fortfarande en map här. Den gör inget
  // returnerar bara en ny Nothing
  // Det är ett bra trick för då behöver man inte 
  // veta om det är en Just eller Nothing. Båda har 
  // map som vet vcad dom skall göra.
  public map(f: (a:any) => any): Nothing {
    return new Nothing(); 
  }
}

{% endhighlight %}


Och så gör vi en `tagged-union` av det hela. Det här innebär att vi har en typ som heter `Maybe` som kan vara antingen `Nothing` eller `Just` det här kan man göra med alla möjliga typer. 

{% highlight ts %}

type Maybe<T> = Nothing | Just<T>;

{% endhighlight %}

Nu har vi alltså en datastruktur som gör att man inte behöver kolla om det finns ett värde eller inte. Vi kan jobba på samma sätt oavsett det!

{% highlight ts %}
let a = new Just(3);
let b = a.map(x => x + 3);
//=> Just(6)

// eller
let c = new Nothing();
let d = c.map(x => x + 3);
// Här har vi ju inget värde? Fungerar det att låtsas 
// att vi har ett och bara anropa en funktion sådär? Jodå!
//=> Nothing()

{% endhighlight %}

Problemet nu är ju att vi aldrig har vårt värde. Vi har bara värdet fast i vår lilla låda. Men bli inte rädd! Vi gör en funtion för att plocka värdet ur en `Maybe`! Här får vi ange ett värde ifall det inte finns något.

{% highlight ts %}

function FromMaybeWithDefault<T>(v: Maybe<T>, defaultValue: T): T {
  // Här kollar vi typen Typescript
  // är så smart att om vi missar en typ här
  // så vår switch är ofullständig så kommer
  // den att klaga vid kompilering. 
  switch (v.kind) { 
    case "just":
      // är det en just så plockar vi värdet ur justen
      return v.value; 
    case "nothing":
      // är det en nothing så returnerar vi defaultvärdet
      return defaultValue;     
  }
}

console.log(FromMaybeWithDefault(b, 0));
//=> 6
console.log(FromMaybeWithDefault(d, 0));
//=> 0

{% endhighlight %}

Vill man då använda det i verkligheten är något i stil med det här bra. Den tar ett värde och är det null eller undefined så blir det nothing annars stoppar den värdet i vår låda

{% highlight ts %}

function MaybeOf<T>(a: any): Maybe<T> {
  if (a === undefined || a === null) 
    return new Nothing();
  else 
    return new Just(a);
}

{% endhighlight %}

Här har vi en ful funktion. Den returnerar ibland null och ibland siffror. Jag har i alla fall varit nog snäll och berättat det men det behövs ju inte ens! Dåligt!

{% highlight ts %}

const CrapReturn = (a: number): null | number => {
  if (a % 2 == 0) 
    return null;
  else 
    return a;
};

// Här är en funktion som multiplicerar med 10 skickar man 
// in null här blir det inte roligt!
const Mult10 = (a: number) => a * 10;

// okej en av dom här borde då vara null men vi behöver inte kolla om något
// bara att köra på som om det inte fanns någon morgondag
let e = MaybeOf(CrapReturn(7))
  .map(x => x + 3) // lägg till tre
  .map(Mult10); // anropa en funktion som multiplicerar med tio
let f = MaybeOf(CrapReturn(8))
  .map(x => x + 3)
  .map(Mult10);
// det här andra exemplet skulle ju kräva if-satser och nästlade grejer. 
// Fult och långt i onödan. Nu slipper vi det!
// Typernas magi!

console.log(FromMaybeWithDefault(e, 0));
//=> 100
console.log(FromMaybeWithDefault(f, 0));
//=> 0

{% endhighlight %}

Allt det här är toppen. Den stora styrkan är att man inte behöver kolla saker hela tiden utan man kan göra på ett sätt rakt igenom och låta datastrukturen hantera när det blir avvikelser. Det finns andra strukturer som likar den här och dom är alla mycket kraftfulla och vet man bara om dom så är det lätt att använda.