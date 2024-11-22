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

The *value type* of a storage declaration is the type of the value it presents to clients. In a `var` or `let`, it is the declared (or inferred) type of the variable. Reference ownership modifiers like `weak` and `unowned` are not part of the value type. In a `subscript`, it is the "return type" of the subscript's signature. Subscript indices must be provided in order to use the subscript, and they are available to the accessors that implement it, but they have no special role in how accesses work in the language.

A reference to a storage declaration is a *storage reference expression*. A storage reference expression for a `var` or `let` is just the name of the declaration, either standalone (e.g. `value`) or as a member of a base expression (e.g. `base.value`. A storage reference expression for a `subscript` is the indexing operator `[ ... ]` applied to a base expression with any appropriate index expressions (e.g. `base[i]`).

Every storage reference expression performs a specific kind of access, which is determined contextually from how the expression is used. Historically, we have said that there are three kinds of access: *reads*, *assignments*, and *modifications*.

A read access conceptually means that the current value of the storage is being read without changing it. A storage reference expression that is written in any position other than those below is a read access. For example, in `print(x)`, there is a read access to the storage reference expression `x`.

An assignment (or write) access conceptually means that the storage is being completely overwritten with a new value. It primarily occurs when the expression is the direct target of the `=` assignment operator. For example, in `base.value = 10`, there is a write access to the storage reference expression `base.value`.

A modification (or update, or read-write) access conceptually means that the storage is being both read and written. It occurs when:
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

## Two Dimensions of Accessors

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

Some of the above exist in the current Swift language, others we expect to be proposed in the near future.

### `get`

The `get` accessor is the natural most-general model of a copying read access. It is an ordinary function that returns an independent value that is conceptually a copy of the current value of the storage.

In principle, since a function can return a non-`Copyable` type, there's no reason a storage declaration with a non-`Copyable` value type couldn't provide a `get` accessor. In practice, however, this is generally either impossible or at least undesirably expensive. Non-`Copyable` types are typically non-`Copyable` because they represent some kind of unique ownership that cannot be duplicated without changing the meaning. These storage declarations generally cannot support a copying read access under any implementation.

### `set`

The `set` accessor is the natural most-general model of an assignment access. It is an ordinary function that takes a new independent value as an argument and conceptually replaces the current value of the storage with it. The value is taken as a `consuming` argument, so the `set` accessor generalizes perfectly well to non-`Copyable` value types.

When combined with an implementation of a copying read access (a `get`), `set` can be used to synthesize an implementation of a modification access:

```swift
var temporary = get()  // copy the current value of the storage
temporary.mutate()     // allow the client to do its modification
set(consume temporary) // replace the current value of the storage
```

This synthesized modification access requires a coroutine model because the call to `set` is a non-trivial finalization.

Because this synthesis requires a copying read access, it generally doesn'y work for non-`Copyable` value types. (It can work if the storage declaration provides a `get`, but such an accessor usually can't be defined unless it's `consuming`, which would prevent the `set` from being called later.) To support modification in this case, a storage declaration must generally also define one of the modification accessors.

A storage declaration can usefully define both a `set` accessor and a modification accessor. Assignment accesses will just call `set`, allowing them to bypass any overhead that might be associated with reading the current value. Modification accesses will ignore the `set` and just use the modification accessor. However, if there isn't any overhead for reading the current value --- for example, if the value is already stored in memory somewhere --- then this is unlikely to be a useful optimization over just defining a non-coroutine modification accessor.

### `read` and `borrow`

The `read` accessor is the natural most-general model of a borrowing read access. It is a coroutine function which yields a borrowed value and can then do arbitrary finalization when resumed.

The `borrow` accessor is a specialization of that model which expresses that no finalization is required. It is an ordinary function that returns a borrowed value. Swift should be able to return borrowed value without adding pointer indirection for simple types. The compiler must prove that the borrow is valid within some some that encloses the call to the accessor.

Both of these accessors naturally work for non-`Copyable` value types.

A copying read accessor (`get`) can be used to synthesize the implementation of a borrowing read accessor:

```swift
let temporary = get()   // copy the current value of the storage
yield temporary         // allow the client to read the value
_ = consume temporary   // destroy the copy
```

This synthesized implementation generally requires a coroutine because the temporary must be destroyed and deallocated as a finalization step. This synthesis also generally doesn't work for non-`Copyable` value types because storage declarations of such types typically cannot provide a `get` in the first place. Such declarations must define some kind of borrowing read accessor.

If the value type is `Copyable`, a borrowing read accessor can be used to synthesize an implementation of a `get` accessor by borrowing the value, copying it, and then immediately ending the borrow.

A `borrow` accessor can be used to synthesize an implementation of a `read` accessor by just yielding the borrowed value returned by the `borrow` accessor and then do nothing in the finalization stage.

Swift has long had experimental support for `read` accessors using the unofficial spelling `_read`. There is currently a proposal being pitched to add these accessors officially to the language with the name `read`; there are some other small differences, but mostly the behavior is the same. We are also exploring a more efficient implementation approach for `read` than that used by `_read`.

`borrow` accessors are not currently implemented, but we expect to propose them for Swift Evolution at some point in the future.

### `modify` and `mutate`

`modify` and `mutate` are both modification accessors.

The `modify` accessor is the natural most-general model of a modification access. It is a coroutine function which yields a reference to mutable memory and can then do arbitrary finalization when resumed.

The `mutate` accessor is a specialization of that model which expresses that no finalization is required. It is an ordinary function that returns a reference to mutable memory. The compiler must prove that the access to that memory is exclusive within some scope that encloses the call to the accessor.

Both of these accessors naturally work for non-`Copyable` value types.

A `mutate` accessor can be used to synthesize an implementation of a `modify` accessor by just yielding the reference returned by the `mutate` accessor and then do nothing in the finalization stage.

Swift has long had experimental support for `modify` accessors using the unofficial spelling `_modify`. There is currently a proposal being pitched to add these accessors officially to the language with the name `modify`; there are some other small differences, but mostly the behavior is the same. We are also exploring a more efficient implementation approach for `modify` coroutines than that used by `_modify`.

`mutate` accessors are not currently implemented, but we expect to propose them for Swift Evolution at some point in the future.

### `unsafeAddress` and `unsafeMutableAddress`

`unsafeAddress` is functionally a borrowing read accessor. It is an ordinary function that returns an `UnsafePointer<ValueType>`.

`unsafeMutableAddress` is functionally a modification accessor. It is an ordinary function that returns an `UnsafeMutablePointer<ValueType>`.

These accessors are early analogues of the `borrow` and `mutate` accessors built on unsafe foundations. The compiler implicitly dereferences the returned pointer to perform the access, with some special logic to keep the access to the base value (if there is one) active during the use of the pointer, which would otherwise cause immediate lifetime-safety problems.

These are currently implemented in the compiler and are used in the standard library, but they're not expected to ever go through Swift Evolution. 

### `unsafeRawAddress` and `unsafeRawMutableAddress`

These are similar to the above, but they allow the implementation to work directly an untyped "raw" pointer while the caller sees an access to typed data in memory.

These are not yet implemented and may not ever go through Swift Evolution. This document will refer to these and the previous two collectively as `unsafe*Address`.

## Distinguishing Features

Accessors vary in how they treat both the storage value and (where applicable) the containing value. In this section, we’ll explore how that affects their use in code.

### Copying vs. borrowing

We've already discussed many of the differences between accessors that implement copying and borrowing reads. To summarize, all of these read accessors except `get` are designed to allow the value of the storage to be "borrowed": reading the information in the value without copying it to produce an independent value.

It might seem that borrowing is strictly better, and in a narrow way that's true: in isolation, it is cheaper to borrow a value out of memory than to copy it. However, borrowing by its nature is *scoped*. This means that there is some duration within the execution of the program during which the borrow is valid. Within this duration, Swift must ensure that no code tries to write to the memory that's been borrowed from. Outside of this duration, Swift must ensure that the borrowed value stops being used. So borrowing can only possibly work if Swift can figure out a scope that it can safely make those two guarantees for.

Unfortunately, doing that in arbitrary code is not reliably possible. Swift frequently inserts implicit conservative copies because it cannot figure out that it could have safely borrowed. A hypothetical Non-Copying Swift that refused to do this would often force programmers to either insert those same copies explicitly or find a clever way to restructure their code to make borrowing possible. It is reasonable to argue that that would not a good trade-off for most code, where the cost of copying is likely small and the usability costs would be quite high.

Furthermore, if a borrow has to be done with a `read` accessor, the coroutine nature of the accessor also has to be considered, because that comes with its own overhead. This is particularly true if the value is going to be copied anyway and so the coroutine ultimately provided no benefit.

None of this is to deny that avoiding copies can often be important for performance or even semantically required. Many of the accessors described in this document are there specifically to enable in-place borrowing and mutation because of the costs of adding extra copies. It just has to be said that borrows are not a panacea and often require substantial differences to coding patterns to be used reliably.

### Exposing transformed temporary values

Because coroutine accessors like `read` and `modify` allow non-trivial finalization steps after they complete, they enable some interesting things to be expressed. One of these is that they can expose a transformed version of a stored value.

For example, `Dictionary`'s default key-based `subscript` exposes the value as an `Optional` so that it can return `nil` if the key is not mapped. `Dictionary` doesn't actually want to store the values as `Optional`s in its internal hashtable because this would require additional storage space for every entry. When the client performs a modification access to this optional value, `Dictionary` can take advantage of the coroutine nature of `modify` by moving the current value (if present) into a temporary `Optional`, allowing the client to mutate that, and then either moving the new value back or, if it's become `nil`, removing the original entry from the hashtable. This avoids any unnecessary copies of the original value, which both avoids some low-level overhead and allows higher-level optimizations like copy-on-write uniqueness checks to continue to succeed.

Another example: providing a `Span` over the contents of a `String` can sometimes require temporary allocation to handle the case where a short `String` is stored inline. This therefore requires a coroutine `read`.

### Access Scopes

Every access has a *scope*. The access begins at one point in the computation history of the program and then ends at another point. Consider this code:

```swift
arrayOne[i] = arrayTwo[j]
```

Assuming that these are `Array`s, that all of these names are simple stored variables, and this code sequence is executed optimally, this performs the following formal sequence of abstract operations:

1. A copying read access begins on `i`.
2. The current value of access 1 is copied.
3. Access 1 ends.
4. A copying read access begins on `j`.
5. The current value of access 4 is copied.
6. Access 4 ends.
7. A borrowing read access begins on `arrayTwo`.
8. A copying read access begins on `Array.subscript` (on access 7, with the index value from 5).
9. The current value of access 8 is copied.
10. Access 8 ends.
11. Access 7 ends.
12. A modification access begins on `arrayOne`.
13. An assignment access begins on `Array.subscript` (on access 12, with the index value from 2).
14. The element value from 9 is assigned into access 13.
15. Access 13 ends.
16. Access 12 ends.

Note that accesses to instance members of value types (here, the `Array.subscript`s in steps 8 and 13) always occur within compatible accesses to the containing value.

Swift has an exclusivity rule which governs accesses to real memory locations and prevents them from conflicting. In this example, this applies to the accesses started at steps 1, 4, 7, and 12. The accesses at steps 8 and 13 are to abstract storage declarations implemented with accessors, not to simple stored variables, and so exclusivity does not directly apply, other than the guarantee of exclusivity on `self` that any method on a value type gets.

(The `Array.subscript` accessors internally perform unsafe memory accesses on the array buffer which Swift cannot enforce exclusivity on. However, the exclusivity of `self` is enough for these unsafe accesses to be proven to follow exclusivity, exactly as if the array elements were stored properties of the array. This is another way of saying that array elements use "value semantics" and are still statically memory-safe.)

Copying read accesses (`get`s) and assignment accesses (`set`s) to real memory are considered "instananeous", which means they begin and end without arbitrary code being run in the middle. The only thing that happens within the access scope is the primitive value copy / assignment. Other accesses are non-instantaneous because arbitrary code is run during the access scope, such as the accessor function calls for the `subscript` accesses to the arrays in the example. Swift generally tries to keep access scopes as short as possible to avoid unnecessary exclusivity conflicts.

* `unsafe*Address` accessors return pointers into the containing value.  This is safe for the caller of these accessors because the compiler knows about this relationship and can extend the lifetime of the containing value as needed.  (These accessors are nominally “unsafe” because their implementation requires constructing an unsafe pointer and there are no checks to ensure that the pointer so constructed is in fact valid.  For example, there is no check on the pointee lifetime.))

