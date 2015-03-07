# Summary

Decorators make it possible to annotate and modify classes and properties at
design time.

While ES5 object literals support arbitrary expressions in the value position,
ES6 classes only support literal functions as values. Decorators restore the
ability to run code at design time, while maintaining a declarative syntax.

# Motivation

[TODO]

# Detailed Design

A decorator is:

* an expression
* that evaluates to a function
* that takes the target, name, and property descriptor as arguments
* and optionally returns a property descriptor to install on the target object

Consider a simple class definition:

```js
class Person {
  name() { return `${this.first} ${this.last}` }
}
```

Evaluating this class results in installing the `name` function onto
`Person.prototype`, roughly like this:

```js
Object.defineProperty(Person.prototype, 'name', {
  value: specifiedFunction,
  enumerable: false,
  configurable: true,
  writable: true
});
```

A decorator precedes the syntax that defines a property:

```js
class Person {
  @readonly
  name() { return `${this.first} ${this.last}` }
}
```

Now, before installing the descriptor onto `Person.prototype`, the engine first
invokes the decorator:

```js
let descriptor = {
  value: specifiedFunction,
  enumerable: false,
  configurable: true,
  writable: true
};

descriptor = readonly(Person.prototype, 'name', descriptor) || descriptor;
Object.defineProperty(Person.prototype, 'name', descriptor);
```

The decorator has the same signature as `Object.defineProperty`, and has an
opportunity to intercede before the relevant `defineProperty` actually occurs.

A decorator that precedes syntactic getters and/or setters operates on the
accessor descriptor:

```js
class Person {
  @nonenumerable
  get kidCount() { return this.children.length; }
}

function nonenumerable(target, name, descriptor) {
  descriptor.enumerable = val;
  return descriptor;
}
```

A more detailed example illustrating a simple decorator that memoizes an
accessor.

```js
class Person {
  @memoize
  get name() { return `${this.first} ${this.last}` }
  set name(val) {
    let [first, last] = val.split(' ');
    this.first = first;
    this.last = last;
  }
}

let memoized = new WeakMap();
function memoize(target, name, descriptor) {
  let getter = descriptor.get, setter = descriptor.set;

  descriptor.get = function() {
    let table = memoizationFor(this);
    if (name in table) { return table[name]; }
    return table[name] = getter.call(this);
  }

  descriptor.set = function(val) {
    let table = memoizationFor(this);
    setter.call(this, val);
    table[name] = val;
  }
}

function memoizationFor(obj) {
  let table = memoized.get(this);
  if (!table) { table = Object.create(null); memoized.set(this, table); }
  return table;
}
```

It is also possible to decorate the class itself. In this case, the decorator
takes the target constructor.

```js
// A simple decorator
@annotation
class MyClass { }

function annotation(target) {
   // Add a property on target
   target.annotated = true;
}
```

Since decorators are expressions, decorators can take additional arguments and
act like a factory.

```js
@isTestable(true)
class MyClass { }

function isTestable(value) {
   return function decorator(target) {
      target.isTestable = value;
   }
}
```

The same technique could be used on property decorators:

```js
class C {
  @enumerable(false)
  method() { }
}

function enumerable(value) {
  return function (target, key, descriptor) {
     descriptor.enumerable = value;
     return descriptor;
  }
}
```

Because descriptor decorators operate on targets, they also naturally work on
static methods. The only difference is that the first argument to the decorator
will be the class itself (the constructor) rather than the prototype, because
that is the target of the original `Object.defineProperty`.

For the same reason, descriptor decorators work on object literals, and pass
the object being created to the decorator.

# Desugaring

## Class Declaration

### Syntax

```js
@F("color")
@G
class Foo {
}
```

### Desugaring (ES6)

```js
var Foo = (function () {
  class Foo {
  }

  Foo = F("color")(Foo = G(Foo) || Foo) || Foo;
  return Foo;
})();
```

### Desugaring (ES5)

```js
var Foo = (function () {
  function Foo() {
  }

  Foo = F("color")(Foo = G(Foo) || Foo) || Foo;
  return Foo;
})();
```

