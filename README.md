# Abstract References for ECMAScript #

## Overview and Motivation ##

Despite its functional programming facilities, ECMAScript exhibits a strong preference
for object-oriented "left-to-right" composition using instance methods.  Unfortunately,
composition by means of instance methods requires that extensibility be achieved by adding
function-valued properties to the object or a prototype ancestor of the object.  This
leads to the following difficulties:

- The API burden for an abstraction is placed upon the abstraction itself, either directly or
  by the introduction of inheritance hierarchies.  This leads to an undesirable centralization
  of API surface.

- Adding API surface to an existing abstraction introduces the possibility of breakage which
  can be difficult to predict.

- By adding API surface directly to the object or one of its prototype ancestors, it is
  impossible to treat the object and the API as different capabilities.  If a user has access
  to the object, then the user necessarily has access to the full API (part of which we
  might want to hide).

This proposal adds support for "left-to-right" syntactic composition using a new binary operator,
which does not require adding properties and methods to the object itself.  It accomplishes this
by introducing the concept of "abstract references".  Abstract references are an extension of the
Reference specification type whose *referenced name* component may be an arbitrary object.

This proposal also provides a solution to the "private fields" problem.

This work was inspired by [relationships](http://wiki.ecmascript.org/doku.php?id=strawman:relationships)
and is intended to supersede it.


## Specification ##

We introduce three new built-in symbols:

- **@@referenceGet** (Symbol.referenceGet)
- **@@referenceSet** (Symbol.referenceSet)
- **@@referenceDelete** (Symbol.referenceDelete)

We introduce a new abstract operation:

- **IsAbstractReference(V)**.  Returns **true** if the *referenced name* component of the reference *V*
  is neither a primitive String nor a Symbol.

The abstract operation **GetValue** becomes:

- ReturnIfAbrupt(*V*).
- If Type(*V*) is not Reference, return *V*.
- Let *base* be GetBase(*V*).
- If IsUnresolvableReference(*V*), throw a **ReferenceError** exception.
- If IsPropertyReference(*V*), then
  - If HasPrimitiveBase(*V*) is true, then
    - Assert: In this case, *base* will never be null or undefined.
    - Let *base* be ToObject(*base*).
  - If IsAbstractReference(*V*) is **true**, then
    - Let *nameObject* be GetReferencedName(*V*)
    - Return the result of Invoke(*nameObject*, **@@referenceGet**, (*base*))
  - Else
    - Return the result of calling the [[Get]] internal method of *base* passing GetReferencedName(*V*)
      and GetThisValue(*V*) as the arguments.
- Else *base* must be an environment record,
  - Return the result of calling the GetBindingValue concrete method of *base* passing
    GetReferencedName(*V*) and IsStrictReference(*V*) as arguments.

The abstract operation **PutValue** becomes:

- ReturnIfAbrupt(*V*).
- ReturnIfAbrupt(*W*).
- If Type(*V*) is not Reference, throw a **ReferenceError** exception.
- Let *base* be GetBase(*V*).
- If IsUnresolvableReference(*V*), then
  - If IsStrictReference(*V*) is true, then
    - Throw **ReferenceError** exception.
  - Let *globalObj* be the result of the abstract operation GetGlobalObject.
  - Return Put(*globalObj*, GetReferencedName(*V*), *W*, **false**).
- Else if IsPropertyReference(*V*), then
  - If HasPrimitiveBase(*V*) is true, then
    - Assert: In this case, *base* will never be **null** or **undefined**.
    - Set *base* to ToObject(*base*).
  - If IsAbstractReference(*V*), then
    - Let *nameObject* be GetReferencedName(*V*)
    - Let *result* be Invoke(*nameObject*, **@@referenceSet**, (*base*, *W*))
    - ReturnIfAbrupt(*result*)
  - Else
    - Let *succeeded* be the result of calling the [[Set]] internal method of *base* passing
      GetReferencedName(*V*), *W*, and GetThisValue(*V*) as arguments.
    - ReturnIfAbrupt(*succeeded*).
    - If *succeeded* is **false** and IsStrictReference(*V*) is **true**, then throw a **TypeError** exception.
  - Return.
- Else *base* must be a Reference whose base is an environment record.
  - Return the result of calling the SetMutableBinding concrete method of *base*,
    passing GetReferencedName(*V*), *W*, and IsStrictReference(*V*) as arguments.

The runtime semantics of the delete operator is modified as follows:

```
UnaryExpression : delete UnaryExpression
```

- Let *ref* be the result of evaluating *UnaryExpression*.
- ReturnIfAbrupt(*ref*).
- If Type(*ref*) is not Reference, return **true**.
- If IsUnresolvableReference(*ref*) is **true**, then
  - Assert: IsStrictReference(*ref*) is **false**.
  - Return **true**.
- If IsPropertyReference(*ref*) is **true**, then
    - If IsSuperReference(*ref*), then throw a **ReferenceError** exception.
    - Let *base* be ToObject(GetBase(*ref*))
    - If IsAbstractReference(*ref*) is **true**, then
      - Let *nameObject* be GetReferencedName(*V*)
      - Let *result* be Invoke(*nameObject*, **@@referenceDelete**, (*base*))
      - Return **true**.
    - Else
      - Let *deleteStatus* be the result of calling the [[Delete]] internal method on
        *base*, providing GetReferencedName(*ref*) as the argument.
      - ReturnIfAbrupt(*deleteStatus*).
      - If *deleteStatus* is **false** and IsStrictReference(*ref*) is **true**, then throw a **TypeError**
        exception.
      - Return *deleteStatus*.
- Else *ref* is a Reference to an Environment Record binding,
  - Let *bindings* be GetBase(*ref*).
  - Return the result of calling the DeleteBinding concrete method of *bindings*, providing
    GetReferencedName(*ref*) as the argument.

The only way to create a reference whose *referenced name* is neither a String nor a Symbol is
by using the "abstract reference" operator:

```
MemberExpression[Yield] :
    MemberExpression[?Yield] :: IdentifierReference[?Yield]
```

- Let *baseReference* be the result of evaluating *MemberExpression*.
- Let *baseValue* be GetValue(*baseReference*).
- ReturnIfAbrupt(*baseValue*).
- Let *nameValue* be the result of evaluating *IdentifierReference*.
- ReturnIfAbrupt(*nameValue*).
- Let *bv* be RequireObjectCoercible(*baseValue*).
- ReturnIfAbrupt(*bv*).
- If the code matched by the syntactic production that is being evaluated is strict mode code,
  let *strict* be **true**, else let *strict* be **false**.
- Return a value of type Reference whose base value is *bv* and whose referenced name is
  *nameValue*, and whose strict reference flag is *strict*.

## Extensions to Built-In Types ##

The built-in types are extended as follows:

*The methods below are implemented in ECMAScript for convenience.*

```js
Function.prototype[Symbol.referenceGet] = function(base) { return this };
```

```js
function mapGetInherited(base) {

    while (base !== null) {

        if (this.has(base))
            return this.get(base);

        base = Object.getPrototypeOf(base);
    }

    return void 0;
}

Map.prototype[Symbol.referenceGet] = mapGetInherited;
Map.prototype[Symbol.referenceSet] = Map.prototype.set;
Map.prototype[Symbol.referenceDelete] = Map.prototype.delete;

WeakMap.prototype[Symbol.referenceGet] = mapGetInherited;
WeakMap.prototype[Symbol.referenceSet] = WeakMap.prototype.set;
WeakMap.prototype[Symbol.referenceDelete] = WeakMap.prototype.delete;
```


## Examples ##

Using an iterator library implemented as a module:

```js
import { map, takeWhile, forEach } from "iterlib";

getPlayers()
::map(x => x.character())
::takeWhile(x => x.strength > 100)
::forEach(x => console.log(x));
```

Using WeakMaps to implement private fields:


```js
const _x = new WeakMap,
      _y = new WeakMap;

class Point {

    constructor(x, y) {

        this::_x = x;
        this::_y = y;
    }

    toString() {

        return `[${ this::_x },${ this::_y }]`;
    }
}
```

