---
title: Warum verwenden wir super(props)?
date: '30.11.2018'
spoiler: Am Ende gibt es eine Wendung.
---


Ich habe gehört [Hooks](https://reactjs.org/docs/hooks-intro.html) sind jetzt in aller Munde. Lustigerweise möchte ich diesen Blog damit beginnen, witze Fakten über *class* Komponenten zu erläutern. Damit habt ihr nicht gerechnet!

**All diese Dinge sind *nicht* relevant, damit du React produktiv verwenden kannst. Aber sie können interessant sein für dich, wenn du genauer herausfinden möchtest, wie diese Dinge funktionieren**

Und schon starten wir mit dem ersten Fakt.

---

Ich habe `super(props)` häufiger in meinem Leben geschrieben, als mir lieb ist:

```jsx{3}
class Checkbox extends React.Component {
  constructor(props) {
    super(props);
    this.state = { isOn: true };
  }
  // ...
}
```

Der [Klassenfeld Vorschlag(class fields proposal)](https://github.com/tc39/proposal-class-fields) lässt uns genau diese Prozedur überspringen:

```jsx
class Checkbox extends React.Component {
  state = { isOn: true };
  // ...
}
```

Dieser Syntax war schon in React 0.13 [geplant](https://reactjs.org/blog/2015/01/27/react-v0.13.0-beta-1.html#es7-property-initializers), als React, im Jahre 2015, die "einfachen Klassen" hinzugefügt hat. `constructor` und der Aufruf von `super(props)` war nur als eine temporäre Lösung angedacht, bis für Klassen felder eine bessere Alternative erscheint.

Aber zurück zu dem Beispiel, welches nur Funktionen aus ES2015 verwendet:

```jsx{3}
class Checkbox extends React.Component {
  constructor(props) {
    super(props);
    this.state = { isOn: true };
  }
  // ...
}
```

**Warum rufen wir `super` auf? Können wir es *nicht einfach überspringen*? Wenn wir es verwenden müssen, was passiert, wenn wir wir nicht die Variable `props` übergeben? Gibt es noch andere Parameter?** Lasst es uns herausfinden.

---

In JavaScript ruft `super` den Konstruktor der übergeordneten Klasse auf. (In unserem Beispiel, wäre das der Konstruktor der `React.Componenet` Klasse.)

Dabei musst du beachten, dass du `this` erst verwenden kannst, *nachdem* du den Konstruktor der übergeordneten Klasse aufgerufen hast. JavaScript lässt dich dies sonst nicht tun:

```jsx
class Checkbox extends React.Component {
  constructor(props) {
    // 🔴 `this` kann noch nicht verwendet werden
    super(props);
    // ✅ Jetzt kann es verwendet werden
    this.state = { isOn: true };
  }
  // ...
}
```

Es gibt einen guten Grund, warum JavaScript dies von dir verlangt. Stellt dir die folgende Klassenhierarchie vor:

```jsx
class Person {
  constructor(name) {
    this.name = name;
  }
}

class PolitePerson extends Person {
  constructor(name) {
    this.greetColleagues(); // 🔴 Das ist nicht erlaubt, lies weiter um herauszufinden wieso dies so ist
    super(name);
  }
  greetColleagues() {
    alert('Good morning folks!');
  }
}
```

Stell dir vor, es ist *erlaubt* `this` zu verwenden, bevor `super` aufgerufen wurde. Einen Monat später, würden wir vielleicht `greetColleagues` so abändern, dass die Methode den Namen der Person ausgibt:

```jsx
  greetColleagues() {
    alert('Good morning folks!');
    alert('My name is ' + this.name + ', nice to meet you!');
  }
```

Aber wir haben vergessen, dass `this.greetColleagues()` vor `super()` aufgerufen wird. Deshalb hatte der `super()` Aufruf noch keine Chance die Variable `this.name` zu erstellen. Aus diesem Grund ist `this.name` bisher nicht definiert! Wie du sehen kannst, ist es bei einem Code, wie diesem, sehr schwierig nach einiger Zeit an diese Dinge zu denken.

Um solche Fettnäpfchen zu vermeiden, **zwingt JavaScript dich dazu zuerst `super` aufzurufen, wenn du `this` in einem Konstruktor verwenden möchtest`**. Lass die übergeordnete Klasse zuerst ihre Arbeit tun! Diese Limitation tritt auch bei React Komponenten Klasse auf:

```jsx
  constructor(props) {
    super(props);
    // ✅ `this` kann benutzt werden
    this.state = { isOn: true };
  }
```

Die andere Frage ist aber, warum übergeben wir `props` an die übergeordnete Klasse?

---

Jetzt denkst du vermutlich es ist notwendig, `props` mit `super` zuübergeben, damit der Basis `React.Componenet` Konstruktor `this.props` initializieren kann:

```jsx
// Inside React
class Component {
  constructor(props) {
    this.props = props;
    // ...
  }
}
```

Genau das ist tatsächlich nicht weit entfernt von der Realität. Genau das [tut es](https://github.com/facebook/react/blob/1d25aa5787d4e19704c049c3cfa985d3b5190e0d/packages/react/src/ReactBaseClasses.js#L22).

Allerdings ist es auch möglich `this.props` zu verwenden, wenn du das `props` Argument nicht beim Aufruf von `super()` übergibst. In der `render` Methode zum Beispiel ist es immer möglich `this.props` aufzurufen. (Wenn du mir nicht glaubst, probier es aus!)

Wie funktioniert das? Es hat sich herrausgestellt, dass **React, `props` nachdem Aufruf *deines* Konstruktors initialisiert:**

```jsx
  // Inside React
  const instance = new YourComponent(props);
  instance.props = props;
```

So even if you forget to pass `props` to `super()`, React would still set them right afterwards. There is a reason for that.

When React added support for classes, it didn’t just add support for ES6 classes alone. The goal was to support as wide range of class abstractions as possible. It was [not clear](https://reactjs.org/blog/2015/01/27/react-v0.13.0-beta-1.html#other-languages) how relatively successful would ClojureScript, CoffeeScript, ES6, Fable, Scala.js, TypeScript, or other solutions be for defining components. So React was intentionally unopinionated about whether calling `super()` is required — even though ES6 classes are.

So does this mean you can just write `super()` instead of `super(props)`?

**Probably not because it’s still confusing.** Sure, React would later assign `this.props` *after* your constructor has run. But `this.props` would still be undefined *between* the `super` call and the end of your constructor:

```jsx{14}
// Inside React
class Component {
  constructor(props) {
    this.props = props;
    // ...
  }
}

// Inside your code
class Button extends React.Component {
  constructor(props) {
    super(); // 😬 We forgot to pass props
    console.log(props);      // ✅ {}
    console.log(this.props); // 😬 undefined
  }
  // ...
}
```

It can be even more challenging to debug if this happens in some method that’s called *from* the constructor. **And that’s why I recommend always passing down `super(props)`, even though it isn’t strictly necessary:**

```jsx
class Button extends React.Component {
  constructor(props) {
    super(props); // ✅ We passed props
    console.log(props);      // ✅ {}
    console.log(this.props); // ✅ {}
  }
  // ...
}
```

This ensures `this.props` is set even before the constructor exits.

-----

There’s one last bit that longtime React users might be curious about.

You might have noticed that when you use the Context API in classes (either with the legacy `contextTypes` or the modern `contextType` API added in React 16.6), `context` is passed as a second argument to the constructor.

So why don’t we write `super(props, context)` instead? We could, but context is used less often so this pitfall just doesn’t come up as much.

**With the class fields proposal this whole pitfall mostly disappears anyway.** Without an explicit constructor, all arguments are passed down automatically. This is what allows an expression like `state = {}` to include references to `this.props` or `this.context` if necessary.

With Hooks, we don’t even have `super` or `this`. But that’s a topic for another day.
