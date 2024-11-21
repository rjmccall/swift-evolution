# A Prospective Vision for Accessors in Swift

Swift properties and subscripts can be implemented by providing one or more "accessors" that retrieve or update the value. Most Swift developers are familiar with the `get` and `set` accessors that are used to define computed properties:

```swift
struct Foo {
  var value: Int {
    get { ... provide a value ... }
    set { ... accept a value ... }
  }
}
```

In this case, the `get` accessor behaves just like a normal method that returns a value of the property's type, while the `set` accessor behaves just a normal method that receives a value of the property's type as an argument.

The `get` and `set` accessors are ideal for implementing operations that copy the current value of the property:

```swift
let copy = myFoo.value  // calls the get accessor for Foo.value
```

or that overwrite the current value of the property:

```swift
myFoo.value = 51        // calls the set accessor for Foo.value
```

Other kinds of operations can also be compiled in terms of `get` and `set`. For example, if you pass a computed property as an `inout` argument:

```swift
myFoo.value += 10
```

Swift will use `get` to initialize a temporary variable, pass that variable as the argument, and then write the new value back with `set`:

```swift
var tmp = myFoo.value   // calls the get accessor for Foo.value
tmp += 10
myFoo.value = tmp       // calls the set accessor for Foo.value
```

However, this approach has significant problems. The biggest is that the `get` accessor has to return an independent value. If the accessor is just returning a value stored in memory, which is very common for data structures, this means the value has to be copied. This is unfortunate on three levels:

1. It adds the runtime performance and memory overhead of copying the inline representation of the value. For example, if the value is an `Array`, the internal buffer of the array must be retained.

2. It can make subsequent uses of the value less efficient. For example, if the value uses a copy-on-write representation like `Array` and `String` do, mutating a copy is likely to dramatically less efficient than mutating a variable in place. (We will explain this in more detail later.)

3. It requires the value to be copyable at all, and so it inherently cannot work for values of non-`Copyable` type.

These problems are amplified when properties and subscripts need to be abstracted over. When Swift knows exactly how a declaration is implemented, the compiler can access it in the best way possible given the implementation. For example, if Swift can see that a property is stored, it can emit code to directly access that memory instead of calling an accessor. However, when Swift doesn't know how the declaration is implemented, it must call some kind of accessor instead, and so it is limited by the capabilities of that accessor.

This kind of abstraction is necessary in several common situations:

- when the declaration is being accessed through a protocol requirement,
- when the declaration is a non-`final` member of a class, or
- when the declaration is from a different library that's been built with library evolution enabled.

For example, suppose we have this code:

```swift
struct Person: Nameable {
  var name: String
}

protocol Nameable {
  var name: String { get }
}

printNameConcretely(_ person: Person) {
  print(person.name)
}

printNameGenerically(_ person: any Nameable) {
  print(person.name)
}
```

In `printNameConcretely`, Swift knows that `name` is a stored property of `Person`, and it can just load that value directly from `person` and pass it to `print`. In `printNameGenerically`, Swift does not know how `name` is implemented, and it must call a `get` accessor to copy the current value of the name. To avoid those costs, the Swift optimizer would have to specialize this function for the specific type that is being passed in; this is something that Swift can and does do, but only as a best-effort optimization, which is not always good enough. And, of course, this code would be ill-formed if `String` were a non-`Copyable` type, because the only way to satisfy a `get` requirement for a stored property is to copy the current value.