* `read`/`modify` expose a value for the lifetime of a coroutine. Coroutines enforce that the containing value remains alive for the duration of the coroutine.  Note that in current Swift, the coroutines are generally quite short-lived and the compiler copies the value into or out of the coroutine fairly aggressively.  In the future, the compiler will likely become more adept at expanding the coroutine lifetime to reduce such copying.

* `borrow`/`mutate` return a borrow of the property value.  This borrow has an implied dependence on the containing value, and the compiler must guarantee that the containing value outlives the property access.

### Ownership of the containing value

When a storage declaration is an instance member of a value type[^1], any access to it is also an access to the containing value. The kind of access performed on the containing value depends on both the kind of access performed on the member and how that access is implemented.

[^1]: Reference types are not accessed when their members are accessed. The reference is provided to the operation, but there is no access scope or exclusivity associated with the referenced object as a whole.

The simplest case is when the member is a simple stored property. Reading from a stored property (whether borrowing or copying) requires a borrowing read of the containing value. Writing to a stored property (whether an assignment or a modification) requires a modification of the containing value.

Most non-stored members of value types are still meant to behave like value members and follow the same rule as stored properties. Reading accessors (`get`, `read`, and `borrow`) are `nonmutating` methods and therefore require a borrowing read access to the containing value. Writing accessors (`set`, `modify`, and `mutate`) are `mutating` methods and therefore require a modification access to the containing value.

However, this can be overridden by explicitly making an reading accessor `mutating`, a writing accessor `nonmutating`, or either kind of accessor `consuming`. This has ownership implications for the containing value when the accessor needs to be used. Some combinations don't make much sense, however.

Making a reading accessor `mutating` allows the accessor implementation to mutate the containing value, but it also requires the caller to treat this as a modification access that needs stronger exclusivity enforcement. This is commonly used for caching or bookkeeping:

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

Conversely, writing accessors can be marked as `nonmutating`. This prevents the accessor implementation from mutating the containing value, but it allows the caller to use the weaker exclusivity enforcement associated with a reading access. This is mostly useful when the type has some kind of reference-like semantics rather than the typical value semantics of a value type.

Accessors can also be `consuming`. Ownership of the containing value will be transfered into the accessor, either by copying it or by ending the lifetime of the value in its original location. This is useful for representing certain types of object transformations:

```swift
struct Values: ~Copyable {
  var dropOldest: Values {
    consuming get {
       let newValues = ... everything but the oldest ...
       return newValues
    }
  }
}

let v = Values()
let v2 = v.dropOldest
// `v` is no longer usable here
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