## Class Method Declaration

### Syntax

```js
class Foo {
  @F("color")
  @G
  bar() { }
}
```

### Desugaring (ES6)

```js
var Foo = (function () {
  class Foo {
    bar() { }
  }

  var _temp;
  _temp = F("color")(Foo.prototype, "bar",
    _temp = G(Foo.prototype, "bar",
      _temp = Object.getOwnPropertyDescriptor(Foo.prototype, "bar")) || _temp) || _temp;
  if (_temp) Object.defineProperty(Foo.prototype, "bar", _temp);
  return Foo;
})();
```

### Desugaring (ES5)

```js
var Foo = (function () {
  function Foo() {
  }
  Foo.prototype.bar = function () { }

  var _temp;
  _temp = F("color")(Foo.prototype, "bar",
    _temp = G(Foo.prototype, "bar",
      _temp = Object.getOwnPropertyDescriptor(Foo.prototype, "bar")) || _temp) || _temp;
  if (_temp) Object.defineProperty(Foo.prototype, "bar", _temp);
  return Foo;
})();
```

## Class Accessor Declaration

### Syntax

```js
class Foo {
  @F("color")
  @G
  get bar() { }
  set bar(value) { }
}
```

### Desugaring (ES6)

```js
var Foo = (function () {
  class Foo {
    get bar() { }
    set bar(value) { }
  }

  var _temp;
  _temp = F("color")(Foo.prototype, "bar",
    _temp = G(Foo.prototype, "bar",
      _temp = Object.getOwnPropertyDescriptor(Foo.prototype, "bar")) || _temp) || _temp;
  if (_temp) Object.defineProperty(Foo.prototype, "bar", _temp);
  return Foo;
})();
```

### Desugaring (ES5)

```js
var Foo = (function () {
  function Foo() {
  }
  Object.defineProperty(Foo.prototype, "bar", {
    get: function () { },
    set: function (value) { }
    enumerable: true, configurable: true
  });

  var _temp;
  _temp = F("color")(Foo.prototype, "bar",
    _temp = G(Foo.prototype, "bar",
      _temp = Object.getOwnPropertyDescriptor(Foo.prototype, "bar")) || _temp) || _temp;
  if (_temp) Object.defineProperty(Foo.prototype, "bar", _temp);
  return Foo;
})();
```

## Object Literal Method Declaration

### Syntax

```js
var o = {
  @F("color")
  @G
  bar() { }
}
```

### Desugaring (ES6)

```js
var o = (function () {
  var _obj = {
    bar() { }
  }

  var _temp;
  _temp = F("color")(_obj, "bar",
    _temp = G(_obj, "bar",
      _temp = void 0) || _temp) || _temp;
  if (_temp) Object.defineProperty(_obj, "bar", _temp);
  return Foo;
})();
```

### Desugaring (ES5)

```js
var o = (function () {
    var _obj = {
        bar: function () { }
    }

    var _temp;
    _temp = F("color")(_obj, "bar",
        _temp = G(_obj, "bar",
            _temp = void 0) || _temp) || _temp;
    if (_temp) Object.defineProperty(_obj, "bar", _temp);
    return Foo;
})();
```

## Object Literal Accessor Declaration

### Syntax

```js
var o = {
  @F("color")
  @G
  get bar() { }
  set bar(value) { }
}
```

### Desugaring (ES6)

```js
var o = (function () {
  var _obj = {
    get bar() { }
    set bar(value) { }
  }

  var _temp;
  _temp = F("color")(_obj, "bar",
    _temp = G(_obj, "bar",
      _temp = void 0) || _temp) || _temp;
  if (_temp) Object.defineProperty(_obj, "bar", _temp);
  return Foo;
})();
```

### Desugaring (ES5)

```js
var o = (function () {
  var _obj = {
  }
  Object.defineProperty(_obj, "bar", {
    get: function () { },
    set: function (value) { }
    enumerable: true, configurable: true
  });

  var _temp;
  _temp = F("color")(_obj, "bar",
    _temp = G(_obj, "bar",
      _temp = void 0) || _temp) || _temp;
  if (_temp) Object.defineProperty(_obj, "bar", _temp);
  return Foo;
})();
```

