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

In this case, the `get` accessor behaves just like a `nonmutating` method that returns a value of the property's type, while the `set` accessor behaves just like a `mutating` method that receives a value of the property's type as an argument.

The `get` and `set` accessors are ideal for implementing operations that copy the current value of the property:

```swift
let copy = myFoo.value  // calls the get accessor for Foo.value
```

or that overwrite the current value of the property:

```swift
myFoo.value = 51        // calls the set accessor for Foo.value
```

Other kinds of operations can also be defined in terms of `get` and `set`. For example, if you pass a computed property as an `inout` argument:

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

2. It can make subsequent uses of the value less efficient. For example, if the value uses a copy-on-write representation like `Array` and `String` do, mutating a copy is likely to be dramatically less efficient than mutating a variable in place. (We will explain this in more detail later.)

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

func printNameConcretely(_ person: Person) {
  print(person.name)
}

func printNameGenerically(_ person: any Nameable) {
  print(person.name)
}
```

In `printNameConcretely`, Swift knows that `name` is a stored property of `Person`, and it can just load that value directly from `person` and pass it to `print`. In `printNameGenerically`, Swift does not know how `name` is implemented, and it must call a `get` accessor to copy the current value of the name. To avoid those costs, the Swift optimizer would have to specialize this function for the specific type that is being passed in; this is something that Swift can and does do, but only as a best-effort optimization, which is not always good enough. And, of course, this code would be ill-formed if `String` were a non-`Copyable` type, because the only way to satisfy a `get` requirement for a stored property is to copy the current value.

As a result, Swift has explored a variety of other accessors throughout its history, none of which have ever been officially added to the language through the Swift Evolution process. (The [observing accessors](#observing-accessors), `willSet` and `didSet`, are officially in the language but are categorically different.) Many of these have been adopted in the standard library for years, but we've been reluctant to make them official because they are variously incomplete, unsafe, or complex.

This vision document lays out the design space of accessors for the next few years, as Swift continues to advance its support for non-`Copyable` and non-`Escapable` types. It explains Swift's basic access model and how it may need to evolve. It explores what developers need from accessors in these advanced situations. Finally, it discusses different kinds of accessors, both existing and under consideration, and how they do or do not fit into the future of the language as we see it.

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
  - a modification if `value` is a stored property of a `struct` or an element of a tuple,
  - a modification if `value` is defined with a `mutating set`,
  - a read if `value` is a stored property of a `class` or `actor`, or
  - a read if `value` is defined with a `nonmutating set`.

The old manifesto's description of modification and assignment accesses has stood up to time, but its description of read accesses arguably has not. As more advanced ownership features have developed in the language, Swift has increasingly needed to distinguish at least two kinds of reads:

A *copying read access* conceptually means that the current value of the storage is being copied (or consumed) to produce an independent value. At minimum, it occurs when the storage reference expression appears in any context where an independent value is required, such as a return value or as the right operand of the `=` operator.

A *borrowing read access* conceptually means that the current value of the storage is being temporarily borrowed in order to read it without copying it. At minimum, it occurs when the storage reference expression appears in any non-mutating context where an implicit copy is not allowed, such as being passed as a `borrowing` argument when the value is non-`Copyable`.

The semantic and implementation-level differences between borrowing and copying/consuming uses will be very important in this document.

It is reasonable to ask whether Swift could categorize *all* read accesses into these two kinds based on context, the same way that it distinguishes reads from assignments. This would be straightforward at a technical level, but it is controversial as a design direction because the most obvious definitions would be very aggressive about borrowing values. This could cause surprising semantic problems for Swift programmers, especially those working extensively with classes, and it could break the behavior of existing code. This remains an open question that this vision does not take a stand on.

## Phases of access

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

But it's also really useful because it frees the client of the burden of running code when the access ends. The general pattern that we described above has to run in two distinct phases. That makes it inherently something like a coroutine: it's going to be split into multiple functions, and it might have to dynamically allocate memory to pass between them. The client must then keep all of this information around dynamically for each access it performs. If the client starts a dynamic number of accesses at once (such as by starting an access on each iteration of a loop), it will need dynamic allocation to remember all of the active accesses. In contrast, if there's a guarantee that finalization is unnecessary, the implementation of the access can work more like a normal function: the function will just perform the first phase and pass back whatever it needs to as a return value. The client has nothing to track, so it can manage even a dynamic number of accesses completely statically.

## Six of Eight Fundamental Accessors

So we have four kinds of access, and we can classify accessor implementations by whether they need to be coroutines. That gives us a 2x4 matrix of possible accessors. Two of these combinations, however, turn out to not be useful:

|              |Copying read |Borrowing read |Modification|Assignment |
|---           |---          |---            |---         |---        |
|**Routine**   |             |               |            |           |
|**Coroutine** |✗            |               |            |✗          |

A copying read access has to generate an independent value during the first phase. Given that, it seems illogical for it to need to do any work during finalization: if the value is only usable within the scope of a coroutine call, it is not truly an independent value. This document will by-and-large accept this logic. However, there is a counterargument that applies to non-`Escapable` value types; see the section below on "Values that represent accesses".

An assignment access just consumes an independent value. Just like above, it's unclear what an assignment accessor could possibly need to do that would require it to be split into multiple phases.

## Existing and Proposed Standard Accessors

Based on the discussion above, we are left with six combinations that are both possible and useful.  The following table gives each one a name:

|              |Copying read |Borrowing read |Modification|Assignment |
|---           |---          |---            |---         |---        |
|**Routine**   |get          |borrow         |mutate      |set        |
|**Coroutine** |✗            |yield          |yield inout |✗          |

Some of the above exist in the current Swift language, others we expect to be proposed in the near future.

### `get`

The `get` accessor is the natural most-general model of a copying read access. It is an ordinary function that returns an independent value that is conceptually a copy of the current value of the storage.

A `get` accessor can be used with a non-`Copyable` value type; the accessor simply has to return an independent value every time. This is usually impossible to implement when the storage declaration is just providing access to a non-copyable value stored in memory. That makes sense: if there is a `T` stored in memory, and `T` is not copyable, it should not be possible to access it with a copying read. For example, if `T` is `~Copyable`, then the `subscript` on `Array<T>` cannot be implemented with a `get` because it would have to copy the element, which it cannot do.

However, a closely-related pattern *is* possible with non-`Copyable` types. If the value type of the storage declaration is not the in-memory type `T`, but instead reflects an ongoing *access* to some underlying `T`, then programmer's use of the storage declaration actually represents a request to begin the underlying access. (In memory-safe code, this requires the value type to be `~Escapable`.) For example, a value of type `Span<T>` represents an ongoing read access to an array of `T` values. `Array<T>` can offer a `span` property that returns a `Span<T>`; by scope-restricting the span to the current borrow of `self`, the array can safely offer access to its elements. This works even for non-`Copyable` element types. This property would have to be implemented with a `get` accessor, because the `Span` value it returns is a new value and does not preexist in memory. See the section "Values that represent accesses" later in this document.

### `set`

The `set` accessor is the natural most-general model of an assignment access. It is an ordinary function that takes a new independent value as an argument and conceptually replaces the current value of the storage with it. The value is taken as a `consuming` argument, so the `set` accessor generalizes perfectly well to non-`Copyable` value types.

A storage declaration can usefully define both a `set` accessor and a modification accessor. Assignment accesses will just call `set`, allowing them to bypass any overhead that might be associated with reading the current value. Modification accesses will ignore the `set` and just use the modification accessor. However, if there isn't any overhead for reading the current value --- for example, if the value is already stored in memory somewhere --- then this is unlikely to be a useful optimization over just defining a non-coroutine modification accessor.

### `yield` and `borrow`

The `yield` accessor is the natural most-general model of a borrowing read access. It is a coroutine function which yields a borrowed value and can then do arbitrary finalization when resumed.

The `borrow` accessor is a specialization of that model which expresses that no finalization is required. It is an ordinary function that returns a borrowed value. Swift should be able to return a borrowed value without adding pointer indirection for simple types. The compiler must prove that the borrow is valid within some that encloses the call to the accessor.

Both of these accessors naturally work for non-`Copyable` value types.

Swift has long had experimental support for `yield` accessors using the unofficial spelling `_read`. There is currently [a proposal being pitched](https://forums.swift.org/t/pitch-modify-and-read-accessors/75627) to add these accessors officially to the language; the current draft uses the name `read`, but `yield` has been floated in the discussion. This document uses `yield` because it avoids conflating read accesses in general with the name of a specific accessor, and it makes it very clear which borrowing read accessor is a coroutine. The pitched feature differs in some small ways from `_read`, but the overall concept is the same. The proposal authors are also exploring a more efficient implementation approach for `yield` than that used by `_read`.

`borrow` accessors are not currently implemented, but we expect to propose them for Swift Evolution at some point in the future.

### `yield inout` and `mutate`

`yield inout` and `mutate` are both modification accessors.

The `yield inout` accessor is the natural most-general model of a modification access. It is a coroutine function which yields a reference to mutable memory and can then do arbitrary finalization when resumed.

The `mutate` accessor is a specialization of that model which expresses that no finalization is required. It is an ordinary function that returns a reference to mutable memory. The compiler must prove that the access to that memory is exclusive within some scope that encloses the call to the accessor.

Both of these accessors naturally work for non-`Copyable` value types.

Swift has long had experimental support for `yield inout` accessors using the unofficial spelling `_modify`. There is currently [a proposal being pitched](https://forums.swift.org/t/pitch-modify-and-read-accessors/75627) to add these accessors officially to the language; the current draft uses the name `modify`, but `yield inout` has been floated in the discussion. As with `yield`, above, this document uses `yield inout` to sidestep some confusion. The pitched feature differs in some small ways from `_modify`, but the overall concept is the same. The proposal authors are also exploring a more efficient implementation approach for `yield inout` than that used by `_modify`.

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

Unfortunately, doing that in arbitrary code is not reliably possible. Swift frequently inserts implicit conservative copies because it cannot figure out that it could have safely borrowed. A hypothetical Non-Copying Swift that refused to do this would often force programmers to either insert those same copies explicitly or find a clever way to restructure their code to make borrowing possible. By inserting its conservative copies, Swift is implicitly arguing that this is not a good trade-off for most code, where the cost of copying is likely small and the usability costs would be quite high.

Furthermore, if a borrow has to be done with a `yield` accessor, the coroutine nature of the accessor also has to be considered, because that comes with its own overhead. This is particularly true if the value is going to be copied anyway and so the coroutine ultimately provided no benefit.

None of this is to deny that avoiding copies can often be important for performance or even semantically required. Many of the accessors described in this document are there specifically to enable in-place borrowing and mutation because of the costs of adding extra copies. It just has to be said that borrows are not a panacea and often require substantial differences to coding patterns to be used reliably.

### Exposing transformed temporary values

Because coroutine accessors like `yield` and `yield inout` allow non-trivial finalization steps after they complete, they enable some interesting things to be expressed. One of these is that they can expose a transformed version of a stored value.

For example, `Dictionary`'s default key-based `subscript` exposes the value as an `Optional` so that it can return `nil` if the key is not mapped. `Dictionary` doesn't actually want to store the values as `Optional`s in its internal hashtable because this would require additional storage space for every entry. When the client performs a modification access to this optional value, `Dictionary` can take advantage of the coroutine nature of `yield inout` by moving the current value (if present) into a temporary `Optional`, allowing the client to mutate that, and then either moving the new value back or, if it's become `nil`, removing the original entry from the hashtable. This avoids any unnecessary copies of the original value, which both avoids some low-level overhead and allows higher-level optimizations like copy-on-write uniqueness checks to continue to succeed.

Another example: providing a `Span` over the contents of a `String` can sometimes require temporary allocation to handle the case where a short `String` is stored inline. This therefore requires a coroutine `yield`.

### Access scopes and exclusivity

Every access in Swift has an *access scope*: the access begins at one point in the computation history of the program and then ends at a later point. Consider this code:

```swift
arrayOne[i] = arrayTwo[j]
```

Assuming these are `Array`s, that all of these names are simple stored variables, and this code sequence is executed optimally, this performs the following formal sequence of abstract operations:

1. A copying read access begins on `i`.
2. The current value of `i` is copied (using access 1, by directly accessing memory).
3. Access 1 ends on `i`.
4. A copying read access begins on `j`.
5. The current value of `j` is copied (using access 4, by directly accessing memory).
6. Access 4 ends on `j`.
7. A borrowing read access begins on `arrayTwo`.
8. A copying read access begins on `arrayTwo.subscript` (using access 7, with the index value read from `j` at step 5).
9. The current value of `arrayTwo.subscript` copied (using access 8, by calling the `get` accessor).
10. Access 8 ends on `arrayTwo.subscript`.
11. Access 7 ends on `arrayTwo`.
12. A modification access begins on `arrayOne`.
13. An assignment access begins on `arrayOne.subscript` (using access 12, with the index value read from `i` at step 2).
14. The element value read at `arrayTwo.subscript` at step 9 is assigned into `arrayOne.subscript` (using access 13, by calling the `set` accessor).
15. Access 13 ends on `arrayOne.subscript`.
16. Access 12 ends on `arrayOne`.

Note that accesses to instance members of value types (here, the `subscript`s in steps 8 and 13) always occur within compatible accesses to the containing value.

Note also that `Array.subscript` does not directly define `get` and `set` accessors; they are synthesized from the accessors it does define. See the section later about implementing accessors using other accessors.

Swift has a memory exclusivity rule which governs all accesses to real memory locations. In this example, this applies to the accesses started at steps 1, 4, 7, and 12. The exclusivity rule requires all memory accesses to not conflict, which they do if:
- they are to the same memory location,
- neither access is derived from the other,
- at least one of them is a writing access (a modification or assignment), and
- the scopes of the accesses overlap (the start of one is not ordered after the end of the other).
This is mostly enforced with a combination of static and dynamic checks. In some cases, such as unsafe pointers or unsafe concurrency, the programmers must take care to obey the exclusivity rule manually.

Exclusivity only applies to real memory locations. Storage declarations that are implemented with accessors are not real memory locations, so the accesses at step 8 and 13 are not directly required to obey exclusivity. If a `struct` has a propety with a `nonmutating get` and a `nonmutating set`, it is legal to call them at overlapping times, even though this would violate the "abstract exclusivity" of the computed property. This would only be a problem if the accessors performed conflicting accesses to real memory locations, and they cannot do that to the stored properties of the `struct` itself because they can only read those locations.

Even though exclusivity isn't directly enforced for storage declarations implemented with accessors, the guarantees offered by exclusivity are often still useful for establishing correctness within accessors. For example, the accessors on `Array.subscript` access memory in the array buffer through unsafe pointers. These accessors are methods, so they receive normal exclusivity guarantees about `self`:
- The reading accessors are `nonmutating` methods, so they have read access to `self` and a guarantee that no other code can have write access to `self` during the method.
- The writing accessors are `mutating` methods, so they have write access to `self` and a guarantee that no other code can have any access to `self` during the method.
`Array` only ever attempts to write to the array buffer within `mutating` methods, so the reading accessors know that there cannot be any writes to the buffer that would conflict with its reads from the buffer. Similarly, the writing accessors know that there cannot be any other accesses to the buffer at all that would conflict with its own reads and writes to the buffer. So even though these buffer accesses are done unsafely, they all still provably obey Swift's memory exclusivity rule.

### Access scopes and lifetime dependencies

Access scopes often must have certain relationships to each other for safety. For example, `Array.subscript` can theoretically allow the element value to be borrowed, but only within the scope of the borrow of the containing array value. The introduction of non-`Escapable` types (beginning in [SE-0446][]) adds a new variation on this because lifetime dependencies can be inherently carried by a value. For example, `Span.subscript` can allow the element value to be borrowed, but only within the original scope that originally gave rise to that `Span` value. That scope is likely to be significantly wider than the scope of a specific access to `Span.subscript`, which is part of the novel power of `Span`.

#### Lifetime dependencies of non-escapable value types

If the value type of a storage declaration is a non-`Escapable` type, the lifetime dependencies of the value are an intrinsic part of the signature of the declaration. They can be related in any way to the access scopes and lifetime dependencies available in the context of the declaration.

For example, it has been proposed that `Array` should have a `span` property. This property would be defined with a `get` accessor that returns a `Span<Element>` that refers to the element storage of the array buffer. This is a new span value, but it carries a lifetime dependency on the access scope of the borrow of the `Array` value that the accessor was called within, because it is only within that scope that the element storage is guaranteed to remain both valid and immutable.

If the array also has non-`Escapable` elements, then the lifetime dependency of the element type of the span must be the same as the lifetime dependency of the element type of the original array.

This property can return the `Span` with a `get`, and use the access scope of the borrow of the `Array`, because the span always refers to the existing memory of the array. A similar property on `String` might not have this option because `String` can store small strings in a compressed form that isn't "in memory". We are looking at whether this problem can be narrowly addressed, but if not, `String.span` would have to produce the `Span` with a `yield` accessor to allow local allocation of the span's array, and it would have to return a `Span` with a lifetime dependency of the yield, not the enclosing borrow of the `String`.

In contrast, suppose that a different struct simply stores a `Span<Int>`. This would again be an instance property of the struct, but the lifetime relationship would be very different from either of the two cases above. The containing struct would have to be `~Escapable` and have its own lifetime dependency matching the dependency of the span. A copying read of that stored property (analogous to using the `get` accessor on `Array.span`) must produce a span with that same lifetime dependency, not a dependency narrowed to the access to the struct. Similarly, a write to the property must leave it holding a span with the same lifetime dependencies, or else subsequent uses of the span (or its elements) might be corrupted.

Reading these examples, you might be tempted to say that the span can actually have a broader or narrower lifetime dependency in some cases. For example, it would be fine to store a `Span` with a broader lifetime into the stored property in the last example. One way to understand this in general is as a dependency-subtyping conversion that changes the lifetime dependencies of the value. `Span` specifically is "covariant" in both its memory dependency and its element dependencies. If the underlying memory of a span is safe within scope `X`, and `Y` is a strictly small scope than `X`, then it's okay to narrow the span's memory dependency to `Y`. Similarly, since `Span` only provides read access to its elements, it's okay to apply a dependency-subtyping conversion to the element type, because this is equivalent to doing the same conversion after every read. The first of these is also true of `MutableSpan`, but the second is not because it would allow a value with a narrower dependency to be stored into the span. That is, `MutableSpan` is covariant in its memory dependency but invariant in its element dependency.

#### Scope of usability of the value

When reading or modifying a storage declaration, the declaration makes guarantees about the access scope in which the value is safe to use. Like the lifetime dependencies of non-`Escapable` value types, these guarantees are intrinsic to the overall signature of the storage declaration. A storage declaration that makes a certain guarantee cannot evolve to make a weaker guarantee. Unlike the dependencies of the value type, the access scope restrictions are more specific to exactly how the access is performed. `get` and `set` accessors simply return and accept independent values, with no scope restrictions necessary. `yield`, `borrow`, `yield inout`, and `mutate` all inherently provide access to the value only within a specific access scope, which can be narrower than any lifetime dependencies of the value itself.

`borrow` and `mutate` require the access scope to be broader than the access itself because the returned value or reference must be valid to use after the accessor returns. A common choice for instance members of value types would be the access scope of `self`, which matches what value types naturally guarantee for their stored properties. But it can also be some other contextual lifetime dependency. For example, `Span.subscript` can borrow elements with a scope matching the memory dependency of the span, which is much broader than some scope associated with the subscript access, because the elements are immutable within that entire scope.

By design, `yield` and `yield inout` can provide access for a narrower scope than `borrow` and `mutate` can. The most conservative assumption is that the value or reference can only be used during the duration of the coroutine, which is to say, for the scope of the access itself. In principle, a `yield` or `yield inout` accessor could promise that the value or reference is safe to access even after the coroutine completes. However, it's unclear why that would ever be useful, because it means that the finalization phase of the access --- the whole purpose of using a coroutine accessor in the first place --- is no longer reliably ordered after the client is done using the value or reference. For example, a `didSet`-like finalization at the end of a `yield inout` accessor which sends notifications whenever the value changes would potentially miss changes because the reference can still be modified after the finalization is triggered. The only useful rule appears to be that the access scope of the value/reference is nested within the coroutine.

### Ownership of the containing value

When a storage declaration is an instance member of a value type[^1], any access to it is also an access to the containing value. The kind of access performed on the containing value depends on both the kind of access performed on the member and how that access is implemented.

[^1]: Reference types are not accessed when their members are accessed. The reference is provided to the operation, but there is no access scope or exclusivity associated with the referenced object as a whole.

The simplest case is when the member is a simple stored property. Reading from a stored property (whether borrowing or copying) requires a borrowing read of the containing value. Writing to a stored property (whether an assignment or a modification) requires a modification of the containing value.

Most non-stored members of value types are still meant to behave like value members and follow the same rule as stored properties. Reading accessors (`get`, `yield`, and `borrow`) are `nonmutating` methods and therefore require a borrowing read access to the containing value. Writing accessors (`set`, `yield inout`, and `mutate`) are `mutating` methods and therefore require a modification access to the containing value.

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

This only leaves `consuming get` and `consuming yield` as meaningful combinations.  A `consuming get` can be used for operations such as the `dropOldest` example above that model a transformation by creating a new value and terminating the old one.  The `consuming yield` variant can be used similarly.  Unlike `borrow`, the `yield` operation coroutine structure forces the containing value to live for a certain period of time, the value is consumed only at the end of that coroutine.

### Implementing access kinds with other accessors

Accessors can often be used to perform other kinds of access than they naturally implement. Effectively, this involves synthesizing some other kind of accessor, although the compiler often prefers to emit the appropriate code inline at use sites instead of actually creating and calling a synthetic function.

This synthesis is important to understand when abstraction is required. For example, suppose a type has a property which is used to satisfy a protocol requirement that expects `yield`, `yield inout`, and `set` accessors. The protocol conformance for the type must synthesize these accessors in terms of the actual implementation of the property. If the synthesis isn't possible, Swift must reject the conformance and report the problem to the programmer.

When the storage declaration is an instance member of a value type, the ownership requirement of a synthesized accessor must be compatible with all of the accesses that the synthesis requires:
- If any of the accesses requires consuming `self`, it must be the last access performed to `self`, and the synthesized accessor must itself be `consuming`.
- If any of the accesses requires mutating `self`, the synthesized accessor must be `consuming` or `mutating`.

#### Stored properties

Every kind of accessor can be synthesized efficiently for copyable stored properties of value types.

If the type of the property is not `Copyable`, `get` cannot be synthesized.

If the property is mutable, and it's either `static` or the containing type is a class, then accesses to it generally require dynamic exclusivity enforcement. This requires a non-trivial finalization step to dynamically record the end of the access, and that means `borrow` and `mutate` cannot be synthesized.

#### `get`

If the value type is `Copyable`, either kind of borrowing read accessor can be used to synthesize a `get` accessor by borrowing the value, copying it, and then immediately ending the borrow.

When synthesizing `get` using a `borrow` accessor, which must return a value borrowed out of memory that will outlive the accessor, this is very likely to be just as efficient as a direct implementation would have been. The synthesized `get` is just performing the copy *after* the return rather than *before* it, which should make no real difference to performance.[^2]

[^2]: It *could* be less efficient if putting the value in borrowable memory is an avoidable step. For example, if the `borrow` is actually generating new values on each access, but must allocate memory to store the value so it can be borrowed, a direct `get` implementation would be more efficient than copying the value returned by `borrow`. That would be a very questionable implementation of `borrow`, however; it should really just be a `get` to begin with.

That is not as true when synthesizing `get` using a `yield` accessor. For one, the low-level overhead of setting up the `yield` coroutine could be avoided by a direct `get` implementation. Additionally, however, a `yield` coroutine is more likely to be setting up a borrow out of temporary memory, which is work that a `get` accessor could avoid by just returning the desired value directly.

#### `set`

A `set` accessor can be synthesized using either kind of modification accessor. There are no restrictions on this synthesis.

When synthesizing `set` using a `mutate` accessor, which must return a reference to stable, mutable memory, this is very likely to be just as efficient as a direct implementation would have been. The synthesized `set` is just performing the assignment *after* the return rather than *before* it. Any exception is likely to be something that shouldn't have been implemented with `mutate` in the first place because the value shouldn't really be guaranteed to be in stable memory.

This is less true when synthesizing `set` using a `yield inout` accessor. For one, the low-level overhead of setting up the `yield inout` coroutine could be avoided by a direct `set` implementation. Additionally, a `yield inout` coroutine is more likely to be setting up a mutation of temporary memory and then doing arbitrary work with the value afterwards. A direct implementation of `set` could avoid the need for the temporary, avoiding both some low-level overhead and the entire step of reading the current value only for it to be completely overwritten.

#### `yield` and `borrow`

The borrowing read accessor `yield` can be synthesized using the copying read accessor `get`:

```swift
let temporary = get()   // copy the current value of the storage
yield temporary         // allow the client to read the value
_ = consume temporary   // destroy the copy
```

This synthesized implementation requires a coroutine because the temporary must be destroyed and deallocated as a finalization step. It therefore can only be used to synthesize `yield` and not `borrow`.

This synthesis usually doesn't work for non-`Copyable` value types because storage declarations of non-`Copyable` type typically cannot provide a `get` in the first place.

`yield` can also be synhesized using `borrow`: the coroutine calls `borrow`, yields the result, and does nothing in the finalization stage.

A `borrow` accessor can only be synthesized using a stored variable or an accessor with an equivalent lifetime guarantee to `borrow`, like `unsafe*Address`. Moreover, it can only be synthesized using a stored variable if exclusivity for the variable can be statically guaranteed, such as if the variable is immutable or is a stored property of a value type. Other stored variables require dynamic exclusivity checks for safety, which adds dynamic finalization to the access.

#### `yield inout` and `mutate`

A `yield inout` accessor can be synthesized using `get` and `set`:

```swift
var temporary = get()  // copy the current value of the storage
temporary.mutate()     // allow the client to do its modification
set(consume temporary) // replace the current value of the storage
```

This synthesized modification access requires a coroutine model because the call to `set` is a non-trivial finalization step.

Note that the `get` can itself be synthetic in this synthesis. For example, if the original storage declaration provides a `borrow` and a `set`, the `get` can be synthesized in terms of the `borrow`, and then the `yield inout` can be synthesized in terms of the `set` and the synthesized `get`. Any requirements for synthesizing the `get` also apply to synthesizing the `yield inout`; in particular, the value type must be `Copyable`.

Because this synthesis requires a `get`, it generally doesn't work for non-`Copyable` value types. (It could work if the storage declaration provides a `get`, but such an accessor usually can't be defined unless it's `consuming`, which would prevent the `set` from being called later.) To support modification, a storage declaration of non-`Copyable` type must generally also define either `yield inout` or `mutate`.

A `yield inout` accessor can also be synthsized using `mutate` by just yielding the reference returned by the `mutate` accessor and then doing nothing in the finalization stage.

A `mutate` accessor can only be synthesized using a stored variable or an accessor with an equivalent lifetime guarantee to `mutate`, like `unsafe*MutableAddress`.  Moreover, it can only be synthesized using a stored variable if exclusivity for the variable can be statically guaranteed, such as if the variable is a stored property of a value type. Other stored variables require dynamic exclusivity checks for safety, which adds dynamic finalization to the access.

### Observing accessors

The observing accessors, `willSet` and `didSet`, are categorically different from other accessors because they do not fully implement any of the basic access kinds. Instead, they "decorate" an underlying implementation. Usually, the underlying implementation is a stored variable, but it can also be inherited from a superclass if the accessor is added in an override. Observing accessors can never combined with non-observing accessors.

Read accesses to a storage declaration with observing accessors always go directly to the underlying implementation.[^3] They can be copying reads or borrowing reads if the underlying implementation allows it.

[^3]: As the current module sees it. If you add observing accessors in an override of a non-`frozen` superclass property defined in a different module with a stable binary interface, the access to the superclass property will be constrained by what's available in the binary interface.

Write accesses to a storage declaration with observing accessors always trigger a call to the accessor(s). The exact behavior depends on which accessors are provided and whether the `didSet` is "simple" (doesn't use its old value argument) as specified by [SE-0268][].

An assignment access behaves as if it were calling a `set` accessor synthesized as follows:

```swift
set(newValue) {
#if <there's a non-simple `didSet` accessor>
  let oldValue = underlyingStorage
#endif

#if <there's a `willSet` accessor>
  willSet(newValue)
#endif

  underlyingStorage = newValue

#if <there's a non-simple `didSet` accessor>
  didSet(oldValue)
#elseif <there's a simple `didSet` accessor>
  didSet()
#endif
}
```

If the storage declaration only provides a simple `didSet`, then a modification access is performed "in place" on the undertlying storage, and then `didSet` is called. This matches the behavior of calling a `yield inout` accessor synthesized as follows:

```swift
yield inout {
  yield &underlyingStorage
  didSet()
}
```

Otherwise, modifications are performed on a temporary produced by copying the underlying storage, which is then written back with the setter above:

```swift
yield inout {
  var temporary = underlyingStorage
  yield &temporary
  set(temporary)
}
```

Note that this exactly matches the rule for synthesizing `yield inout` for a storage declaration with `get` and `set` accessors. As usual, this is ill-formed if it is not possible to perform a copying read of the underlying storage (such as if it is a stored variable of non-`Copyable` type).

### Values that represent accesses

It is often useful to build up values from other values. A simple example is that wrapping a value in `Optional` technically makes a new value that stores the old value inside it. A more complex example might be a struct that combines a `TouristSite` with its interest score and its distance from your hotel, for use in planning a trip to a city. In any case, the easiest way to do this is generally to copy the value into the new compound value. But if you're working with non-`Copyable` types, or if you simply need to avoid copying for performance reasons, that may not be acceptable or even allowed.

The alternative is to store a type that represents an ongoing access to the value. Such a type is necessarily only safe to use within the scope of the access, which means it has to be non-`Escapable`. A type that represents an ongoing read access can still be `Copyable` because it's okay to have simultaneous read accesses to the same memory. A type that represents an ongoing modification access needs to also be `~Copyable` to preven simultaneous modifications. A value that stores one of these types generally inherits these restrictions.

For example, the `Span` type introduced by [SE-0447][] represents an ongoing read access to a contiguous array of elements. As expected, it is `~Escapable` but still `Copyable`. `MutableSpan`, which is a future direction of that proposal, would be both `~Escapable` and `~Copyable`. `Array<T>` may someday offer a `span` property that returns a `Span<T>`; reading this property would begin a read access to the array that would end when the caller was done with the span.

We are also considering types that more narrowly represent ongoing accesses to individual values. For example, `Borrow<T>` (`~Escapable`, `Copyable`) might represent an ongoing read access to a single value, while `Inout<T>` (`~Escapable`, `~Copyable`) might represent an ongoing modification access. Either could be produced directly from any storage reference expression of the right type (a mutable one, for `Inout`).

To see how these could be used, let's consider Swift's existing library facilities for lazy collections. `Sequence.lazy` returns a sequence that wraps `self` to apply certain operations like `map` and `filter` lazily instead of having them immediately produce a new collection. Today, this `LazySequence` value must store a copy of the original collection:

```swift
extension Sequence {
  public var lazy: LazySequence<Self> {
    return LazySequence(_base: self)
  }
}

