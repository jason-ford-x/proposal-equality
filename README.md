# Equality API

## Status

Champion(s): *TBC*  
Author(s): *Jason Ford*  
Stage: 0

## About
Inspired by an aspect of the [**composites**](https://github.com/tc39/proposal-composites) proposal, which <u>requires an algorithm for determining if two data structures are identical</u>. A part of this proposal provides a solution to the same problem space and so thus competes with **composites**.

Such an algorithm could be considered another kind of equality. To clarify this new algorithm and help developers avoid common issues with `==` vs `===`, I am proposing a top-level `Equality` API that names these various algorithms and provides powerful utility methods around equality.

## New Equality: "Uniform"
Some other languages allow you to compare objects and know if they are effectively identical, regardless of references. This is not trivial in Javascript; using `JSON.stringify()` to compare serialized versions of those objects is imperfect and costly.

This proposal includes an algorithm for doing a value/structure comparison between objects, with some nuances and caveats:

- `Weak*` are skipped entirely — based on their nature, this seems acceptable
- `Symbols` are coerced to `String`
- The ordering within `Object` **does not** matter
- The ordering within `Map`/`Set` **does** matter
- Circular references are skipped when encountered
- The algorithm is depth-first and returns as soon as a *difference* is found. It has various pre-checks to avoid unnecessary work, like strict-equality checking, type-checking, etc.
- Considered *less strict* than `==` since it returns `true` in more scenarios than `==` would.

## Naming
I've chosen 'feel' based names, as 'jargon' names would likely impeded learning/understanding, and 'what' names that try to convey what is included/excluded are likely to be overloaded. The names chosen are:

- `loose` for `==`  
- `strict` for `===`  
- `uniform` for `~=` (new; symbol is unnecessary). Uniform in the sense of 'one-form' and clothing — intentionally identical in value (colors) and structure (pattern) but not the same object.

## Methods
`Equality` is a new top-level object. All methods below return a `Boolean` except for those that indicate otherwise.

### Equality.loose( a, b )
Just a `return a == b` statement;
```js
Equality.loose( 0, '0' ) // true
Equality.loose( [], [] ) // false
```

### Equality.strict( a, b )
Just a `return a === b` statement;
```js
Equality.strict( 0, '0' ) // false

let a=b=[]
Equality.strict( a, b ) // true
```

### Equality.uniform( a, b, depth=Infinity )
Uses the proposed `uniform` algorithm. The depth parameter is like `Array.flat`, where for nested objects it sets the maximum depth to travel before returning a result. Depth cannot be `<1` and defaults to `Infinity`, as I imagine most would want to compare the entire object.
```js
Equality.uniform( [[0]], [[0]] ) // true
Equality.uniform( [[0]], [[1]] ) // false
Equality.uniform( [[0]], [[1]], 1 ) // true
```

### Equality.test( a, b, method:required )
Allows you to specify the equality method as an argument (string or method reference) rather than using the named method. This can be more convenient in certain situations with spread syntax and such.
```js
Equality.test( 1, '1', 'loose' ); // vs Equality.loose( 1, '1' )
```

### Equality.of( a, b, method:optional ) :: String or Boolean
If `method` is not provided, this will return the name of the most strict equality found between `a` and `b`, otherwise returns `none`.
```js
Equality.of(1,1); 				// 'strict'
Equality.of(1,'1'); 			// 'loose'
Equality.of(1,2); 				// 'none'
Equality.of([1],[1]); 			// 'uniform'
Equality.of(myObj,myObj); 		// 'strict'
```

When `method` is provided, it will assert it against the result, returning a Boolean.
```js
Equality.of(1,'1','loose'); 	// true
Equality.of(1,'1') == 'loose' 	// true, looks odd to mix Equality and == in the same line hence the assert capability
```

### Equality.none( a, b )
Equivalent to `Equality.of( a, b, Equality.none )`, to confirm `a` and `b` are not equal under any of the algorithms.
```js
Equality.none( 1, 2 ) // true
Equality.none( 1, '1' ) // false
```

### Equality.all( <Set/Array\>, method='strict' )
Returns `true` if all items are equal under the provided method. Parameter `method` can be provided via reference or name.

An alternative to the laborious `a == b && a == c && ...` or using a `Set` to consolidate.
```js
let a = 1;
let b = 1;
let c = 1;
Equality.all( [a,b,c], 'strict' ) // true; method name
Equality.all( [a,b,c], Equality.strict ) // true; method reference
```

### Equality.any( <Set/Array\>, method='strict' )
Returns `true` if at least 2 of the items are equal under the provided method.

An alternative to the laborious `a == b || a == c || ...` or using a `Set` to consolidate.
```js
let a = 1;
let b = '1';
let c = 2;
Equality.any( [a,b,c], 'loose' ) // true
Equality.any( [a,b,c], 'strict' ) // false
```

## Equality.has*
These methods let you test the key/value/entry of an `Object`, `Array`, `Map`, or `Set` using a specific equality algorithm, defaulting to 'strict'. They all return a `Boolean` as soon as possible.

**Note:** I do not think these methods should be added to the respective data types — having to provide an `Equality` method already forces you to dip into this API, might as well use it wholly.

### Equality.hasKey( target, key, method='strict' )
Returns if the target has the provided key, based on the specified equality algorithm. Since keys for `Map` can be anything, `uniform` provides a convenient way to match on identical-looking objects at the key level.

For `Set`/`Array`, keys are the indices.
```js
// Map
let myMap = new Map([ [0,1], [[2],3] ]);
Equality.hasKey( myMap, 0 ); // true
Equality.hasKey( myMap, '0', 'loose' ); // true
Equality.hasKey( myMap, [2], 'uniform' ); // true

// Object
let myObj = { '1':2 }
Equality.hasKey( myObj, 1, 'loose' ); // true; matches exp "1 in myObj"

// Set
let mySet = new Set([ 'a', 'b' ]);
Equality.hasKey( mySet, 1, 'loose' ); // true

// Array
let myArr = ['a','b'];
Equality.hasKey( myArr, 1, 'loose' ); // true; matches exp "1 in myArr"
```

### Equality.hasValue( target, value, method='strict' )
Returns if the target has the provided value, based on the specified equality algorithm. Since values for `Object`/`Map`/`Array`/`Set` can be anything, `uniform` provides a convenient way to match on identical-looking objects on the value level.
```js
// Map
let myMap = new Map([ ['a',[1]] ]);
Equality.hasValue( myMap, [1] ); // false
Equality.hasValue( myMap, [1], 'uniform' ); // true

// Object
let myObj = { 'a':[1] }
Equality.hasValue( myObj, [1], 'uniform' ); // true

// Set
let mySet = new Set([ [0],[1] ]);
Equality.hasValue( mySet, [1], 'uniform' ); // true

// Array
let myArr = [ [0],[1] ];
Equality.hasValue( myArr, [1], 'uniform' ); // true
```

### Equality.hasEntry( target, key, value, method='strict', method_value=<same as method\> )
Returns if the target has the provided key/value pair/entry, each based on the specified equality algorithm. You can provide a second equality algorithm to be used on `value` instead of using the same from `key`. Since a `Map`'s `key` and `value` can be nearly anything, `uniform` provides a convenient way to match on identical-looking objects on the key and value level.

For `Set`/`Array`, keys are the indices.
```js
// Map
let myMap = new Map([ [['a'],[1]] ]);
Equality.hasEntry( myMap, ['a'], [1], 'uniform' ); // true
```

### Equality.custom( name, function, methodToPrecede='none' )
You can register a custom equality function via unique name.

`methodToPrecede` lets you control the order of tests within `Equality.of` — specify which algorithm is 'looser' than yours to ensure your test is evaluated before that one.

The function will be given values `a`, `b` and should return a `boolean` (will be coerced). You can leverage `Equality` and its methods within the function for convenience.

You deregister a custom function by calling this method with the same name and `null` as the function parameter.

#### Value
By registering the function, it becomes invocable just like the built-in methods, including the `Equality.has*` methods. This is intended to make it easy to be consistent across the codebase by conforming to this API.
```js
// register a custom equality function
Equality.custom('combo',function( a, b ){
	// combo is order-agnostic, sort to simulate then use uniform to compare (or DIY)
	return Equality.uniform( a.toSorted(), b.toSorted() );

	// for .has* methods, a is the key/value/entry item of the target iterable
}, Equality.uniform );

// returns custom function's name
console.log( Equality.of( [0,1], [1,0] ) ); // combo

// find combo in set
let combo 	= [ 1,0 ];
let combos 	= new Set([ [0,1], [1,2] ]);
console.log( Equality.hasValue( combos, combo, Equality.combo ) ); // true

// find combo in map keys
let combos 	= new Map([ [[0,1],'a'], [[1,2],'b'] ]);
console.log( Equality.hasKey( combos, combo, Equality.combo ) ); // true
```

## Performance
### Uniform Algorithm
The `uniform` algorithm is depth-first recursive. It's not possible to know at runtime if depth-based is more appropriate than breadth-first, as the differences could be deep or wide. It does return as soon as it finds a disqualifier.

Since the algorithms are named, it would be possible to provide a variant under a modified name such as `uniform-breadth`.

### Equality.has* + Loops
In the case where a list of items needs to be tested against a collection using `uniform`, it is recommended to flatten the collection into an `Array` and use `Equality.hasValue`. The algorithm does this on invocation, thus a loop would be re-doing it unnecessarily.
```js
let flatten = myMap.keys().toArray()
let found 	= [ [0],[1],[2],[3] ].find((item)=>{
				Equality.hasValue( flatten, item, 'uniform' )
			});
```

## Benefits
- Complex `&&`/`||` statements can be replaced with more intuitive `Equality.all/any` statements. This particularly addresses the common tripping point that `a == b == c` may not evaluate as expected, whereas `Equality.all([a,b,c])` likely does.
- `==` and `===` are not objects you can easily bind to structures or storage. With this proposal, you could store the equality by reference or name and evaluate with `Equality.test()` rather than rolling your own wrappers for each.
- Some style guides may prefer the more explicit `Equality` methods as a defense against typos with `==`/`===` — it would presumably be more difficult to mix up `loose` and `strict` as words.
- Centralizing equality algorithms under an API can help ensure each new algorithm conforms to a standard (name, supported types, etc). It could also reduce pressure to give each new algorithm an equality symbol. This proposal does not contain any, but type/data-structure specific algorithms could be provided.
- `Equality.custom()` allows for mixing and matching the algorithms conditionally and to write your own algorithm without writing the overhead logic, similar to the conveniences of `Array.sort`.
- The [composites](https://github.com/tc39/proposal-composites) proposal addresses the common use-case of wanting to check an object's membership in a collection, not by reference but when the object is semi/identical to one in the collection. For example, knowing that the structure `[0,1]` is in `[[0,1],[1,2],[2,3]]` regardless of references. This proposal solutions that common case out of the box, and `Equality.custom` supports variation:
```js
Equality.hasValue([[0,1],[1,2],[2,3]], [0,1], 'uniform') // true
```