As a result, Swift has explored a variety of other accessors throughout its history, none of which have ever been officially added to the language through the Swift Evolution process. (The observing accessors, `willSet` and `didSet`, are officially in the language but are arguably in a different category because they don't serve as complete operations.) Many of these have been adopted in the standard library for years, but we've been reluctant to make them official because they are variously incomplete, unsafe, or complex.

This vision document lays out the design space of accessors for the next few years, as Swift continues to advance its support for non-`Copyable` and non-`Escapable` types. It explains Swift's basic access model and how it may need to evolve. It explores what developers need from accessors in these advanced situations. Finally, it discusses different kinds of accessors, both existing and under consideration, and how they do or not fit into the future of the language as we see it.

This is a prospective vision which has not yet been reviewed by the Language Steering Group. Even if it is approved by the Language Steering Group in exactly this form, it is merely laying out a high-level vision for the language design and does not constitute pre-approval of any specific ideas in this document. Everything in this document will need to be separately proposed and reviewed under the normal Swift Evolution process before it is part of the Swift language.

## Swift's access model

Swift's access model was first described during the development of Swift 4 in the [ownership manifesto](https://github.com/swiftlang/swift/blob/main/docs/OwnershipManifesto.md). This section will repeat that model using the conventional terminology, but with some clarifications and notes.

Swift provides two kinds of *storage declaration*: `var`s and `subscript`s. From this perspective, `let` is just a special kind of `var` which cannot be modified and cannot have accessors. A property is just a special name for a `var` or `let` that's defined as a member of a type. `var`s and `subscript`s are referenced very differently in the syntax, but semantically they work very similarly, especially with respect to accessors.

A reference to a storage declaration is a *storage reference expression*. A storage reference expression for a `var` or `let` is just the name of the declaration, either standalone (e.g. `value`) or as a member of a base expression (e.g. `base.value`. A storage reference expression for a `subscript` is the indexing operator `[ ... ]` applied to a base expression with any appropriate index expressions (e.g. `base[i]`).

Every storage reference expression performs a specific kind of access, which is determined contextually from how the expression is used. Historically, we have said that there are three kinds of access: *reads*, *assignments*, and *modifications*.

A read access conceptually means that the current value of the storage is being read without changing it. A storage reference expression that is written in any position other than those below is a read access. For example, in `print(x)`, there is a read access to the storage reference expression `x`.

An assignment (or write) access conceptually means that the storage is being completely overwritten with a new value. It primarily occurs when the expression is the direct target of the `=` assignment operator. For example, in `base.value = 10`, there is a write access to the storage reference expression `base.value`.

A modification (or update) access conceptually means that the storage is being both read and written. It occurs when:
- a storage reference expression is passed as an `inout` argument, including as the left operand of a compound assignment operator like `+=`; or
- a storage reference expression is the base expression of another storage reference expression that must mutate its base in order to perform its own requested access. For example, in `base.value = 10`, the access to `base` is:
  - a modification if `value` is a stored property of a value type,
  - a modification if `value` is defined with a `mutating set`,
  - a read if `value` is a stored property of a reference type, or
  - a read if `value` is defined with a `nonmutating set`.

The old manifesto's description of modification and assignment accesses has stood up to time, but its description of read accesses arguably has not. As more advanced ownership features have developed in the language, Swift has increasingly needed to distinguish at least two kinds of reads:

A *copying read access* conceptually means that the current value of the storage is being copied (or consumed) to produce an independent value. At minimum, it occurs when the storage reference expression appears in any context where an independent value is required, such as a return value or as the right operand of the `=` operator.

A *borrowing read access* conceptually means that the current value of the storage is being temporarily borrowed in order to read it without copying it. At minimum, it occurs when the storage reference expression is appears in any context where an implicit copy is not allowed, such as passing it as a `borrowing` argument when the value is non-`Copyable`.

The semantic and implementation-level differences between borrowing and copying/consuming uses will be very important in this document.

It is reasonable to ask whether Swift could categorize *all* read accesses into these two kinds based on context, the same way that it distinguishes reads from assignments. This would be straightforward at a technical level, but it is controversial as a design direction because the most obvious definitions would be very aggressive about borrowing values. This could cause surprising semantic problems for Swift programmers, especially those working extensively with classes, and it could break the behavior of existing code. This remains an open question that this vision does not take a stand on.

## Two Dimensions of Access

To summarize the previous section, there are four basic kinds of access:

- copying reads (producing an independent value equivalent to the current value)
- borrowing reads (allowing temporary use of the current value)
- modifications (updating the current value in place)
- assignments (replacing the current value with a new, independent value)

We can implement accesses to storage in a wide variety of ways, and each of them could give rise to a different kind of accessor. It's reasonable to look for a general pattern that all accessors of a particular access kind would have to follow. To do this, we can observe that every access can be split into three phases:
1. The implementation does something to set up for the access.
2. The client code does whatever it's going to do during the access.
3. The implementation possibly does something to finalize the access.

For example, if we call a `mutating` method on a stored property of a `struct`:
1. The "implementation" of the property adds the correct offset to the base address of the struct to derive the address of the stored property.
2. The client code passes that address as `self` to the `mutating` method.
3. Nothing has to be done dynamically to finalize the access.

Similarly, if we call a `mutating` method on an element of an `Array`:
1. The implementation of `Array.subscript` ensures that the array buffer is uniquely referenced, then derives the address of the element.
2. The client code passes that address as `self` to the `mutating` method.
3. Nothing has to be done dynamically to finalize the access.

These two examples have a common characteristic: the implementations do not need to do any dynamic finalization of the access. This turns out to be very important for practical purposes.

For one, it is very common when using value semantics. If a storage declaration is implemented by just keeping the value somewhere in memory, and exclusivity for that memory is guaranteed statically by the exclusivity of the containing value, there's nothing to do as finalization. This covers stored properties of value types and the vast majority of data structures.

But it's also really useful because it frees the client of the burden of running code when the access ends. The general pattern that we described above has to run in two distinct phases. That makes it inherently something like a coroutine: it's going to be split into multiple functions, and it might have to dynamically allocate memory to pass between them. The client must then keep all of this information around dynamically for each access it performs. If the client starts a dynamic number of accesses at once, it will need dynamic allocation to remember all of the active accesses. In contrast, if finalization is guaranteed to be trivial, the implementation of the access can work more like a normal function: the function will just perform the first phase and pass back whatever it needs to as a return value. The client has nothing to track, so it can manage even a dynamic number of accesses completely statically.

## Six of Eight Fundamental Accessors

So we have four kinds of access, and we can classify accessor implementations by whether they need to be coroutines. That gives us a 2x4 matrix of possible accessors. Two of these combinations, however, turn out to not be useful:

|              |Copying read |Borrowing read |Modification|Assignment |
|---           |---          |---            |---         |---        |
|**Routine**   |             |               |            |           |
|**Coroutine** |✗            |               |            |✗          |

A copying read access has to generate an independent value during the first phase. Given that, it's unclear what a copying read accessor could possibly need to do during finalization. If there's some sort of cleanup it has to do, and after the cleanup the value shouldn't be used, then the value wasn't really independent in the first place.

An assignment access just consumes an independent value. Just like above, it's unclear what an assignment accessor could possibly need to do that would require it to be split into multiple phases.

## Existing and Proposed Standard Accessors

Based on the discussion above, we are left with six combinations that are both possible and useful.  The following table gives each one a name:

|              |Copying read |Borrowing read |Modification|Assignment |
|---           |---          |---            |---         |---        |
|**Routine**   |get          |borrow         |mutate      |set        |
|**Coroutine** |✗            |read           |modify      |✗          |

Some of the above exist in the current Swift language, others we expect to be proposed in the near future.  Here is a slightly more detailed explanation of each one:

* `get` and `set` - These are the original standard accessors. They are effectively just normal functions that return or accept an independent value of the value type.
* `borrow` - This accessor is not yet implemented, but we expect to propose it for Swift Evolution at some point in the future. It is effectively just a normal function that returns a borrowed value, using Swift's existing representation that avoids pointer overhead for simple types. Swift will have to prove that the returned value can be safely borrowed without finalization and with the right lifetime.
* `mutate` - This accessor is not yet implemented, but we expected to propose it for Swift Evolution at some point in the future. In various discussions, it has sometimes been called `inout`. It is effectively just a normal function that returns the address of a mutable value. Swift will have to prove that the returned address has the right lifetime.
* `read` and `modify` -  These exist today as experimental `_read` and `_modify` implementations, and we expect to propose a revised final form without the underscored names in the near future. They compile into coroutines that “yield” the value. In effect, each such accessor becomes two functions, each performing one phase of the implementation as discussed above.

In the process of developing the above, a number of other approaches have been explored.

* `unsafeAddress` and `unsafeMutableAddress` - These are implemented but are not expected to ever go through Swift Evolution. They compile into methods that return a pointer to the value in question. This pointer value is explicit in the property implementation but is not visible in the Swift source code of the client. This does not require copying the value, but does require that the value already be present somewhere in memory.  This was an early form of the idea behind `borrow` and `mutate`, but it is based on unsafe constructs.
* `unsafeRawAddress` and `unsafeRawMutableAddress` - These are not yet implemented and may not ever go through Swift Evolution.  These are similar to the above but allow the property implementation to work directly with an untyped “raw” pointer while the caller sees an access to typed data in memory.  This document will refer to these and the previous two collectively as `unsafe*Address`.
* `_read` and `_modify` - These were the early experimental forms of `read` and `modify`.  They will likely continue to be supported in order to avoid breaking existing code, but users should migrate to `read` and `modify` once they are finalized.

## Distinguishing Features

These accessors vary in how they treat both the property value and the containing value.  In this section, we’ll explore the forms that variation takes:

### Copying vs. Borrowing

A key distinction is whether a particular accessor copies the property value or whether it provides access to the value without copying it.  The `get` and `set` accessors copy the value by returning it as a method result or accepting it as a method argument.  Other accessors provide access to a value without copying:

*  `unsafe*Address` have the accessor provide an explicit pointer
* `read`/`modify` *yield* access to a value from a coroutine.  This can be a directly stored value, or a constructed value stored temporarily in a coroutine execution frame.
* `borrow` and `mutate` allow the client to *borrow* the value.  Following the existing Swift ABI rules, this may involve passing a pointer or “bitwise borrowing” the value to avoid pointer overhead for small values.  (Note that bitwise borrowing can be used even for noncopyable values since it is not formally a copy.)

The `unsafe*Address` and `borrow`/`mutate` can avoid copying because the property value is already in memory within the containing value.  The compiler needs to ensure that the containing value is not destroyed or modified until after the pointer or borrow reference is no longer in use.

A `read` or `modify` accessor can usually avoid a copy, but not always:  In practice, the implementation cannot delay the bottom half of the coroutine indefinitely.  This can lead to situations where the value is still needed after the coroutine ends, which requires copying the value from the coroutine’s execution frame into the calling context.  When this happens, a `_read` or `_modify` accessor is generally slower than a regular `get`, since it incurs copy overhead similar to a `get` in addition to the coroutine execution overhead.  This also limits the use of `read` or `modify` with property values that are not copyable.

### Exposing transformed temporary values

The `read` and `modify` accessors provide the ability to expose a transformed version of a stored value. This is important for dictionary mutation, which in the current API exposes the value as an Optional which is `nil` if the value is not currently set.  This allows a dictionary update such as `dict[key]?.modify()` to copy the dictionary value into a constructed optional in the top half of the accessor, then run the method call directly on the value, then run the bottom half to deconstruct the optional and store the result back into the dictionary storage.

Another example:  Providing a `Span` over the contents of a String will require that we somehow handle the case where a short String is stored inline.

### Access Scope

The `get`/`set` accessors conceptually represent “instantaneous” access of the value.  After the `get` accessor returns, there is no relation between the returned copy of the property value and the containing value.  Each of the other accessors implies that the containing value must continue to exist for some period.  This period is generally fairly short, but optimization can extend this interval in order to simplify other operations:

* `unsafe*Address` accessors return pointers into the containing value.  This is safe for the caller of these accessors because the compiler knows about this relationship and can extend the lifetime of the containing value as needed.  (These accessors are nominally “unsafe” because their implementation requires constructing an unsafe pointer and there are no checks to ensure that the pointer so constructed is in fact valid.  For example, there is no check on the pointee lifetime.))
* `read`/`modify` expose a value for the lifetime of a coroutine. Coroutines enforce that the containing value remains alive for the duration of the coroutine.  Note that in current Swift, the coroutines are generally quite short-lived and the compiler copies the value into or out of the coroutine fairly aggressively.  In the future, the compiler will likely become more adept at expanding the coroutine lifetime to reduce such copying.
* `borrow`/`mutate` return a borrow of the property value.  This borrow has an implied dependence on the containing value, and the compiler must guarantee that the containing value outlives the property access.

### Ownership of the containing value

All of the above accessors include a reading variant (`get`, `read`, `borrow`) and a writing variant (`set`, `modify`, `mutate`).  By default, the reading variant is not considered to be a mutation of the enclosing value.  The writing variant conversely is considered to be a write access of the enclosing value and the usual Swift exclusivity rules apply to limit the scope of such access.

However, a reading accessor can be explicitly marked as `mutating`.  This relaxes checks on the accessor implementation so that it can mutate the containing value, and also causes the caller to treat this as a write access requiring exclusivity enforcement.  This is commonly used for caching or bookkeeping:

```swift
struct Foo {
  private var accessCount: Int
  private var _cachedBaz: Baz?
  var baz: Baz {
     mutating get {
       accessCount += 1 // OK because of `mutating`
       if _cachedBaz == nil {
         _cachedBaz = computeBazPropertyValue() // OK because of `mutating`
       }
       return _cachedBaz
     }
  }
}
```

Conversely, writing accessors can be marked as `nonmutating`, though this is not generally useful in practice as it would require that the accessor not in fact mutate the containing value.  This could only be useful for modeling non-property operations as setters.

Accessors can also be `consuming`.  This indicates that the containing object must end its lifetime once this access is complete.  This is useful for representing certain types of object transformations:

```swift
struct Values {
  var dropOldest: Values {
    consuming get {
       let newValues = ... everything but the oldest ...
       return newValues
       // Current value will end it's lifetime after this returns
    }
  }
}

let v = Values()
let v2 = v.dropOldest
// `v` is no longer alive here
```

Note that `consuming` makes no sense for setters — there is no point to changing a property on a value and then immediately ending the lifetime of that value.  Similarly, `consuming` makes no sense for `borrow` or `unsafeAddress` accessors — those require that the containing value survive for the duration of the returned borrow access or address, so it does not make sense to explicitly terminate the value lifetime immediately.

This only leaves `consuming get` and `consuming read` as meaningful combinations.  A `consuming get` can be used for operations such as the `dropOldest` example above that model a transformation by creating a new value and terminating the old one.  The `consuming read` variant can be used similarly.  Unlike `borrow`, the `read` operation coroutine structure forces the containing value to live for a certain period of time, the value is consumed only at the end of that coroutine.

## Some Observations

Based on the above considerations, we can make a few useful observations about how these different accessors complement one another:

#### Diversity is needed

No one pair of accessors suffices to cover the needs of the language:

* `get`/`set` require the property value to be copyable.
* `unsafeAddress`/`unsafeMutableAddress` and `borrow`/`mutate` require the value to be already available in memory.  In particular, the mutating forms cannot be used to store a new value into a collection such as `Dictionary`, since the storage for such a value would not already be initialized.
* `read`/`modify` exposes the value only for the lifetime of a coroutine, which is itself limited by implementation restrictions.

#### Some combinations are nonsensical

* Providing both a `mutate` and a `modify` for the same property doesn’t make sense.  If `mutate` is possible, then you can provide the value without making a copy, which means `modify` is unnecessary and redundant.

#### We need to support existing APIs

The current Swift API seems to require the following:

* `set` is needed to be able to initialize new values for subscript operations.  `modify` is capable of doing this for types that can be trivially initialized, but there can be considerable overhead to construct an empty placeholder, expose it for mutation, and then record the final result.
* Either `borrow` or `read` is needed to provide safe reading of noncopyable values
* Either `mutate` or `modify` is needed to provide safe mutation of noncopyable values
* We have `Dictionary` APIs (among others) that expose values as a different type than they are stored.  These can be supported using `read`/`modify` and storing the transformed value in the coroutine frame for the duration of the access.  They could also be supported using `get` to return a transformed value, but that would require a more expensive read-modify-write cycle for modifications.
* Because `read`/`modify` have constraints on the lifetime of the access, they cannot fully replace `borrow`/`mutate`.

#### Address accessors are redundant with borrow/mutate

The `unsafe*Address` accessors should be unnecessary once we have safe `borrow` and `mutate` to replace them.   But it may take a while to implement the latter, so the former are important interim tools for now.

#### Minimal sets of accessors

The above considerations allow us to outline a possible minimal set of accessors:

* The `borrow`, `mutate`, and `set` accessors provide the absolute minimum required for performant property access.
* In addition, there are a few APIs (especially Dictionary) that expose transformed values; those require `modify` for performant update operations.
* Providing a freshly-constructed value on each access requires `get`.

#### Roadmap

Combining the above, we expect that Swift will converge on a standard set of six accessors — `get`, `set`, `read`, `modify`, `borrow`, and `mutate` — over the next couple of years.

In the interim, the `unsafe*Address` accessors will continue to be needed and useful for implementing many types of containers.

The legacy `_read` and `_modify` accessors will continue to be supported by the compiler for ABI compatibility but should not be used by new code.

## Recommended Usage

Based on the above, we can make some concrete recommendations for how each of these accessors should be used.

#### `get`

This should be used when the access constructs and returns a new value or is part of a resilient interface which might be changed to operate this way at some point in the future.  Note that this works correctly even for non-copyable types when the value truly is returned fresh from each access and not being stored locally.

Use `mutating get` when you need to implement a `get` operation that might cache the value or otherwise update mutable state on the containing object.

Use `consuming get` to model transmutation operations, where the original instance is being transformed into some other type, dissolving it in the process. One use case of this is unwrapping box types, where the returned instance is extracted from the box, destroying it in the process.

#### `set`

As above, this is the only reasonable way to implement subscript operations that create new entries.  It can be used with copyable or noncopyable property values.

#### `unsafe*Address`

These are not recommended for general use, as they require working with unsafe pointer types.  However, until `borrow`/`mutate` can be implemented, they will be the only good choice for containers that need to flexibly provide access to non-copyable values.

```swift
struct MyContainer<T> {
  subscript(i: Int) -> T {
    unsafeAddress {
      baseAddress! + i * MemoryLayout<T>.stride
    }
  }  
}

let contents = MyContainer<Foo>
let x = contents[0]

```

> Important:  The above example uses unsafe pointer operations in the `subscript` implementation, but the use of pointers is completely invisible to the client source code.

#### `read`

> This is currently implemented as `_read` with slightly different semantics than described here.  In particular `_read` will not always run the “bottom half” if there is a thrown error in the caller during the scope of the coroutine.


Read accessors expose direct access to a value, allowing that value to be created on demand for the duration of the access, and destroyed at the end of it. The caller is only given borrow access to the entity — it is not allowed to mutate or consume it.

The result is scoped to a narrow, unique lifetime tied to the specific access, rather than the instance on which the property was invoked.  Because it is implemented as a scoped coroutine, this lifetime cannot extend past the end of an accessing function:

```swift
extension Foo {
  var noncopyable: NonCopyableType {
    read {
      let temp = ... transform existing value to construct `temp` ...
      yield temp
      ... clean up `temp`, possibly restoring the original value ...
    }
  }
}

func access(foo: Foo) -> NonCopyableType {
  // Coroutine starts before accessing the property.
  // The coroutine `yield` provides this code with
  // a reference to the temp value stored in the
  // coroutine execution frame.
  return foo.noncopyable
  // Note: If this function were to do more with the value,
  // then the end of the coroutine can be delayed until
  // the last operation.
  // But note that coroutines _must_ end before the
  // function does, so the return in this case must
  // be transferred out of the coroutine frame.
  // Problem: coroutine still owns the temp value,
  // so it can't be transferred here without copying
}
```

If the example above were implemented as a `get` operation, we could return the constructed non-copyable temporary value  but that would not give the `Foo` object any opportunity to clean up the temporary.

#### `modify`

When a property needs to be exposed for mutation with a fundamentally different type than is being stored, `modify` is the only practical choice.

#### `borrow`

When it becomes available, `borrow` will be the preferred way to provide read access to values that are either noncopyable or expensive to copy (for example, because they are large).  This includes values with generic type that might be either noncopyable or expensive to copy.

Compare how the above example would work with a `borrow` implementation:

```swift
extension Foo {
  var noncopyable: borrowing NonCopyableType {
    borrow {
      return ... value stored in some structure ...
    }
  }
}

func access(foo: Foo) -> borrowing NonCopyableType {
  return foo.noncopyable
}
```

Unlike the `read` version, this form has no problems returning the borrowed value.  This requires a new “borrowing returns” language feature which would track the reliance on `foo` and ensure that `foo` was not modified for as long as the value borrowed from it was being accessed.

Compared to `unsafeAddress`, `borrow` provides the same functionality without the need to do any explicit pointer manipulation.  The subscript implementation here is written as if it returned the value, but the implementation only uses safe constructs:

```swift
struct MyContainer<T> {
  subscript(i: Int) -> borrowing T {
    borrow {
      return innerContainer[i]
    }
  }  
}
```

> Note:  Swift’s implementation of borrowing uses “bitwise borrowing” whenever that would be more efficient than accessing through a pointer.  Formally, “bitwise borrowing” works by invalidating the original value, sharing a copy of that value, then resuscitating the original value when the copy is no longer in use.  Since “invalidating the original value” and “resuscitating the original value” are no-ops at runtime, this can provide the same functionality as “borrow by pointer” while ensuring that borrowing is never less efficient than copying.

#### `mutate`

A `mutate` accessor can be thought of as a “reverse `inout`”.  In fact, some discussions have advocated using the term “inout” for this accessor.  As with `read`, this requires a new return capability that internally returns a reference to the stored value and tracks the lifetime of that reference against the lifetime of the containing value:

```swift
extension Foo {
  var noncopyable: mutating NonCopyableType {
    mutate {
      return ... value stored in some structure ...
    }
  }
}

func access(foo: Foo) -> mutating NonCopyableType {
  return foo.noncopyable
}
```

## Status and Next Steps

The current status of the accessors described above is:

* `get`/`set` are fully supported, standard parts of the Swift language.
* `_read` / `_modify` have been implemented in the compiler for some time as an experimental form of `read`/`modify`.  The underscored forms will continue to be supported in the compiler for the foreseeable future until all existing users have migrated.
* `read`/`modify` will be the final standard form.  We expect to have these implemented in the compiler and ready for Swift Evolution review in the coming months.
* `unsafeAddress`, `unsafeMutableAddress` are implemented in the compiler today (with some limitations).  We do not expect to ever submit them for Swift Evolution review.
* We hope to have experimental implementations of `borrow` and `mutate` in another year or two and submit them for Swift Evolution at that time.

There are a number of open questions that will need to be resolved in the process of implementing the above:

* Protocols can specify `get` and `set` requirements for properties.  We will want them to be able to specify other accessor types as well.
* The current experimental `_read` and `_modify` implementation has some performance problems with certain types of generic code.  We will want to resolve this for the final `read` and `modify` implementation.
* The `borrow` and `mutate` accessor implementations will require new return value conventions to be fully useful.

Of course, everything in this document is subject to community review through the Swift Evolution process.
