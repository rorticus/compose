# @dojo/compose

[![Build Status](https://travis-ci.org/dojo/compose.svg?branch=master)](https://travis-ci.org/dojo/compose)
[![codecov.io](http://codecov.io/github/dojo/compose/coverage.svg?branch=master)](http://codecov.io/github/dojo/compose?branch=master)
[![npm version](https://badge.fury.io/js/%40dojo%2Fcompose.svg)](https://badge.fury.io/js/%40dojo%2Fcompose)

A composition library, which works well in a TypeScript environment.

**WARNING** This is *beta* software.  While we do not anticipate significant changes to the API at this stage, we may feel the need to do so.  This is not yet production ready, so you should use at your own risk.

## Background

In creating this library, we were looking to solve the following problems with Classes and inheritance in ES6+ and TypeScript:

* Single inheritance: ES6 classes are essentially syntactic sugar around prototypal inheritance. A descended class can only derive from a single ancestor class
* Confusion around mixins and inheritance
* No plans to add mixins to ES6+ (original proposal was withdrawn)
* TypeScript class decorators do not augment the shape of the underlying class based on the decorator, and were not an effective mechanism to provide mixins/traits functionality
* ES6 classes do not support properties in the class prototype
* TypeScript classes do support properties in the class prototype plus some visibility modifiers. However, TypeScript’s private properties are headed on a collision course with ES private properties with are currently proposed for a future version of the language
* Both ES6 and TypeScript classes allow for the mutability of ancestor code, which can cause unexpected side effects
* Traditional inheritance often leads to large base classes and complex inheritance chains

Prior to TypeScript 1.6, we did not have an easy way to solve this, but thankfully the TypeScript team added support for generic types which made this library possible.

## Goals

### Pragmatism

A purely functional library or a purely OO library does not solve the needs of our users.

### Composition

Embrace the concepts of "composition" versus classical Object Oriented inheritance.  The classical model follows a pattern whereby you add functionality to an ancestor by extending it.  Subsequently all other descendants from that class will also inherit that functionality.

In a composition model, the preferred pattern is to create logical feature classes which are then composited together to create a resulting class.  It is believed that this pattern increases code reuse, focuses on the engineering of self contained "features" with minimal cross dependency.

### Factories

The other pattern supported by compose is the factory pattern.  When you create a new class with compose, it will return a factory function.  To create a new instance of an object, you simply call the factory function.  When using constructor functions, where the `new` keyword is used, it limits the ability of construction to do certain things, like the ability for resource pooling.

### Immutability

Also, all the classes generated by the library are "immutable".  Any extension of the class will result in a new class constructor and prototype.  This is in order to minimize the amount of unanticipated consequences of extension for anyone who is referencing a previous class.

The library was specifically designed to work well in a environment where TypeScript is used, to try to take advantage of TypeScript's type inference, intersection types, and union types.  This in ways constrained the design, but we feel that it has created an API that is very semantically functional.

### Challenges and possible changes

The TypeScript 2.2 team made a recent change to their Class implementation that improves support for mixins and composable classes, which may be sufficient for most use cases.

In parallel, we've found that there remain challenges in properly typing widgets and other composed classes, which is a potential barrier to entry for new and experienced users. Furthermore, we're finding that the promise of composition has not been fully appreciated with `@dojo/widgets`. For example, with the `createDialog` widget, it's already dependent on `themeable` and `createWidgetBase`. Both of these depend on `createEvented`, which depends on `createDestroyable`. While four layers deep isn't terrible, `dojo/compose` is not currently preventing us from repeating history unfortunately.

As such, we are exploring options for leveraging TypeScript 2.2 Classes for Dojo 2, which may change dojo/compose or may reduce our reliance on it. We'll have an update once we know more. Regardless of the final approach we take, Dojo 2 will have a solid solution for object composition.

## Usage

To use `@dojo/compose`, install the package along with its required peer dependencies:

```bash
npm install @dojo/compose

# peer dependencies
npm install @dojo/core
npm install @dojo/has
npm install @dojo/shim
```

## Features

- [Class Creation](#class-creation)
  - [Creation](#creation)
  - [Creation with Initializer](#creation-with-initializer)
- [Class Extension](#class-extension)
  - [Implementing an Interface](#implementing-an-interface)
- [Adding Initialization Functions](#adding-initialization-functions)
- [Merging of Arrays](#merging-of-arrays)
- [Using Generics](#using-generics)
- [Overlaying Functionality](#overlaying-functionality)
- [Adding static properties to a factory](#adding-static-properties-to-a-factory)
- [Mixins](#mixins)

The examples below are provided in TypeScript syntax.  The package does work under JavaScript, but for clarity, the examples will only include one syntax.  See below for how to utilize the package under JavaScript.

### Class Creation

The library supports creating a "base" class from ES6 Classes, JavaScript constructor functions, or an object literal prototype.  In addition an initialization function can be provided.

#### Creation

The `compose` module's default export is a function which creates classes.  This is also available as `.create()` which is decorated onto the `compose` function.

If you want to create a new class via a prototype and create an instance of it, you would want to do something like this:

```typescript
import compose from '@dojo/compose/compose';

const fooFactory = compose({
	foo: function () {
		console.log('foo');
	},
	bar: 'bar',
	qat: 1
});

const foo = fooFactory();
```

If you want to create a new class via an ES6/TypeScript class and create an instance of it, you would want to do something like this:

```typescript
import compose from '@dojo/compose/compose';

class Foo {
	foo() {
		console.log('foo');
	};
	bar: string = 'bar';
	qat: number = 1;
}

const fooFactory = compose(Foo);

const foo = fooFactory();
```

You can also subclass:

```typescript
import compose from '@dojo/compose/compose';

const fooFactory = compose({
	foo: function () {
		console.log('foo');
	},
	bar: 'bar',
	qat: 1
});

const myFooFactory = compose(fooFactory);

const foo = myFooFactory();
```

#### Creation with Initializer

During creation, `compose` takes a second optional argument, which is an initializer function.  The constructor pattern for all `compose` classes is to take an optional `options` argument.  Therefore the initialization function should take this optional argument:

```typescript
import compose from '@dojo/compose/compose';

interface FooOptions {
	foo?: Function,
	bar?: string,
	qat?: number
}

function fooInit(options?: FooOptions) {
	if (options) {
		for (let key in options) {
			this[key] = options[key]
		}
	}
}

const fooFactory = compose({
	foo: function () {
		console.log('foo');
	},
	bar: 'bar',
	qat: 1
}, fooInit);

const foo1 = fooFactory();
const foo2 = fooFactory({
	bar: 'baz'
});
```

### Class Extension

The `compose` module's default export also has a property, `extend`, which allows the enumerable, own properties of a literal object or the prototype of a class or ComposeFactory to be added to the prototype of a class. The type of the resulting class will be inferred and include all properties of the extending object. It can be used to extend an existing compose class like this:

```typescript
import * as compose from 'dojo/compose';

let fooFactory = compose.create({
    foo: 'bar'
});

fooFactory = compose.extend(fooFactory, {
    bar: 1
});

let foo = fooFactory();

foo.foo = 'baz';
foo.bar = 2;
```

Or using chaining:

```typescript
import * as compose from 'dojo/compose';

const fooFactory = compose.create({
    foo: 'bar'
}).extend({
    bar: 1
});

let foo = fooFactory();

foo.foo = 'baz';
foo.bar = 2;
```

#### Implementing an interface

`extend` can also be used to implement an interface:

```typescript
import * as compose from 'dojo/compose';

interface Bar {
    bar?: number;
}

const fooFactory = compose.create({
    foo: 'bar'
}).extend<Bar>({});
```

Or

```typescript
const fooFactory = compose.create({
    foo: 'bar'
}).extend(<Bar> {});
```
### Adding Initialization Functions

As factories are extended or otherwise modified, it is often desirable to
provide additional initialization logic for the new factory. The `init` method
can be used to provide a new initializer to an existing factory. The type
of the instance and options will default to the type of the compose factory
prototype and the type of the options argument for the last provided
initializer.

```typescript
const createFoo = compose({
	foo: ''
}, (instance, options: { foo: string } = { foo: 'foo' }) => {
	// Instance type is inferred based on the type passed to
	// compose
	instance.foo = options.foo;
});

const createFooWithNewInitializer = createFoo
	.init((instance, options?) => {
		// If we don't type the options it defaults to { foo: string }
		instance.foo = (options && options.foo) || instance.foo;
	});

const createFooBar = createFoo
	.extend({ bar: 'bar' })
	.init((instance, options?) => {
		// Instance type is updated as the factory prototype is
		// modified, it now has foo and bar properties
		instance.foo = instance.bar = (options && options.foo) || instance.foo;
	});
```


Sometimes, as in the `createFooBar` example above, additional properties may need to be added to the options parameter of the initialize function. A new type can be specified as a generic or by explicitly typing options in the function declaration.

```typescript
const createFoo = compose({
	foo: ''
}, (instance, options: { foo: string } = { foo: 'foo' }) => {
	instance.foo = options.foo;
});

const createFooBar = createFoo
	.extend({ bar: 'bar' })
	// Extend options type with generic
	.init<{ foo: string, bar: string }>((instance, options?) => {
		instance.foo = (options && options.foo) || 'foo';
		instance.bar = (options && options.bar) || 'bar';
	});

const createFooBarToo = createFoo
	.extend({ bar: 'bar' })
	// Extend options type in function signature
	.init(instance, options?: { foo: string, bar: string }) => {
		instance.foo = (options && options.foo) || 'foo';
		instance.bar = (options && options.bar) || 'bar';
	});
```
### Merging of Arrays

When mixing in or extending classes which contain array literals as a value of a property, `compose` will merge these values
instead of over writing, which it does with other value types.

For example, if I have an array of strings in my original class, and provide a mixin which shares the same property that is
also an array, those will get merged:

```typescript
const createFoo = compose({
	foo: [ 'foo' ]
});

const createBarMixin = compose({
	foo: [ 'bar' ]
});

const createFooBar = createFoo.mixin(createBarMixin);

const foo = createFooBar();

foo.foo; // [ 'foo', 'bar' ]
```

There are some things to note:

* The merge process will eliminate duplicates.
* When the factory is invoked, it will "duplicate" the array from the prototype, so `createFoo.prototype.foo !== foo.foo`.
* If the source and the target are not arrays, like other mixing in, last one wins.

### Using Generics

`compose` utilizes TypeScript generics and type inference to type the resulting classes.  Most of the time, this will work without any need to declare your types.  There are situations though where you may want to be more explicit about your interfaces and `compose` can accommodate that by passing in generics when using the API. Here is an example of creating a class that requires generics using `compose`:

```typescript
class Foo<T> {
    foo: T;
}

class Bar<T> {
    bar(opt: T): void {
        console.log(opt);
    }
}

interface FooBarClass {
	<T, U>(): Foo<T>&Bar<U>;
}

let fooBarFactory: FooBarClass = compose(Foo).extend(Bar);

let fooBar = fooBarFactory<number, any>();
```

### Overlaying Functionality

If you want to make modifications to the prototype of a class that are difficult to perform with simple mixins or extensions, you can use the `overlay` function provided on the default export of the `compose` module. `overlay` takes one argument, a function which will be passed a copy of the prototype of the existing class, and returns a new class whose type reflects the modifications made to the existing prototype:

```typescript
import * as compose from 'dojo/compose';

const fooFactory = compose.create({
    foo: 'bar'
});

const myFooFactory = fooFactory.overlay(function (proto) {
    proto.foo = 'qat';
});

const myFoo = myFooFactory();
console.log(myFoo.foo); // logs "qat"
```
Note that as with all the functionality provided by `compose`, the existing class is not modified.

### Adding static properties to a factory

If you want to add static methods or constants to a `ComposeFactory`, the `static` method allows you to do so. Any properties
 set this way cannot be altered, as the returned factory is frozen. In order to modify or remove a static property
on a factory, a new factory would need to be created.
```typescript
const createFoo = compose({
	foo: 1
}).static({
	doFoo(): string {
		return 'foo';
	}
});

console.log(createFoo.doFoo()); // logs 'foo'

// This will throw an error
// createFoo.doFoo = function() {
//	 return 'bar'
// }

const createNewFoo = createFoo.static({
	doFoo(): string {
		return 'bar';
	}
});

console.log(createNewFoo.doFoo()); // logs 'bar'
```

If a factory already has static properties, calling its static method again will not maintain those properties on the
returned factory. The original factory will still maintain its static properties.
```typescript
const createFoo = compose({
	foo: 1
}).static({
	doFoo(): string {
		return 'foo';
	}
})

console.log(createFoo.doFoo()); //logs 'foo'

const createFooBar = createFoo.static({
	doBar(): string {
		return 'bar';
	}
});

console.log(createFooBar.doBar()); //logs 'bar'
console.log(createFoo.doFoo()); //logs 'foo'
//console.log(createFooBar.doFoo()); Doesn't compile
//console.log(createFoo.doBar()); Doesn't compile
```

Static properties will also be lost when calling mixin or extend. Because of this, static properties should be applied
to the 'final' factory in a chain.

### Mixins

One of the goals of compose is to enable the reuse of code, and to allow
clean separation of concerns. Mixins provide a way to encapsulate
functionality that may be reused across many different factories.

This example shows how to create and apply a mixin:
```typescript
const createFoo = compose({ foo: 'foo'});

const fooMixin = compose.createMixin(createFoo);

createFoo.mixin(fooMixin);
```

In this case the mixin won't actually do anything, because we applied it
immediately after creating it. Another thing to note in this exapmle, is
that passing `createFoo` to `createMixin` is optional, but is generally
a good idea. This lets the mixin know that it should be mixed into something
that provides at least the same functionality as `createFoo`, so the mixin
can automatically include the prototype and options types from `createFoo`.

In order to create a mixin that's actually useful, we can use any of the
`ComposeFactory` methods discussed above. The mixin will record these calls,
and when mixed into a factory will apply them as if they were called directly
on the factory.


```typescript
const createFoo = compose({
	foo: 'foo'
}, (instance, options?: { foo: string }) => {
	instance.foo = (options && options.foo) || 'foo';
});

const createFooBar = createFoo.extend({ bar: 'bar'});

const fooMixin = compose.createMixin(createFoo)
	// Because we passed createFoo, the types of instance and options
	// are both { foo: string }
	.init((instance, options?) => {
		instance.foo = (options && options.foo) + 'bar';
	});
	.extend({ baz: 'baz'});

const createFooBaz = createFoo.mixin(fooMixin);
/* Equivalent to calling
	createFoo
        .init((instance, options?) => {
            instance.foo = (options && options.foo) + 'bar';
        });
        .extend({ baz: 'baz'});
*/

const createFooBarBaz = createFooBar.mixin(fooMixin);
/* Equivalent to calling
	createFooBar
        .init((instance, options?) => {
            instance.foo = (options && options.foo) + 'bar';
        });
        .extend({ baz: 'baz'});
*/
```

Compose also provides the ability to mixin a factory directly, or a
`FactoryDescriptor` object, but these are allowed only for the backwards
compatibility. The `createMixin` API is the preferred method for creating
and applying mixins.
## How do I use this package?

The easiest way to use this package is to install it via `npm`:

```
$ npm install @dojo/compose
```

In addition, you can clone this repository and use the Grunt build scripts to manage the package.

Using under TypeScript or ES6 modules, you would generally want to just `import` the `@dojo/compose/compose` module:

```typescript
import compose from '@dojo/compose/compose';

const createFoo = compose({
	foo: 'foo'
}, (instance, options) => {
	/* do some initialization */
});

const foo = createFoo();
```
## How do I contribute?

We appreciate your interest!  Please see the [Contributing Guidelines](https://github.com/dojo/meta/blob/master/CONTRIBUTING.md) and [Style Guide](https://github.com/dojo/meta/blob/master/STYLE.md).

### Installation

To start working with this package, clone the repository and run `npm install`.

In order to build the project run `grunt dev` or `grunt dist`.

### Testing

Test cases MUST be written using [Intern](https://theintern.github.io) using the Object test interface and Assert assertion interface.

90% branch coverage MUST be provided for all code submitted to this repository, as reported by Istanbul’s combined coverage results for all supported platforms.

## Prior Art and Inspiration

A lot of thinking, talks, publications by [Eric Elliott](https://ericelliottjs.com/) (@ericelliott) inspired @bryanforbes and @kitsonk to take a look at the composition and factory pattern.

@kriszyp helped bring AOP to Dojo 1 and we found a very good fit for those concepts in `dojo/compose`.

[`dojo/_base/declare`](https://github.com/dojo/dojo/blob/master/_base/declare.js) was the starting point for bringing Classes and classical inheritance to Dojo 1 and without @uhop we wouldn't have had Dojo 1's class system.

@pottedmeat and @kitsonk iterated on the original API, trying to figure a way to get types to work well within TypeScript and @maier49 worked with the rest of the [dgrid](https://github.com/SitePen/dgrid) team to make the whole API more usable.

© 2015 - 2017 JS Foundation. [New BSD](http://opensource.org/licenses/BSD-3-Clause) license.