# Grammar

## Expressions

# Grammar for Decorator Syntax

  *DecoratorList*<sub> [Yield]</sub> :
   *DecoratorList*<sub> [?Yield]opt</sub>  *Decorator*<sub> [?Yield]</sub>

  *Decorator*<sub> [Yield]</sub> :
   `@` *AssignmentExpression*<sub> [?Yield]</sub>

  *PropertyDefinition*<sub> [Yield]</sub> :
   *IdentifierReference*<sub> [?Yield]</sub>
   *CoverInitializedName*<sub> [?Yield]</sub>
   *PropertyName*<sub> [?Yield]</sub>  `:` *AssignmentExpression*<sub> [In, ?Yield]</sub>
   *DecoratorList*<sub> [?Yield]opt</sub> *MethodDefinition*<sub> [?Yield]</sub>

  *CoverMemberExpressionSquareBracketsAndComputedPropertyName*<sub> [Yield]</sub> :
   `[` *Expression*<sub> [In, ?Yield]</sub> `]`

NOTE  The production *CoverMemberExpressionSquareBracketsAndComputedPropertyName* is used to cover parsing a *MemberExpression* that is part of a *Decorator* inside of an *ObjectLiteral* or *ClassBody*, to avoid lookahead when parsing a decorator against a *ComputedPropertyName*.

  *PropertyName*<sub> [Yield, GeneratorParameter]</sub> :
   *LiteralPropertyName*
   [+GeneratorParameter] *CoverMemberExpressionSquareBracketsAndComputedPropertyName*
   [~GeneratorParameter] *CoverMemberExpressionSquareBracketsAndComputedPropertyName*<sub> [?Yield]</sub>

  *MemberExpression*<sub> [Yield]</sub>  :
   [Lexical goal *InputElementRegExp*] *PrimaryExpression*<sub> [?Yield]</sub>
   *MemberExpression*<sub> [?Yield]</sub> *CoverMemberExpressionSquareBracketsAndComputedPropertyName*<sub> [?Yield]</sub>
   *MemberExpression*<sub> [?Yield]</sub> `.` *IdentifierName*
   *MemberExpression*<sub> [?Yield]</sub> *TemplateLiteral*<sub> [?Yield]</sub>
   *SuperProperty*<sub> [?Yield]</sub>
   *NewSuper* *Arguments*<sub> [?Yield]</sub>
   `new` *MemberExpression*<sub> [?Yield]</sub> *Arguments*<sub> [?Yield]</sub>

  *SuperProperty*<sub> [Yield]</sub> :
   `super` *CoverMemberExpressionSquareBracketsAndComputedPropertyName*<sub> [?Yield]</sub>
   `super` `.` *IdentifierName*

  *ClassDeclaration*<sub> [Yield, Default]</sub> :
   *DecoratorList*<sub> [?Yield]opt</sub> `class` *BindingIdentifier*<sub> [?Yield]</sub> *ClassTail*<sub> [?Yield]</sub>
   [+Default] *DecoratorList*<sub> [?Yield]opt</sub> `class` *ClassTail*<sub> [?Yield]</sub>

  *ClassExpression*<sub> [Yield, GeneratorParameter]</sub> :
   *DecoratorList*<sub> [?Yield]opt</sub> `class` *BindingIdentifier*<sub> [?Yield]opt</sub> *ClassTail*<sub> [?Yield, ?GeneratorParameter]</sub>

  *ClassElement*<sub> [Yield]</sub> :
   *DecoratorList*<sub> [?Yield]opt</sub> *MethodDefinition*<sub> [?Yield]</sub>
   *DecoratorList*<sub> [?Yield]opt</sub> `static` *MethodDefinition*<sub> [?Yield]</sub>

# Notes

In order to more directly support metadata-only decorators, a desired feature
for static analysis, the TypeScript project has made it possible for its users
to define [ambient decorators](https://github.com/jonathandturner/brainstorming/blob/master/README.md#c6-ambient-decorators)
that support a restricted syntax that can be properly analyzed without evaluation.