public struct LazySequence<Base: Sequence> {
  internal var _base: Base
}
```

This makes it both less efficient and incapable of working with non-`Copyable` collections. It is only fairly marginally inefficient for copy-on-write collections like `Array`, because copying an `Array` just performs an extra reference-counting operation, but it would be a serious performance problem for a collection like [SE-0453][]'s `Vector` that would need to copy every element in the collection.

It would be better if the new sequence instead stored a borrow of the original collection:

```swift
extension Sequence {
  // The returned sequence is implicitly restricted to the scope of the
  // borrow of the value passed as `self`.
  public var borrowedLazy: BorrowedLazySequence<Self> {
    return BorrowedLazySequence(_base: self)
  }
}

public struct BorrowedLazySequence<Base: Sequence>: ~Escapable {
  internal var _base: Borrow<Base>
}
```

This would guarantee that the original collection would never get copied. The `BorrowedLazySequence` would only be usable within the scope of a borrow of the original collection. That's fine for typical uses of lazy sequences: programmers usually just perform a few lazy operations in a row, then immediately use the result.

Notice that `borrowedLazy` is implemented with a `get` here. In this document, we've described three reading accessors: `get`, `borrow`, and `yield`. Why does this use `get`?

Because we're working with borrows, it's natural to expect that maybe this should use `borrow`. After all, isn't a `BorrowedLazySequence` in some sense nothing but a borrow of the original sequence? But `borrow` is not a catch-all for all operations that involve borrowing; it's used specifically when the return value is being borrowed from an existing memory location. There is no existing `BorrowedLazySequence` value in memory; this accessor needs to return a new value, albeit a scope-restricted one. Therefore, this cannot be implemented with `borrow`.

Both `get` and `yield` could work here. The difference is just the usual difference between `get` and `yield`:
- `yield` allows dynamic finalization to be performed after the access;
- `yield` yields a borrowed value, not an owned one; and
- `yield` yields a value that is scope-restricted within the yield, not within the wider scope of the borrow of `self`.

Since no dynamic finalization is required, and the value is safe to use within the wider scope of the borrow of `self`, this can be implemented with a `get`.

If dynamic finalization *is* required, `yield` is potentially a problematic choice because it always returns a borrowed value, which restricts what the caller can do with the value.

Consider a `DiscontiguousArray<T>` type with a `span` property that might need to allocate temporary memory. (We are not considering whether it is actually a good idea for such a type to offer a `span` property, just how Swift would work if it did.) This allocation requires finalization, so `get` is impossible; either the property must be implemented with `yield`, or it must be redesigned as a method that passes the `Span` to a callback.

Many uses of `Span` actually need ownership of the span. For example:

```swift
extension DiscontiguousArray {
  var isPalindrome: Bool {
    var s = self.span
    while s.count > 1 {
      if s.first != s.last { return false }
      s = s.extracting(droppingFirst: 1).extracting(droppingLast: 1)
    }
    return true
  }
}
```

Note the scope of the access to `self` here. The value yielded by `self.span`'s `yield` accessor is `~Escapable` and scope-restricted to the yield, so the coroutine cannot be resumed until all uses of `s` are done. One way this could work is to tie it to the scope of `s`, so that the coroutine would be finished as part of leaving that scope (which in this case is always done by returning from the function). The exact language rule here will be designed and reviewed as part of the lifetime dependency feature.

`self.span` yields a borrowed value, but assignment to a mutable variable generally requires an owned value, so the yielded span has to be copied as part of the assignment. This is fine because `Span` is a `Copyable` type.

However, a superficially similar mutating algorithm using `MutableSpan` would not compile. First, `MutableSpan` is not a `Copyable` type. But even more importantly, all of the mutation operations on `MutableSpan` require exclusive access to the span, which the algorithm fundamentally never receives because it gets a borrowed span from the `yield` accessor.

To allow this, Swift would need a copying read accessor that was implemented with a coroutine. (Perhaps this could be called `yield consuming`.) This is the counterexample to the logic laid out above in the section on the six fundamental accessors, and it makes perfect sense: the accessor is yielding an independent value that can only be used within the scope of the yield, a restriction that can be safely enforced because of the non-`Escapable` nature of the type.

### Effects

Swift has direct support for two kinds of function effect: error propagation (`throws`) and asynchronous suspension (`async`). Concrete accesses to memory don't normally result in either of these effects, but it can be useful to allow abstract accesses to have them, and programmers will generally understand what that means: the code that must be executed in order to perform the abstract access can have one or both of these effects.

For example, the `value` property of a `Task` has a `get` accessor that is both `async` and (potentially) `throws`. This reflects the fact that reading the value is an asynchronous operation that can block the current task, as well as the fact that the `Task` function can throw. (If the function cannot throw, the `Task` should use `Never` as its `Failure` type argument, and then Swift will know that reading `value` statically cannot throw, either.)

Currently, Swift only allows effects on `get` accessors, and only for immutable storage. There's no fundamental language design reason for this restriction; the language implementation simply doesn't yet support effectful coroutines.

A synthesized accessor must combine the effects of all of the accessors it uses. That is, if a computed property is implemented with an `async` `get` accessor and a `throws` `set` accessor, a synthesized `yield inout` (and modification accesses in general) is both `async` and `throws`. Error types would have to be unified using the `errorUnion` rule described in [SE-0413][]. This is not an issue in Swift today because there is no meaningful synthesis of effectful accessors given the restriction mentioned above.

### Key paths

In principle, key paths are meant to abstractly specify an arbitrary path of properties and subscript accesses. However, Swift gives programmers a great deal of flexibility about what can be expressed with accessors:

- Any of the four basic access kinds may or may not:
  - mutate or consume the base value,
  - throw an error, or
  - asynchronously suspend.

- If the value type of the key path is non-`Copyable`, there may or may not be any way to access it with a copying read access.

- The lifetime of a borrowing read or modification may be limited to a coroutine scope (q.v. `yield` and `yield inout`), or it may only be restricted to the scope of the access to the base (q.v. `borrow` and `inout`).

Furthermore, the key path value itself may have value limitations (such as [non-sendability][SE-0418]) because of captured subscript indices.

Each of these adds its own dimension of complexity for both the type system of key paths and the runtime support for them. Trying to support all possible combinations would become a huge drag on language development, and many of them would have minimal if any use in practice. Swift has instead generally taken the approach of only support certain combinations. For example, the distinction between `WritableKeyPath` and `ReferenceWritableKeyPath` is that the assignment and modifications are `mutating` for the former and non-`mutating` for the latter, and this reflects the default rules for properties on structs and classes, respectively. However, there is no `KeyPath` type that supports properties with `mutating` `get` accessors, and while this has doubtlessly occasionally been annoying to some users, it has largely been an acceptable compromise for the focus of language development.

The new accessor features discussed in this vision are likely to receive a similar treatment. The builtin `subscript(keyPath:)` storage declaration allows the path to be read with either a borrow or a copy. The key path runtime supports all four basic access kinds, but its borrow and modification accesses present a coroutine interface, and they do not support `async` or `throws` accessors. A storage declaration that cannot satisfy those requirements may remain indefinitely unusable in key paths. Specifically, that includes when:

- read accesses to the storage are `mutating`,
- any of the storage's accessors are `consuming`,
- any of the storage's accessors are `async` or `throws`, or
- the value type of the storage is non-`Copyable`.

Otherwise, it should always be possible to synthesize accessors that work with the key path runtime. For example, a `subscript` with `borrow` and `inout` accessors should still be usable in a key path; yielded values will simply lose the stronger lifetime guarantees of the original accessors and will be restricted to a coroutine scope when accessed through `subscript(keyPath:)`.

Allowing non-`Copyable` value types in key paths is a potential future direction, but it would require a careful approach. Abstract storage of non-`Copyable` type generally supports exactly one of being borrowed, being consumed, or being "copied" by creating a new value. We are very unlikely to want to enhance the `KeyPath` type system to express all of these alternatives. If Swift can force key paths of non-`Copyable` type to be only ever be read with a borrow, that would allow key paths to be created from stored properties of non-`Copyable` type. That is probably the best choice for expressivity, but it would rule out using key paths for storage that creates new values with a `get`.

## Observations and recommendations

Based on the above considerations, we can make a few useful observations about how these different accessors complement one another:

### Diversity is needed

No simple set of accessors covers all possible needs in the language.

`get` typically requires the value type to be copyable. `set` does not by itself, but it does require copies for modification accesses (which are very common) if it's the only available way to mutate the storage. Even when copying is possible, performing an unnecessary copy imposes significant performance burdens that are sometimes unacceptable.

`borrow` and `mutate` are very efficient for the cases they support, but they cannot generalize over all possible implementations, including cases as simple as mutable class properties. Even in fully concrete situations, there are good uses for finalization, like what `Dictionary.subscript` does in its `yield inout`.

`yield` and `yield inout` require the use of coroutines, which add significant performance overhead and limit how clients can use the value (by both restricting the access scope the value is available for and forcing clients to dynamically finalize the access).

### Some combinations are useless

There is no reason to explicitly define both `borrow` and `yield`. If `borrow` is possible to implement, you don't need to also implement `yield`.

There is no reason to explicitly define both `mutate` and `yield inout`. If `mutate` is possible to implement, you don't need to also implement `yield inout`.

### Address accessors are redundant with borrow/mutate

The `unsafe*Address` accessors should be unnecessary once we have safe `borrow` and `mutate` to replace them. But it may take a while to implement the latter, and the former are important tools in the interim.

### Roadmap

Combining the above, we expect that Swift will converge on a standard set of six accessors — `get`, `set`, `yield`, `yield inout`, `borrow`, and `mutate` — over the next couple of years.

In the interim, the `unsafe*Address` accessors will continue to be needed and useful for implementing many types of containers.

The legacy `_read` and `_modify` accessors will continue to be supported by the compiler for ABI compatibility but should not be used by new code.

## Recommended Usage

Based on the above, we can make some concrete recommendations for how each of these accessors should be used.

### Recommended sets of accessors

The above considerations allow us to suggest several recommended sets of accessors:

- A storage declaration that can't be directly modified and computes a fresh value every time you call it should just provide a `get`.

- A storage declaration that computes a fresh value on the first access but memoizes it in thereafter-immutable memory should just provide a `borrow`.

- A storage declaration that's trying to model an always-present stored component of a value type should just provide `borrow` and (if mutable) `mutate`.

- A storage declaration that's trying to model an always-present stored component of a reference type should just provide `yield` and/or `yield inout`, adding `get` and `set` if performance evaluations suggest it's important. See the section on synthesizing `borrow` and `mutate` for why reference types require coroutine accessors.

- A storage declaration that presents a transformation or wrapping of a value it stores internally should provide at least `get` and `set`. It should consider providing `yield` and/or `yield inout` if it can implement them more efficiently than they would be synthesized from the `get` and `set`.

- An abstract storage declaration (like a protocol requirement) that's trying to ensure the most efficient access possible and is willing to only allow implementations that represent an always-present stored component of a value type should just provide `borrow` and `mutate`.

- An abstract storage declaration (like a protocol requirement) that's trying to ensure the most efficient access possible without constraining its implementations should provide the full gamut of most-general accessors: `get` (if `Copyable`), `set`, `yield`, and `yield inout`. Providing `yield` and `yield inout` instead of `borrow` and `mutate` will restrict the scope in which the value can be used, but this is necessary when giving implementations enough additional flexibility that they might require dynamic finalization of the access.

### `get`

`get` should be used when the access computes a new value or is part of a resilient interface which might be changed to operate this way at some point in the future. Note that this works correctly even for non-copyable types when the value truly is returned fresh from each access and is not being stored locally.

Use `mutating get` when you need to implement a `get` operation that might cache the value or otherwise update mutable state on the containing object.

Use `consuming get` to model transmutation operations, where the original instance is being transformed into some other type, dissolving it in the process. One use case of this is unwrapping box types, where the returned instance is extracted from the box, destroying it in the process.

### `set`

`set` is a basic operation for all mutable storage declarations. It should generally be defined unless it is straightforwardly redundant with a `yield inout` or `mutate` operation.

If `mutate` is possible to define, `set` will usually be redundant.

If `set` is defined, it should usually be combined with a `yield inout` or `mutate` unless the definition of `yield inout` would be exactly the `get`/yield/`set` pattern that's automatically synthesized anyway.

### `unsafe*Address`

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

### `yield`

`yield` accessors are usually unnecessary when using value semantics. `yield` and `borrow` generally only work well when it's possible to borrow some value that's currently in storage. When you combine that with value semantics, `borrow` is usually both possible and preferred.

If the containing value doesn't store a value of the exact type, and so the accessor has to construct it from parts anyway, it's probably best to just return that with a `get`. In principle, you can imagine taking those parts, building them into the new value without copies, and then putting them all back at the end of the access, but this is generally not allowed in a `nonmutating` accessor because the access cannot be assumed to be exclusive.

`yield` accessors are necessary when borrowing out of mutable reference-semantics memory that has to be protected by dynamic exclusivity checks.

`yield` accessors can also theoretically be beneficial when something about the decisions above is dynamic: for example, it's possible to borrow the value directly out of memory, but only in some cases, and otherwise some kind of transformation applies. But in this case, it should almost certainly be combined with a `get` if possible so that copying read accesses aren't punished by the coroutine overhead.

The borrowed value yielded by `yield` is always scoped to a narrow, unique lifetime tied to the duration of the coroutine, rather than to anything that outlives the current access:

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

If the example above were implemented as a `get` operation, we could return the constructed non-copyable temporary value, but that would not give the `Foo` object any opportunity to clean up the temporary.

### `borrow`

When it becomes available, `borrow` will be the most efficient way to provide read access to values that either
- already exist in memory or
- will will be stored in memory by the end of the accessor (e.g. because the accessor computes the value and then memoizes it).

It is particularly important to provide `borrow` or `yield` access to values that either noncopyable or expensive to copy (for example, because they are large).  This includes values with generic type that might be either noncopyable or expensive to copy. Of these two options, `borrow` should be preferred whenever it is possible to use. Most generic data structures should provide access to their data via `borrow` accessors.

Compare how the example from the `yield` section would work with a `borrow` implementation:

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

Unlike the `yield` version, this form has no problems returning the borrowed value.  This requires a new “borrowing returns” language feature which would track the reliance on `foo` and ensure that `foo` was not modified for as long as the value borrowed from it was being accessed.

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

> Note:  Swift’s implementation of borrowed arguments uses “bitwise borrowing” in many cases, which avoids introducing extra indirection by directly sharing the representation of the value rather than just sharing a pointer. The goal is that ensure that borrowing is never less efficient than copying. We are exploring whether a similar idea can work for `borrow` accessors; abstraction may sometimes make it impossible.

### `yield inout`

Unlike `yield`, `yield inout` is broadly useful despite being a coroutine. `yield inout` and `mutate` allow modification accesses to potentially work on values in-place rather than requiring them to be copied. This is very important for many types, including data structures like `String`, `Array`, and `Dictionary`, as well as any other large or non-copyable type.

There are two principal exceptions where there's no reason to provide either `yield inout` or `mutate`:

- The value type is truly trivial to copy, such as a simple `Int`, `Double`, or `Bool`.

- The `yield inout` would just be equivalent to the `get`/yield/`set` pattern that Swift can automatically synthesize. This includes cases such as when the old value needs to be copied before the modification in order to compare values after the modification is complete.

The efficient pattern for transformed values that doesn't work with `yield` because it's a nonmutating method *does* generally work for `yield inout`, which is typically `mutating` and therefore does not need to concern itself with simultaneous accesses. `Dictionary` uses exactly this trick internally.

`mutate` is more efficient than `yield inout` when it's possible to define. However, when writing memory-safe code, `mutate` generally requires the memory being referenced to behave like a value-semantics component of the containing value type:

- the memory can only be accessed during some kind of access to the containing value, and

- it can only be mutated during a mutating access to the containing value.

If these two properties aren't true, then the only way to enforce exclusivity is dynamically, which precludes the use of `mutate` because the end of the access has to be dynamically tracked.

### `mutate`

A `mutate` accessor can be thought of as a “reverse `inout`”. In fact, some discussions have advocated using the term “inout” for this accessor.

As with `borrow`, this requires a new return capability that internally returns a reference to the stored value and tracks the lifetime of that reference against the lifetime of the containing value:

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
* `_read` / `_modify` have been implemented in the compiler for some time as an experimental form of `yield`/`yield inout`.  The underscored forms will continue to be supported in the compiler for the foreseeable future until all existing users have migrated.
* `yield`/`yield inout` will be the final standard form.  We expect to have these implemented in the compiler and ready for Swift Evolution review in the coming months.
* `unsafeAddress`, `unsafeMutableAddress` are implemented in the compiler today (with some limitations).  We do not expect to ever submit them for Swift Evolution review.
* We hope to have experimental implementations of `borrow` and `mutate` in another year or two and submit them for Swift Evolution at that time.

There are a number of open questions that will need to be resolved in the process of implementing the above:

* Protocols can specify `get` and `set` requirements for properties.  We will want them to be able to specify other accessor types as well.
* The current experimental `_read` and `_modify` implementation has some performance problems with certain types of generic code.  We will want to resolve this for the final `yield` and `yield inout` implementation.
* The `borrow` and `mutate` accessor implementations will require new return value conventions to be fully useful.

Of course, everything in this document is subject to community review through the Swift Evolution process.

[SE-0268]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0268-didset-semantics.md
[SE-0413]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0413-typed-throws.md
[SE-0418]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0418-inferring-sendable-for-methods.md
[SE-0446]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0446-non-escapable.md
[SE-0447]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0447-span-access-shared-contiguous-storage.md
[SE-0453]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0453-vector.md
