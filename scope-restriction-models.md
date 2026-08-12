# Scope and lifetime restrictions in Swift

This document tries to describe the current state of lifetime dependencies in Swift, make a case for switching to a new type-driven model of scope restrictions, lay out some of the complexity and future directions of the new model, and give a preliminary sketch for how the switch could be carried out.

## Introduction

### Temporarily available resources

It is common for resources to only be available to a program on a temporary basis.
This can apply to resources up and down the abstraction stack.
A high level resource might be something like an open HTTP connection, which can only be used after the connection is open and before it is closed.
A low level resource might be the right to use a particular piece of memory, which can only be used after the memory is allocated and before it is deallocated.
In any case, there's a point in the program's execution when the resource becomes available to use, and another point when the resource stops being available.

Safe programming languages must eliminate certain safety-implicating bugs[^1] relating to using resources when they're no longer available.
This is widely recognized for low-level resources such as memory; nobody should call a language safe if it is exposes pointers to deallocated memory.
However, it is often the case that the same techniques that work to establish safety for low-level resources can also be used for high-level resources.

[^1]: Or at least strongly discourage them, e.g. by only admitting them when `unsafe` features are used.
      Ideally, the language would encourage such features to at least still be used in patterns that can be proven safe outside the language.

There are many such techniques.
Automatic memory management works well for dynamically-allocated memory.
Non-copyable value ownership generalizes that to cover a host of other dynamically-managed resources.
But neither works for resources that are naturally tied to a scope in the program.

### Scoped resources

The most obvious resource that's naturally tied to a program scope is stack memory.
Small, fixed-size amounts of stack memory can be allocated at no dynamic cost[^2] within the function's ordinary stack frame.
Even when it needs to be done dynamically, stack allocation is also typically significantly faster than heap allocation.
But stack memory can be immediately reused once the current function returns, tying it to the outermost scope in the function.
Any use of the allocation after that point risks arbitrary memory corruption.

[^2]: Well, at the usually negligible cost of increasing the frame size and the ensuing locality loss for other stack allocations.

But there are other scope-associated resources that naturally arise.
These resources need not be as literal as a memory allocation.
Safe languages often use static analysis of the program's control flow to prove that certain things are done correctly.
Often, this takes the form of proving that one thing happens before another.
For example, Swift's [exclusivity rule][SE-0176] says that a variable cannot be mutated while a read of it is active.
Another way to put that is that there is a scope (the duration of the read) during which a resource (the right to read the variable without violating exclusivity) is available.
Since the variable is immutable within the scope, passing around this resource is essentially the same as passing around the value, but without any of the overhead of copying it.
This is an important tool for building efficient abstractions for managing complex operations, like iterating a collection.[^3]
Swift's exclusivity rule also says that a variable cannot be accessed while it is being mutated.
As above, this can be seen as a resource (the right to mutate the variable) that is only available within the scope of the mutation.
And just as above, this is an important tool for building efficient abstractions for managing complex mutating operations.[^4]
Similar ideas can apply to allow abstraction over almost any control-flow-based language rule.

[^3]: This was actually recently added to Swift as the `Ref` type, in [SE-0519][].
[^4]: This was also added in [SE-0519][], as the `MutableRef` type.

Furthermore, scoped resources give rise to more scoped resources.
If a value can only be safely used within a particular scope, then anything built on top of it must also only be safely usable within that scope.
For example, suppose that you have a function that breaks down an array into slices.
If the array you start with comes from a scoped resources, then the slices can be only be used within the original array's scope.
Suppose you instead present your API as an iterator type with a `next()` method that parses the next slice out.
Then not only are the slices restricted, but your iterator also needs to be restricted, since calling `next()` accesses the original array.

### Models of lifetimes

In Swift, we enforce safe access to scoped resources using [non-escapable types][SE-0446].
There's a natural intuition for what it means to have a value of a non-escapable type: the value cannot be used outside of some scope.
That scope is usually clear from context, and intuitive reasoning leads to a set of rules that any reasonable language feature for scoped resources would need to follow.
For example, if you have a parameter of non-escapable type, you cannot assign that value to a global variable, or to any location where you can no longer reason about whether the value has escaped.

But you cannot design language features by enumerating every code pattern that would use the feature and deciding intuitively how each should work.
Instead, you define general rules, which you can then validate against your intuition as part of the design process.
These general rules are called the formal model underlying the feature.
The core ideas of the formal model are predominantly responsible for determining how the feature works in the language.
The rest of the language design of the feature can usually be thought of as relatively superficial tweaks on top of those core ideas.[^5][^6]

[^5]: The rule of exclusivity defined by [SE-0176][] has an example of such a tweak.
      The general model of exclusivity is that, whenever you use a variable, there is a corresponding access scope, and conflicting access scopes (e.g. two mutations of the same variable) cannot overlap.
      But there is a small exception to the model that permits overlapping accesses to different stored properties of the same variable.

[^6]: Another example is the addition of [region-based isolation][SE-0414] to the core model of non-sendable types.
      This was a much deeper change, and one with significantly wider implications to the core feature.
      Even so, it had to be built in order to fit on top of the existing rules.
      Core model replacements are rarely strictly additive in this way, which is a large part of why they are difficult to do retroactively.

It is my contention that the current formal model we use for scope restrictions in Swift is not what it needs to be.
The current interpretation of non-escapable types makes it difficult to correctly handle certain kinds of abstraction over scoped resources because the scope restrictions do not propagate properly through types.
(This has some other knock-on effects.)
I believe that the limitations we've been imposing on non-escapable types up until recently have given us a window in which it still remains acceptable to change the model.
But we will close that window very quickly if we start adding generalizations over non-escapable types to the standard library.
We will regret doing so.

Now, the current model works fine for many simple, useful code patterns.
We do not need to withdraw any of the library features we've already released using non-escapable types.
All we need to do is continue to disallow certain kinds of generalization over non-escapable types until we can introduce the right model for them.

## The value-dependency model of scope restrictions

Swift's current formal model for scope restrictions is based on lifetime dependencies between abstract values.

An abstract value represents a computation that is performed (implicitly or explicitly) as part of evaluating a function.
For example, if the function contains a function call, there is an abstract value corresponding to the return value of that call.
At every point in a local variable or `inout` parameter's scope, an abstract value can be determined that represents how the value of the variable at that point was computed.
There are special abstract values representing "computations" such as parameters, initial values of `inout` parameters, final values of `inout` call arguments, and values that are computed differently on different control flow paths.

Every abstract value has a set of lifetime dependencies.
These dependencies are either (1) specific local access scopes or (2) "root" abstract values such as parameters or initial values of `inout` parameters.

The way that an abstract value is computed determines the relationship between its lifetime dependencies and those of the abstract values that it was computed from:
- Abstract values of escapable type, such as `Int`s, generally have no lifetime dependencies, superseding all other rules.
- Abstract values for parameters (including initial values of `inout` parameters) are roots and just have themselves as lifetime dependencies.
- Abstract values for most other computations generally have the same lifetime dependencies as their data dependencies.
  For example, an abstract value representing the read of a non-escapable stored property of a struct value has the same lifetime dependencies as the original struct value.
  Similarly, the result of a control-flow-merge computation carries the union of the dependencies of all of the different possible inputs.
- However, call results (return values and final `inout` argument values) are special: their dependencies are determined based on the dependency signature of the called function.
  This is described in more detail below.
The abstract value's set of lifetime dependencies is the solution of this system of set relationships.

Every function has a dependency signature as part of its type.
This signature describes the dependencies of the results, including the final abstract values of any `inout` call arguments.
Dependencies in a function dependency signature are sets of any of:
- the access scope of a function parameter, if it is a `borrowing` or `inout` parameter,
- the abstract value of a function parameter, if it has a non-escapable type, or
- the initial abstract value of an `inout` parameter, if it has a non-escapable type.
When determining dependencies for a call result or final `inout` call argument argument value, these rules are applied to the lifetime dependencies of the corresponding arguments.

For example, if the dependency signature says that the call result is dependent on the abstract value of parameter #4 and the initial value of `inout` parameter #6, then the lifetime dependencies of the call result abstract value are the union of the lifetime dependencies of the abstract value passed as call argument #4 and the lifetime dependencies of the current abstract value (at the point of the call) of the variable passed as `inout` argument #6.

The formal model then comes down to two restrictions:

1. Every use of a value that has a lifetime dependency on an access scope must occur within that access scope.
   (If the value has dependencies on multiple scopes, the use must occur within all of the scopes.)

2. The lifetime dependencies of abstract values corresponding to function results must be a subset of the corresponding dependencies declared in the function's dependency signature.

For example, suppose that the dependency signature for the current function says that the call result is dependent on the abstract value of parameter #4 and the initial value of `inout` parameter #6.
Consider the lifetime dependencies of the abstract value that is returned by the function.
It's okay if these dependencies are the empty set, or just the abstract value of parameter #4.
But if the dependencies include something not in the declared set, that is an error.

### History of the value-dependency model

This model arises naturally as an extension of several things that are already built into Swift.
Almost all of the rules for abstract values defined above fall out automatically from a basic data flow analysis of the function body.[^7]
This analysis is automatically performed by the compiler for every function and is required for a lot of existing language features and basic optimizations.
Similarly, the primitive value dependencies introduced by things like projecting out property addresses are very important to optimization, and the compiler has extensive support for working with the data dependencies introduced by these operations.
Using these tools, the lifetime dependency set of any particular abstract value can be computed by just finding the fixed point of the system of set inequalities described above for the abstract value.
There's a good reason we started with this model.

[^7]: Compiler developers generally refer to this analysis as putting the function into [static single assignment][SSA] form.
      Swift's "SIL" SSA representation does not normally model the special abstract values described above for `inout` parameters and arguments, though.

(This is not meant to downplay the amount of work that's gone into implementing the current feature.
The compiler does some very impressive things, especially around shrinking and extending scopes in order to avoid unnecessary violations of the model's restrictions.
And it's worth noting that almost all of that work would still be necessary if Swift switched to a different language model.
The analysis parts of it would just get consumed in a different way, and the rewriting parts should be exactly the same.)

## Problems with the value-dependency model

### Conflation of different scope restrictions

The current value-dependency model does not have the ability to distinguish different kinds of lifetime dependency.
This is unfortunate because values may naturally have scope restrictions for multiple independent reasons.

Consider a `Span` of non-escapable values.
There is a natural scope restriction associated with the borrow of the array that the span refers to.
The elements stored in that array also came from some scope, almost certainly a different scope.
So the scope restrictions are almost certainly different.
But in the value-dependency model, the span value must have the union of those dependencies, and so much any element extracted from it.
Putting the value into the span and then taking it out again has unavoidably lost information.
This greatly restricts what can be done with the element.

For example, suppose that you're writing an algorithm that works on a `Span` of values.
Your algorithm naturally wants to filter and rearrange the elements as part of its operation, but it can't modify the original span, and you don't want to pay to copy the whole thing (if you even can).
However, you can efficiently make a scratch array to hold `Ref`s to the interesting elements, since copying around a `Ref`s doesn't require copying its referent.
When you pull those `Ref`s originally out of the `Span`, they start out with the same scope restriction.
But if the scratch array is itself non-escapable --- for example, if it's temporary memory created by `withTemporaryAllocation`, a very efficient choice --- then any `Ref` pulled out of it will also carry a dependency on it.
That means Swift can't let you just return that `Ref` from your algorithm, even though there's no real-world problem with doing so.

There's an especially important special case of this: some values of types that are generally non-escaping actually do not require any dependencies at all.
A global constant can be safely borrowed for a scope that covers the entire duration of the program.
It can be useful to create collections of values like that, like arrays of statically-allocated strings.
If those collections can also be generated as global constants, then great, those collection values need no dependencies.
But if not, and the collection needs to be temporary, then the value-dependency model has no way to separate the global-ness of the elements from that temporary-ness.

This problem also has a huge impact on the ability of higher-order algorithms to usefully work with non-escaping values.
Consider a generic algorithm like the following, which calls a function for each element in a collection and returns the first non-`nil` result:

```swift
extension Collection where Element: ~Copyable & ~Escapable {
  func firstReturning<R>(operation: (borrowing Element) -> R?) -> R?
    where R: ~Copyable & ~Escapable
}
```

Here the algorithm has been generalized to permit a return value of arbitrary type.[^8]
Unfortunately, there's a problem with this.
The `operation` closure is a non-escaping function, as it should be, since `firstReturning` doesn't plan to escape it.
But that means that, when we call it within `firstReturning`, its return value is naturally going to gain a dependency on the closure.
This makes sense: after all, the closure certainly could capture something non-escapable and return a value dependent on it.
The dependency is necessary in order to conservatively model that possibility.
On some level, that's fine; it just means that `firstReturning` has to be declared with a dependency signature that says that the return value gains a dependency on `operation`.
But now clients are really restricted: the closure has to be kept around at least as long as the result of `firstReturning` is being used.
So you could not, for example, use `firstReturning` to implement a function like the following:

```swift
extension Collection where Element: ~Copyable & ~Escapable {
  func firstRefMatching<R>(predicate: (borrowing Element) -> Bool) -> Ref<Element>? {
    // The closure we pass as `predicate` is temporary in this function,
    // so when the `Ref` ends up dependent on it, it means we can no longer
    // return it out of this scope.
    firstReturning { element in predicate(element) ? Ref(element) : nil }
  }
}
```

[^8]: Swift's existing `Collection` protocol probably can't be retroactively generalized to support non-`Copyable` or non-`Escapable` elements.
       But we obviously want there to be *some* protocol that can do so, so drop that in instead.
       I'm just using `Collection` for familiarity.

Now, these examples can be changed to work around this problem.
A simple solution would just be to use indexes instead of `Ref`s.
For example, the scratch array of `Ref`s could just use a scratch array of indexes.
And `firstRefMatching` could just use `firstIndex(where:)` and then build a `Ref` directly to that element.
This does require more indexing operations, but that's probably not too high of a burden.
However, it also means that the values no longer stand alone.
For example, if you wanted to sort the indexes by their value, you wouldn't be able to use the standard `sort` function for `Comparable` elements; you'd have to use a comparator that has access to the original span.
Algorithms being generalized to work with non-escapable types would need to be rewritten in a less fluent and more boilerplate-y way, not because the existing implementation actually does anything that might escape the value, but just because the language model of non-escapable types is incapable of proving the correctness of the code.

Another solution would be to extend the current value-dependency model in ways that accommodate multiple sources of source restriction.
Several ideas have been explored for this, generally under the name "nested lifetimes".
Generally, the idea is that a type can declare one or more lifetime members that can be somewhat independent of the overall value's lifetime dependencies.
These approaches differ in their exact treatment of these nested lifetimes.

If the nested lifetime can be concretely constrained to a specific scope, this is essentially adopting a type-based model, at least for the nested lifetimes.
It is somewhat unclear how this would compose with the rules for local values, which would still be following value-dependency rules.
Values constrained by nested lifetimes can often become available as local values and vice-versa, e.g. when they are read out of a collection, and so it would be important to make them interact correctly.
For example, if you gain mutable access to a `MutableSpan` of non-escapable values that's stored in a collection, then inserted something into that span, it could add a dependency to the span.
That would need to be incorporated back into the nested lifetime somehow.
This could get very complicated, both in the theory and in the implementation.
It might be more reasonable to just simply switch wholly to a type-based model.

A different approach would be to make the nested lifetimes simply accumulate their own independent set of dependencies, as if they were separate values.
The dependency signatures of functions would be able to express changes in these dependency sets the same as they can express them for top-level values.
This is probably workable, and it would allow some more programs to be type-checked.
But most of the other problems with the value-dependency model would remain.

### Abstract propagation of scope restrictions

Another major problem with the value-dependency model is the way that scope restrictions propagate through APIs.
The value-dependency model defaults to being conservative about dependencies.
For example, return values are assumed to depend on all of the parameters to a function unless otherwise annotated.
This is appropriate in some cases; in fact, the type-based model applies essentially the same rule for functions that return a non-escaping type with unbound scope restrictions.
But it is quite over-conservative for common patterns of abstraction over non-escapable values.

Consider an `[Span<Int>]`.
When the programmer uses this value, there will be some kind of scope restriction that the elements of the array are collectively expected to have.
This may be an intersection of different scopes, since different elements may have different sources, but that kind of conservatism is inherent to the problem: we cannot reasonably expect Swift to track more specific associations through the abstract indexing API of `Array`.

In the type-based model, this scope restriction will be set (perhaps implicitly) on the argument type of `Array`.
It will therefore automatically and perfectly transfer by type substitution to everywhere in the `Array` API that refers to its `Element` type parameter.
This works out such that generic code that works with `Element`s does not actually need to reason about the specific scope restrictions of that type.
It can freely copy or move the value around wherever it likes (assuming the other constraints on the type permit that), as long as it doesn't statically erase the type (e.g. by wrapping it up as an `Any`).
This is because the generic function knows that the scope restriction in `Element` cannot refer to any of its own scopes.
That scope restriction was bound into the type by some function up the stack, using scopes meaningful in that function.
That function necessarily calls some function during that scope, or else the use would be in violation of its local rules.
Every function called subsequently, all the way down to the generic function, is therefore wholly contained within that scope, so the data flow of `Element` values no special need to be locally restricted within the function.
Any context that receives a value statically typed as `Element` (or terms of it) will maintain that same knowledge via type substitution of the original scope restriction.
Ultimately that propagates all the way back out to the original function that bound `Element`'s scope restriction to some locally-meaningfully scope.
Whenever that function works with an `Element`, type substitution will replace `Element` with a type containing the actual scope restriction.
Swift then simply needs to check that the value is only used within that locally-meaningful scope.

The result is that values of non-escapable types are only very lightly restricted by their non-escapability within generic functions.
As long as their types aren't erased, and they aren't used in some inherently escaping way (like being captured in an escaping function), they're free to be used exactly like escapable values.
There isn't any way for a function that just works with an `Element` to somehow impose extra scope restrictions on it, because the scope restrictions are bound immutably into the type.

That is not how it works under the value-dependency model.
Dependencies can get conservatively mixed up on essentially any function call.
It is up to the dependency signature of each specific function to only report the dependencies that it actually adds.
Any amount of preemptive caution or generality at any level risks permanently losing information by adding unnecessary dependencies.
This is a somewhat fraught programming model.
It is not in any way unsafe, but any failure to minimize dependencies can bring the whole house of cards down and make it impossible to write algorithms that have no business being forbidden.

The `Iterable` protocol proposed by [SE-0516][] provides an excellent example of this.
The protocol defines an `IterableIterator` associated type which is allowed (actually expected) to be non-escapable.
It also defines an `Element` associated type, which we would also like to allow to be non-escapable in order to support collections of non-escapable values.

Now, the protocol requires an `Iterable` value to have a `makeIterableIterator` method.
Whenever you call this method, you must borrow the collection value, and the iterator it returns should only be used within the scope of that borrow.
The value-dependency model has no problem expressing this scope restriction.
Different calls will be restricted to different borrow scopes.
Nothing about the type of the collection has anything to say about the scope of the iterator, nor should it.

The iterator thus produced is now required to have a `nextSpan` method that returns back a `Span<Element>`, and this is where the model runs into a problem.
What is the scope restriction of the elements of this span?

In the current model, lacking nested lifetimes, there is only one option.
The elements must shared the same dependencies as the span.
Since we want to allow the span to be temporary to the current call to `nextSpan`, the elements are also restricted to that call.
This means that elements cannot safely be persisted across calls to `nextSpan()`.
When the iterator is used for a `for` loop, this means that elements cannot be stashed between iterations of the loop, much less stashed outside of the loop.
This is extremely restrictive and applies even if the element type is known to be copyable.

Essentially, this option is permitting the iterator to not just synthesize a span in temporary memory in the iterator, but to synthesize the elements of that span in some way that depends on temporary memory in the iterator.
This seems admirably general, but it takes some effort to imagine a collection type that could take advantage of the additional flexibility.
A permutation iterator, maybe, where the elements are spans of values.
The cost of this generality is that iteration over anything approaching a normal stored collection of non-escapable values cannot use `Iterable` without facing heavy restrictions.

Now, still under the current model, a refinement of `Iterable` could say that the spans returned by `nextSpan` depend only on the current iterator value.
(This would need an explicit annotation.)
The iterator value will generally only directly depend on the borrow of the collection performed for the call to `makeIterableIterator`.
Thus both the spans and the elements will depend on that borrow scope.
This is a significant generalization in some ways.
Multiple spans can be used at once, and so can elements from different spans.
This permits some amount of element data flow between iterations.
But it's still the case that elements are restricted within the borrow of the collection, even if they could reasonably just be copied around as independent values.
So this refinement still heavily restricts how the element values can be used.

To do better, we would need nested lifetimes.
With these, we can give collections, iterators, and spans a nested lifetime corresponding to the elements.
In the signature for `makeIterableIterator`, we need to say the iterator's element lifetimes depend only on the collection's.
Similarly, in `nextSpan`, we need to say that the the element lifetime of the span depened on the iterator's element lifetime.
Finally, in `Span`, we need to say on element accessors (like the `subscript`) just returns a value that depends on the span's element lifetime.
With this careful set of annotations, and a protocol refinement devoted to the purpose, we can make sure that element lifetimes propagate correctly for this specific API --- when we can statically make use of the refinement.

When I compare this to the simple guarantee afforded automatically to all generic code by type substitution, I feel that this is a significant loss in usability for collections (and other abstractions) with non-escapable elements.
And the best case here already relies on significant extensions (and widespread adoption thereof) beyond what the current value-dependency model is capable of expressing.

### Lack of explicit local annotations

Another disadvantage of the current value-dependency model is that it provides no direct way to state the expected scope restrictions on a specific value.
A function's dependency signature can say that its return value is dependent on a particular parameter.
However, there is no equivalent way to say within the function that the value in a local variable can only depend on that parameter.
There is no direct translation of such a statement in terms of the model; it would require an additional rule.
This is unfortunate, because while I expect that most programmers will usually be happy to just rely on the implicit inference of scope restrictions, there are some good if situational reasons to sometimes be more explicit.

First, explicit annotations can make it easier for people to learn and teach the scope restriction rules.
When programmers feel comfortable with a language feature, they often really appreciate inference rules and syntactic sugar that make the feature less obtrusive and heavyweight.
But newcomers to a feature often really appreciate the ability to spell these things out.
Some programming teachers even make that a class rule, forcing their students to write out things like type declarations in assignments to help them internalize what the compiler is doing automatically for them.

Second, explicit annotations provide a natural way for tools to communicate inferred scope restrictions back to programmers.
Consider a programmer who's running into a mysterious problem with their code; maybe they've written a `Span` algorithm that's somehow crashing.
It's reasonable for them to ask their IDE what the lifetime of the span actually is.
Many IDEs already have similar features for e.g. spelling out the inferred type annotation when you hover over a reference to a variable.
But those features are generally designed around making things explicit that can actually be written in the language.
Without the ability to express such a restriction in the language, the IDE has to find some other way to communicate it (maybe as a prose comment), and the programmer has no way to double-check that the inferred annotation is actually correct.

Third, explicit annotations can help programmers narrow down the cause of a compiler error.
In an ideal world, of course, compiler diagnostics would always point you exactly at the line of code that you need to fix.
In practice, this can be difficult.
Lifetime errors often arise because of a conflict: a value has a lifetime dependency that it shouldn't have at the point where it's used.
The compiler cannot know whether the problem is that the value is being used wrong (it should be okay that it has that dependency) or defined wrong (it shouldn't have that dependency).
By being explicit about their assumptions, programmers can move diagnostics from the use site to the points where the assumption was violated.

For example, consider code like this:

```swift
1  var span: Span<Int>
2  if useGlobalArray {
3    span = globalArray.span
4  } else {
5    span = localArray.span
6  }
7  ...
8  saveSpan(span)
```

Suppose that `saveSpan` requires the span it's passed to be immortal, which is to say, to have no lifetime dependencies.
A span over the global constant `globalArray` fits the bill, but a span over the local variable `localArray` does not.
A perfect diagnostic might report on line 8 that `span` is not necessarily immortal like it's required to be, together with a note on line 5 that it won't be immortal if it's computed this way.
But compilers don't always deliver perfect diagnostics, and it's not hard to imagine that the compiler might sometimes emit this error without the extra note, leaving the programmer to figure out why the span isn't immortal for themselves.
However, if the programmer can add an annotation to `span` saying that it's expected to be immortal, then the compiler will stop reporting the error on line 8.
Instead, it will report an error on line 5, when a non-immortal span is assigned into a variable that requires something immortal.
(Of course, this isn't an excuse for compiler developers to not still try to emit the better diagnostic.)

Finally, explicit annotations can help to enforce correctness when interacting with an unsafe interface.
In safe code, the compiler will analyze both the uses of a value and how it's defined.
This creates a complementary balance: the narrower the scope restriction that the compiler infers for the value, the more restricted the uses of the value will be.
But with unsafe code, the compiler often just has to trust one side or the other, eliminating this balance.
An explicit annotation can make sure that the compiler still enforces the assumptions that the unsafe code requires.

For example, a C function might need to be passed a pointer that's valid for a specific duration in order to behave correctly.
It's a C function, so it just takes an unsafe pointer; the lifetime restriction requirement is a documented requirement, not something that's going to be automatically enforced by Swift.
Now suppose that some safe Swift code computes a span with the goal of passing that span to the C function.
Without an annotation, a bug in that computation can result in a span with a narrower than expected scope, silently causing the pointer to not meet the documented restriction.
But an explicit annotation of the required scope of the span prior to extracting the pointer from it will not just document the expectation in source, it will actually enforce it: the compiler will object if a too-narrow span is ever assigned to the explicitly-annotated variable.

### Flow-sensitive diagnostics for invariant lifetime requirements

This last disadvantage is significant enough to be worthy of inclusion.
I will readily acknowledge that it is less important than the others, though.

The value-dependency model always associates lifetime dependencies with specific abstract values.
When there's a restriction on the lifetime dependencies for some mutable variable, the model is not generally going to enforce that restriction as an invariant on the variable.
Instead, it's going to enforce it specifically on the abstract value of the variable at some specific point.
This is a more control-flow-sensitive rule and can easily lead to diagnostics that seem misplaced.

As an example, consider a function with an `inout` parameter of non-escapable type.
Suppose that the dependency signature of the function says that the function does not add dependencies to the parameter; that is, the final value in the `inout` parameter must have no more dependencies than the initial value.
This is very common: it is in fact the default rule for `inout` parameters.

Now suppose that, within the body of the function, there is a change to the parameter (let's suppose it's an `insert` call) which adds a dependency to its value.
The lifetime checker will not generally be able to diagnose this immediately at the `insert` call, because it does not enforce the restriction on the parameter as an invariant.
Instead, it will just update its internal tracking to record that there's a new dependency and continue onwards.
In some ways, this is arguably good.
The checker is allowing the value in the parameter to subsequently change, and if it changes to a value with the original dependencies (or less), the postcondition on the parameter will be satisfied and there's no reason to diagnose.

But the consequence of not enforcing the restriction as an invariant is that the diagnostic becomes sensitive to control flow.
Assume for the sake of argument that the value *isn't* restored to something with fewer dependencies.
Then a fully precise statement of the error is that *there exists a control flow path leading to an exit from the function which leaves a value in the `inout` parameter with too many dependencies*.
This is fundamentally harder to diagnose than just pointing at the `insert` call, like it could if it were enforcing an invariant on the parameter.
The analysis has to walk the control flow of the function, and it is likely to only detect the problem when that walk actually reaches the exit.
It might reaonably just emit the diagnostic there without even noting the call that added the dependency.
Even if it does point out the `insert` call, it has to also point out the path that led to the exit.
After all, the bug might not be that `insert` was called; it might just be that the value was expected to be reset later.
It is just fundamentally harder to provide a good, concise diagnostic under this rule.

## Advantages of the value-dependency model

None of that is to say that there aren't upsides to the value-dependency model.

Probably the biggest advantage is that it builds relatively directly on top of existing analyses in the compiler.
This is, after all, what has allowed the feature to be delivered so far.
And many use cases of non-escapable values do not run into any of the difficulties around abstraction that I've laid out above.
If you just want to support working with spans of trivial values, with minimal abstraction, you don't need much from the lifetime system.

A more fundamental advantage arises from flow-sensitivity.
In type-based lifetime models, it is still generally true that any given variable has a single type for its duration.
The lifetime constraints in this type can be inferred "globally" within the function, as discussed later, but ultimately the scopes will be picked and applied everywhere the variable is used.
This means that the checking ends up being very conservative if the lifetime restrictions of the variable significantly change over the course of the function.
For example, suppose that you have an array of spans:

```swift
  var spans = [RawSpan]()
  use1(spans) // might require an unrestricted array
  spans.append(longTermArray.span)
  use2(spans) // might require a broadly-restricted array
  spans.append(shortTermArray.span)
  use3(spans) // must work with a narrowly-restricted array
```

In this example, the lifetime constraint on the elements of `spans` gets progressively narrower as the function executes.
It's possible that `use1` and `use2` could take advantage of the broader lifetime restrictions that still hold at the time of those calls, allowing this code to pass lifetime checking when a type-based analysis would reject it.
However, this is unlikely.
Most code that builds up a collection this way does not have interleaved uses of the collection; there tend to be distinct phases.
Moreover, the uses are likely to be uniform, meaning that if they work with the narrow restriction at the end, they would also work with an artificially narrow restriction at the beginning.
And if the programmer really needs this, they can assign the array to a new variable, allowing the compiler to infer a broad scope for the first variable and a narrow scope from the second.

If we find it too useful to lose, this kind of flow-sensitive typing can still be supported under the type-based model.
Rather than modeling the local declaration as having a single type, we would allow each use of it to observe a different type.
A similar data flow analysis as to that currently performed for the value-dependency model would then relate the type at each observation to the types at previous observations.
Ultimately this would just factor into the system of scope relationships that must be solved, here inferring bound types at each observation rather than globally for the local declaration.
This flow-sensitive analysis would only be used if the variable lacked an explicit scope specifier.

## The type-based model of scope restrictions

Let's go back to the basic problem.
The core correctness property we'd like to allow to be proven is that certain values are only used within appropriate scopes.
Static type systems are designed to restrict how values can be used.
It's reasonable to ask if these scope restrictions could instead be expressed in types.

Now, we have an existence proof that this can work, because this is exactly what Rust does.
It does have some challenges and complexities, which I'll discuss below.
But it also allows a lot of things to be expressed that we don't know how to express in the value-dependency system, allowing a lot of basic generic expressivity over non-escapable types.
And it's demonstrated a reasonable ability to evolve over time, as Rust's lifetime system has been gradually expanded over the years to allow for more things.
Adopting a type-based model does not require immediately adding every feature of Rust's lifetime types system; Swift can decide where to draw the line, release by release, and if we think a particular generalization is not worth the implementation cost, we don't have to add it.

It also does not require adopting Rust's syntax for lifetime qualifiers.
As I go through the language, I will sketch out what I feel is a workable and more Swift-like design.
Of course, there are many other options.
I am providing a concrete syntax primarily to elucidate the text.

### Concrete scope restrictions

Any given non-escaping type has a set of concrete scope restrictions.
These are the scopes that we need to be able to talk about in order to properly restrict the use of values of the type.

Most non-escapable types have at most one such restriction.
As I'll discuss soon, scope restrictions associated with generic arguments are handled differently.
Therefore, a type only needs a concrete scope restriction if it is unconditionally non-escapable.
It only needs multiple concrete scope restrictions if it has multiple independently-scoped reasons why it's unconditionally non-escapable.

So let's sketch out a syntax for specifying within a value's type that the value is restricted to a specific scope:

```swift
// We'll talk about what goes in the parentheses later.
var span: @scoped(array) Span<Int>
```

Whenever we have a value of such a type, the type must have a scope restriction.
Of course, we would normally want that scope restriction to be inferred:

```swift
// We'll talk about how the `@scoped` restriction can be inferred later.
var span: Span<Int>
```

But it can always be spelled out.

This syntax assumes that there's exactly one concrete scope restriction associated with the type.
That's true for `Span`.
As long it's true, we don't need any way to name different scope restrictions, or to declare names for them in the type.
However, it's not true for all types.
If it were only false for really weird types, we could probably reasonably subset it out of the language, at least to start.
Unfortunately, it does come up quite easily with types that just store multiple non-escapable values, like the following:

```swift
struct SpanPair<T>: ~Escapable {
  let left: Span<T>
  let right: Span<T>
}
```

Stored properties are types of values, so the concrete scope restrictions do need to be given in these types.
We could add a syntax for declaring named scope restrictions, like so:

```swift
struct SpanPair<T>: ~Escapable {
  scope _left
  scope _right

  let left: @scoped(_left) Span<T>
  let right: @scoped(_right) Span<T>
}
```

But I don't think this is actually required.
(It may be necessary to express more complex cases, but it can probably be subsetted out of the language to start.)
Instead, I think we can simply infer anonymous scope restrictions to "fill in" all the unspecified scope restrictions on stored properties.

We do need to extend the `@scoped` attribute (or whatever the syntax ends up being) to allow multiple scope restrictions to be specified:

```swift
var pair: @scoped(left: &array1, right: &array2) SpanPair<Int>
```

Here I've just allowed `@scoped` to take multiple name/scope pairs.
Names can resolve to properties of non-escapable type, which provides a natural way to specify the otherwise-anonymous scope restrictions created for stored properties.
If there's a single "pair", and it doesn't actually have a name, the type needs to only have a single scope restriction.

Note that this model hasn't actually added any syntactic burden to the definition of the non-escapable type so far.
We've just found a reasonable interpretation of the existing code that lets us propagate implicit scope specifiers on types.
We've only gained the option of being more specific.

So how do you specify a scope?
This is definitely a place we can expand over time.
What I think we clearly need at start is at least:

- The name of a variable of non-escapable type, from which we take its concrete scope restriction, e.g. `@scoped(otherSpan) Span<Int>`.

- Some syntax for specifying the scope of a `borrowing` or `inout` parameter (not necessarily of non-escapable type), e.g. `@scoped(&self) Span<Int>`.

- Some syntax for specifying the global scope, e.g. `@scoped(immortal) Span<Int>`.

To this we could gradually add member paths, intersections, scopes of local variables, and so on.
We'll probably need to support most of those in the implementation right away --- they can come up in scope inference very easily --- but we don't necessarily need user-facing syntax for them.

### Bound and unbound non-escapable types

We've said that the types of values must always specify any concrete scope restrictions in the type.
Such a type is said to be *bound*.
But not every place you can write a type is immediately the type of a value.
Forcing every non-escapable type to always be bound to a specific scope restriction would rule out a lot of useful code patterns.

This is straightforward to see with a simple `typealias`:

```swift
typealias ISpan = Span<UInt32>

func slice(span: ISpan) -> ISpan { ... }
```

If all references to a non-escapable type had to bind the type's scope restrictions, every use of `ISpan` would share the same scope restriction.
That's probably not want the programmer wants, though.
If that's what they wanted, they could've written the scope restriction explicitly in the `typealias`, like so:

```swift
typealias ISpan = @scoped(immortal) Span<UInt32>

func slice(span: ISpan) -> ISpan { ... }
```

What they probably want is for writing `ISpan` to behave just like an abbreviation of writing `Span<UInt32>`.
In other words, they want the reference to `Span` in the alias to stay unbound.
When they use `ISpan` in a context that requires scope restrictions to be bound, they can provide the scope restrictions themselves or just allow them to be decided from context exactly as if they'd written `Span<UInt32>`.

This divide between bound and unbound types is very important in the type-based model.
That's especially true for abstract positions like generic parameters and associated types.
With `typealias`es, Swift can really muddle through well enough using any rule; the compiler can always locally choose to look through the `typealias` or throw away scope restrictions if it helps make reasonable code compile.
With the generic positions, Swift really needs to know what the programmer wants, because it's not reasonable for the compiler to do some global analysis of how a generic parameter is used to figure it out.

`Iterable` provides an excellent example of a protocol that requires both:

```swift
protocol Iterable<Element>: ~Copyable, ~Escapable {
  associatedtype IterableIterator: ~Copyable & ~Escapable
  associatedtype Element: ~Copyable & ~Escapable
  where Element == IterableIterator.Element

  func makeIterator() -> IterableIterator
}
```

The `Element` associated type almost certainly ought to be a bound type.
This would rule out some largely theoretical conformances --- types that synthesize non-escapable elements during each call to `nextSpan`  --- while promoting a very clean, unrestricted programming model for standard conformances when they're generalized to support non-escapable elements.
This is because bound types provide a very straightforward generic model.
The bound scope restriction must be derived somehow from the type of `self`, which means the scope in the restriction must always be broader than the current function call.
Any context that expects a value of the bound type will have its own contextually-equivalent understanding of the scope restriction expressed in the element type.
This all means that generic code can usually move and copy values of bound types around freely, subject only to fairly minor restrictions, like not doing truly escaping things like e.g. wrapping them up as an `Any`.
That's a very strong and desirable property for collection elements in generic code.

In contrast, unbound types tend to end up with highly conservative scope restrictions that make them dependent on specific calls and accesses rather than allowing broad data flow limited only by the type.
This is necessary for types like `IterableIterator`, where the protocol really does need clients to infer a different scope restriction for every unique call to `makeIterator`.

Since there are use-cases for both, it's necessary to have a syntax for declaring generic parameters and associated types as either.
At the moment, I believe that the right default is for these positions to expect an bound type.
Unbound types therefore ought to be called out when they're needed.

I'm not sure what the right syntax for unbound types would be.
Considered in full generality, unbound types aren't merely "unbound" as binary flag: there's a whole expected signature for the scope restrictions that need to be applied to them.
This is effectively a very restricted form of higher-kinding in the type system.
I would guess that we don't really need to support more than the simple pattern of a single scope restriction, though.
An attribute like `@unscoped` on the generic parameter or associated type might be fine for that.

### Function signatures

Parameters and results of functions are types of values, so if they have non-escapable type, those types must have concrete scope restrictions.

Function declarations involving non-escaping types will generally have some number of scope parameters.
Usually these will all be implicit, and we can probably start with that.
It's going to be beneficial for the explanation if I can write them out, though, so I'm going to invent a syntax.
I can't think of a better choice offhand than writing it like a generic parameter:

```swift
func returnEither<scope a, scope b>(spanOne: @scoped(a) Span<Int>,
                                    spanTwo: @scoped(b) Span<Int>)
         -> @scoped(a & b) Span<Int>
```

Just like we did with type definitions, we can infer all of this by default:

```swift
func returnEither(spanOne: Span<Int>, spanTwo: Span<Int>) -> Span<Int>
```

The way this works is straightforward.
Since parameters have to have concrete scope restrictions, and the programmer hasn't given us one explicitly for `spanOne` or `spanTwo`, we synthesize new anonymous scope parameters to fill them in.
The compiler then applies heuristics to find a scope restriction for the return type, just like it does today under the value-dependency model.
That ends up being an intersection of the two anonymous parameter scopes.

If the programmer needs to take control, they won't usually have to write out explicit scope parameters.
In most cases, they should be able to just name the value parameters:

```swift
func returnFirst(spanOne: Span<Int>, spanTwo: Span<Int>)
         -> @scoped(spanOne) Span<Int>
```

An explicit scope parameter might be necessary in more complex cases:

```swift
func returnASpan<scope s>(span: Span<@scoped(s) Span<Int>>)
       -> @scoped(s) Span<Int>
```

Note that this is already not expressible in the value-dependency model without losing information through dependency conflation.
I think subsetting this capability at first would probably be fine.

Swift expresses ownership on function parameters and (in some cases) results as its own built-in concept.
(This is in contrast to Rust, in which non-consuming ownership is universally expressed with `&` types.)
In some situations, it may be necessary to express scope restrictions directly for the borrow / `inout` access scopes associated with those parameters.
For example:

```swift
extension Collection {
  // The scope restriction lets us indicate that the borrow of the element
  // passed to `visitor` is valid for exactly as long as the borrow
  // of `self` performed for the call to `visit`. Otherwise, the closure
  // will only be able to assume that the borrow is valid for the scope of
  // the current call to the closure. (That is, the element could be
  // temporary to the implementation of `visit`, rather than borrowed
  // directly from the collection.)
  //
  // This conservative assumption would be the default (and therefore
  // sometimes need to be overridden) under any model.
  //
  // Note that the narrowness of the value ownership of the element is
  // orthogonal to the issue we discussed previously about the scope
  // restrictions in the Element type. This annotation is useful even if
  // Element is an escapable type.
  func visit(visitor: (@scoped(&self) borrowing Element) -> Void)
}
```

### Inferring scope restrictions

The uses of non-escapable types in a function under the type-based model naturally create a system of scope equalities and inequalities.
Inferring scope restrictions essentially involves solving this system.
A lot of the logic of that should be very similar to the process of solving the dependency-set relationships introduced by the scope-dependency model.
However, there are some significant differences.

The first difference is that, in the value-dependency model, variables in the system are associated 1-1 with specific abstract values.
In the type-based system, variables are introduced mostly for declarations and calls.
A use of a function or type that's generic over scope restrictions essentially "opens" that entity's signature, creating fresh scope variables in the solver.
Explicit scope specifiers on types can immediately resolve some of these variables, but the rest must be inferred through solving.

The second is that, in the type-based model, type substitution can create complex equality relationships between different parts of the system.
However, it is still the case that the system is purely conjunctive.

And finally, the type-based model must reason about scope variance relationships between various types.

### Scope relationships and scope variance

Scopes have natural relationships with each other: a scope can be a *subscope* of another, meaning that its duration is contained within the other's duration.
Two scopes can also always be intersected, and the resulting scope is a subscope of both of the original scopes.
Mathematically, we can say that scopes form a bounded meet-semilattice, with the immortal global scope as the greatest element.

In principle, the intersection of any two scopes might be empty, if they do not overlap at all.
But this is not a useful result, because a programming language is only interested in the values that are usable at a particular point in the program.
Every scope that can be used at a particular point should contain that point, and so the intersection between such scopes must also contain that point.
Generally, if a scope intersection would end up being empty, it must be an error.

#### Natural subtyping of scope restrictions

This relationship between scopes also implies a relationship between types that carry concrete scope restrictions.
Suppose that `smallScope` is a subscope of `bigScope`.
A span that's restricted to `bigScope` can safely be used as if it were restricted to `smallScope` instead.
Code that satisfies the narrower restriction of `smallScope` automatically satisfies the looser restriction of `bigScope` at the same time.
Another way of thinking about this is that knowing that the span is valid for all of `bigScope` provides strictly more information than only knowing that it's valid for `smallScope`.
These properties make `@scoped(bigScope) Span<T>` a natural subtype of `@scoped(smallScope) Span<T>`.
Converting a value of the former to a value of the latter removes information, but in a way that remains safe.

Because `@scoped(a) Span<T>` is a subtype of `@scoped(b) Span<T>` when `b` is a subscope of `a` (rather than the other way around), `Span` is said to be *contravariant* in its scope parameter.
Almost all unconditionally non-escapable types are contravariant in their scope parameters.
This is because the scope parameters of these types generally encode restrictions on where the type's values can be used, and it's always safe to use a value in a narrower scope.
(Other forms of variance even over immediate scope parameters are possible; I will discuss them later.)

#### Variance of scope restrictions

This subtype relationship between types that arises from scope relationships carries over to generic types with non-escapable type arguments.
Generic types can be covariant, invariant, or contravariant in their non-escapable type parameters.
Type variance typically follows the following rule:
- if a `G<T>` only *produces* values of type `T`, it should be covariant over `T`;
- if it only *consumes* values of type type `T`, it should be contravariant over `T`; and
- if it both produces and consumes values, it must be invariant over `T`.

In particular, types with value semantics are naturally covariant in the types of their component values because a value can only *produce* its component values.
For example, the tuple type `(T, Int)` is a subtype of `(U, Int)` if `T` is a subtype of `U`: all you can ultimately do with such a tuple is break it into its `T` and `Int` elements, and then those elements can be safely used as if they had type `U` and `Int` respectively.
The same logic applies to a struct `G<T>` that just has a stored property of type `T`.
It also extends to types like `Array<T>` that logically behave like collections of `T` values, and even to types with immutable reference semantics like `Ref<T>` and `Span<T>`.
But types that store `T` values with mutable reference semantics, like `MutableRef` and `MutableSpan` can both produce and consume values of type `T`; they therefoer must be invariant over `T`.[^9]

[^9]: Java famously gets this wrong: its built-in array type `T[]` is covariant in `T` despite having mutable reference semantics.
      This means that, for example, code that converts a `String[]` to `Object[]` and then inserts a non-`String` object will successfully type-check.
      The soundness of the type system has to be maintained dynamically with a runtime type check whenever you insert into an array.

(Contravariance is relatively uncommon because it is rare to have an abstraction that only consume values of a given type.
But it does happen with, say, a function value that takes a parameter of type `T`.)

#### Examples of variance

Let's look at that with some concrete types.

Suppose I've got an immutable value of type `UniqueArray<@scoped(bigScope) Span<T>>`.
Whenever I pull an element out of this array, it's a `@scoped(bigScope) Span<T>`.
As we've already discussed, it's safe for me to instead use this element as a `@scoped(smallScope) Span<T>`.
And the only thing I can really do with the span, in terms of its elements, is pull those elements out.
I can't add new elements to it because I've only got an immutable value.
So it's actually fine for me to treat this whole array as if it were a `UniqueArray<@scoped(smallScope) Span<T>>` instead.
This is what we mean by covariance.
(This conversion would be a problem if spans with different scope restrictions had real differences in their in-memory representation.
But they don't: this whole system relies on all the scope restrictions being erased at runtime.)

Now suppose instead that I've got a `MutableRef<@scoped(bigScope) Span<T>>`.
If I read the span out of this reference, I get a `@scoped(bigScope) Span<T>`.
I can safely use that span as a `@scoped(smallScope) Span<T>`, so you might think that it'd be okay to allow the ref to be converted to `MutableRef<@scoped(smallScope) Span<T>`.
But this is not safe, because it would let me write a `@scoped(smallScope) Span<T>` into the reference.
If I do, then any code that subsequently reads from the referenced memory and expects it to hold a `@scoped(bigScope) Span<T>` will become unsound.
The scope restrictions in the element type cannot be allowed to change.
This is what we mean by invariance.

#### Usefulness of variance

Note that I'm describing abstract, natural rules of value subtyping here.
Swift does have a general system of value subtyping, conversion, and variance that applies to some types.
This system is what permits, e.g., a value of type `Int` to be implicitly converted to type `Int?`.
However, most types cannot participate in this system; Swift special-cases it for a handful of fundamental and library types.
This is because, in general, subtypes might have a different in-memory representation from their supertypes.
Applying these natural subtyping and variance rules to an arbitrary user-defined type would therefore require recursively transforming values of the type.
This would be complex, expensive, prone to source-compatibility problems, and typically not very useful.
But subtyping arising from scope restrictions is different because it just results in use restrictions and never affects in-memory representations.
We very badly want the implementation to be able to use an *erasure* rule where the scope restrictions have no runtime representation.
Since the in-memory representations don't change, the subtype conversion has no runtime effect and therefore no compiler complexity beyond checking the correctness rules.

Moreover, subtyping in scope restrictions is very convenient because it implicitly gives programs a lot of flexibility about the exact scopes involved.
For example, consider a function that takes two spans and returns a slice of one of them, chosen dynamically.
Subtyping of scope restrictions naturally allows the spans to come from different places; the return value just has to be conservatively restricted to the intersection of the two scopes.[^10]
Without subtyping, this would have to be rejected.

[^10]: This intersection doesn't even need to be explicit.
       The function can simply say it takes two spans of scope `s` and returns a span of scope `s`.
       If the caller passes two spans of different scopes, the compiler will have to find a common supertype between them.
       This will be a span with the intersection of those two scopes.

In addition to being useful as a way to accept more valid programs, scope subtyping is specifically important for achieving expressive parity with Swift's current value-dependency model.
When the compiler is checking whether a function's implementation satisfies its dependency signature, it permits values to have fewer dependencies than the signature allows.
Translating such a situation into the typing constraints introduced by type-based scope restrictions introduces a type mismatch that requires scope subtyping to resolve.
Providing at least simple value subtyping is therefore necessary in order to continue to accept programs that the current model easily accepts.

#### Invariance and covariance of immediate scope parameters

As mentioned above, almost all unconditionally non-escapable types are naturally contravariant over their scope parameters because the parameter represents a restriction on where the value can be used.
When the parameter represents something else, however, other variances are possible.
Typically this arises when the scope constraint is applied to a value that is not just a value-semantics component of the type.

For example, consider a type that stores a mutable reference to a span:

```swift
struct SMR<T> {
  let ref: MutableRef<Span<T>>
}
```

We have two levels of scope restriction here, which we can make explicit:

```swift
struct SMR<scope r, scope s, T> {
  let ref: @scoped(r) MutableRef<@scoped(s) Span<T>>
}
```

`SMR` is contravariant in the scope parameter `r`: it's always fine to use the ref as if it were constrained to a narrowing scope than it actually is.
But it must be invariant in the scope parameter `s`, because narrowing this scope would allow a span with a narrower scope to be written into the reference.

### Standard library Scope type

There are certain situations where it would be useful to be able to explicitly specify a scope when calling a function.
For example, `Span` has an initializer that accepts an `UnsafeBufferPointer`.
Currently the resulting span value has no dependencies and is therefore somewhat treacherous to use, since there's no way to tie it to the scope in which the buffer is presumptively valid.

The syntax discussed up to now has no way to explicitly provide a scope argument to a function.
In fact, Swift generally has no way to explicitly provide generic arguments of any kind to a function.
Instead, Swift expects generic arguments to be given using metatype parameters: the client passes `Int.self` to a parameter of type `T.Type`, and the type checker infers that `T` must be `Int`.
A similar concept can apply to scope parameters; we just need some type whose sole purpose is to carry a scope specifier.

This could be done by just creating a trivial non-escapable type in the standard library:

```swift
struct Scope: ~Escapable {}
```

A function that wants an explicit scope parameter would just take a value of this type:

```swift
extension Span {
  init<scope s>(wrapping buffer: UnsafeBufferPointer<Element>,
                in scope: @scoped(s) Scope)
     -> @scoped(s) Span<Element>
}
```

This could be called something like this:

```swift
Span(wrapping: buffer, in: @scoped(&self) Scope())
```

Both the type and the initialization syntax could be sugared, of course.

### Non-escapable values and first-class functions

What does it mean to have a function that takes or returns a non-escaping value?

For standalone functions, this is often fairly unambiguous from the overall function signature.
Parameter values are restricted in scope.
They must not be stored into memory that does not carry the scope restriction, or else the language will not be able to keep them restricted in scope.
Only other parameters can carry the correct scope restriction.
Results must be trivial (e.g. an empty span) or else derived in some way from the parameters.
Non-trivial results cannot come from memory that does not carry the scope restriction, because to be in that memory in the first place, those values would have needed to escape there.

None of that reasoning works for first-class function values, such as closures (anonymous functions).
The caller of a function that takes a closure parameter knows, locally, what the scope arguments of the call are.
It can therefore soundly do things with non-escapable parameters and results in the closure that a standalone function could not do.

For example, consider a function that generates all of the prefix spans of an array, passing them to a callback:

```swift
func generatePrefixes(from array: Array<Int>,
                      into callback: (Span<Int>) -> Void)
```

A standalone function with the signature `(Span<Int>) -> Void` must only use the span locally.
Unless it specifically requires an *immortal* span, it must not care what scope the span is restricted to, because there is no other scope it could possibly know about.
Therefore it is reasonable to assume that it is generic over the scope of the span it receives.

But an arbitrary function value with this signature does not have this limitation.
The callback *might* just use the span locally, in which case it could reasonably be generic over the scope of the span.
If we wanted `generatePrefixes` to insist on this, we could force the callback to be generic over the scope, like so:

```swift
func generatePrefixes(from array: Array<Int>,
                      into callback: (<scope s> @scoped(s) Span<Int>) -> Void)
```

But there's no reason for `generatePrefixes` to do that.
The spans passed to the callback are derived from the borrowed `Array` parameter, which means they all have a restriction to that borrow scope.
That is, we really want to say:

```swift
func generatePrefixes(from array: Array<Int>,
                      into callback: (@scoped(array) Span<Int>) -> Void)
```

And the caller of `generatePrefixes` can actually use that fact.
After all, it knows something locally about what the scope of the borrow is.
There's no reason that the callback function passed in here shouldn't be able to save one or more of the spans, as long as they're still used within that borrow:

```swift
let array = [2,4,7,13]
var spans = [Span<Int>]()
generatePrefixes(from: array) { span in
  guard span.allSatisfy { $0.isMultiple(of: 2) } else { return }
  spans.append(span)
}
print(spans) // okay because we can extend the borrow of `array`
```

So there's an important difference here.
Type-based scope restrictions give us an excellent way to talk about this difference, as well to express further gradations like exactly which enclosing scope the value depends on.
But the compiler can't reasonably choose a default, the way it might for a standalone function, without an intelligent understanding of how all the values interact.

Similarly, a standalone function that returns a non-escapable value must somehow derive it from one of its parameters.
Furthermore, as we already discussed, it must be generic over the scopes of those parameters.
But an arbitrary function value could reasonably derive it from something else that the function has access to.
Again, this can be sound because of the caller's local understanding of all the scopes involved.
For example:

```swift
func operateOnSpans<scope s>(fromGenerator gen: () -> @scoped(s) Span<Int>)

let span = ...
operateOnSpans {
  guard !span.isEmpty else { return span }
  span = span.extracting(droppingLast: 1)
  return span
}
```

Here, `operateOnSpans` might well care about the fact that the spans all have the *same* scope restriction.
This would allow it to save and work with multiple span values, not just the last one returned by the callback.
But it could also depend on parameters, if there were any.
Another alternative would be that it returns a fresh span each time, which may be invalidated by the next call.
(For example, it might be building a value in a common buffer, then return the current state of that.)
That would be useful, but it's not immediately obvious to me how to express that, even with type-based scope restrictions.

#### Higher-rank scope polymorphism

Some of these expressive possibilities rely on having first-class function values that are generic over scopes.
In PL theory, this is known as higher-rank polymorphism.[^11]
Here it would be restricted to polymorphism over scopes.

[^11]: Not to be confused with higher-*kinded* polymorphism.

       Polymorphic types can be thought of as having a *universal quantifier*, which in Swift is written as a generic parameter list, like `<T: Hashable>`.

       Higher *rank* means that the quantifier is allowed in nested positions in another type.
       In the following code, `<U>` is in a higher-rank position, meaning that `foo` takes a function which is itself generic.

       ```swift
       func foo<T>(fn: <U> (U) -> T) -> [T]
       ```

       Higher *kind* means that the quantifier allows generic parameters to still themselves be generic.
       In the following code, the generic parameter `T` is constrained to be a generic type, which can then be applied to different generic arguments at different positions in the function signature:

       ```swift
       func bar<T<U>>(value: T<Int>) -> T<String>
       ```

       These are very different things; the only similarity between them is the name.

Higher-rank scope polymorphism is important for expressing APIs such as [`withTemporaryAllocation`][SE-0524]:

```swift
public func withTemporaryAllocation<T: ~Copyable, R: ~Copyable, E: Error>(
  of type: T.Type,
  capacity: Int,
  _ body: (inout OutputSpan<T>) throws(E) -> R) throws(E) -> R
```

The `OutputSpan` is allocated in a scope internal to the execution of `withTemporaryAllocation`.
In the type-based model, this scope cannot be a scope parameter of `withTemporaryAllocation` because that would make it external to the call, not defined within it.
The correct way to model this is by requiring `body` to be generic over the scope:

```swift
public func withTemporaryAllocation<T: ~Copyable, R: ~Copyable, E: Error>(
  of type: T.Type,
  capacity: Int,
  _ body: <scope s> (inout @scoped(s) OutputSpan<T>) throws(E) -> R) throws(E) -> R
```

This forces the parameter function to accept a span of *any* scope.
`withTemporaryAllocation` can then pass in a span scoped internally to itself.

This sort of callback with a temporary value is a major use pattern for non-escapable types.
Supporting higher-rank scope polymorphism is therefore a mandatory part of implementing the type-based model.[^12]

[^12]: I have seen it said that Rust made do without this feature for several years.
       This appears to be a misunderstanding; thank you to Aviva Ruben for this clarification.
       Rust's function traits did not allow traits to be polymorphic over scopes until [RFC 387](https://rust-lang.github.io/rfcs/0387-higher-ranked-trait-bounds.html) in 2014.
       However, Rust's legacy closure types supported scope polymorphism well before this.
       So it wasn't impossible to provide safe callback APIs, it just required using boxed function values.
       Not ideal, of course, but still completely expressible.

A lot of the difficulty in supporting higher-rank polymorphism is just dealing with the ensuing complexity in both type representation and substitution.
Implementing higher-rank scope polymorphism would require solving some of these problems and therefore may make it easier in the future to support higher-rank polymorphism over types as well.
There are some compelling use cases for such a feature; a recent example came up in [SE-0526][] for walking the active deadlines.
However, there also arguments against it.
Most importantly, it would become another code pattern where the compiler would often be unable to specialize generic code, which can be bad for performance.
However, this performance concern does not apply to scope polymorphism because the scopes are erased at runtime.

### Flow-sensitive refinement of scope restrictions

The value-dependency model reasons about a local variable of non-escapable type using the abstract values that the variable takes on over the course of the function.
Different abstract values can have different dependencies.
As a result, the model is capable of flow-sensitive reasoning: if the value of a variable has fewer dependencies at a particular point in the program, Swift is more permissive about how the variable can be used there.

For example, consider a variable that holds an array of spans.
At first, the abstract value in the variable is an empty array and has no dependencies.
Uses of the variable at this point will always satisfy dependency checking.
If the program later calls `append` to add a span to the array, the new abstract value of the variable will have exactly the scope dependencies of that span value.
Uses of the variable can now only satisfying dependency checks that are compatible with that.
If another span is subsequently added to the array, the new abstract value will have the union of the scope dependencies of the two span values, and uses will be further restricted.

This does not naturally translate into the type-based model because variables normally have a fixed type across their entire scope.
If the variable is not declared with a specific scope restriction, the restriction will be inferred by finding a scope that makes all of the `append` calls valid.
Generally, this will be the intersection of the scope restrictions of the individual spans.
This scope restriction becomes part of the type of the variable, and all loads of the variable will have that same type.
Effectively, this makes the model more conservative: restrictions introduced by later changes to the variable end up being "backdated" to apply to earlier uses of it.

It's unclear how often Swift programs today actually require this permissiveness in order to satisfy the dependency checker.
It may be vanishingly uncommon.
If so, the type-based model can achieve *de facto* parity with the value-dependency model without worrying about this problem.

On the other hand, if we decide that it's necessary to make code like this compile, it is possible to express it using type-based rules.
The scope restrictions in the variable's type just need to be *refined* over the course of its scope.
Effectively, different abstract values of the variable would observe different types.
Whenever the data flow of the function links two abstract values (such as one being produced by mutating another), the earlier value is required to be a subtype of the later value.
Of course, this would require a more expensive and complex analysis in order to infer scope restrictions: both the control flow graph and the data flow of the variable need to be computed in order to identify the abstract values it takes on.

## Engineering planning sketch for adopting a type-based model

If we accept that we should switch to the type-based model of scope restrictions, we will need a plan for how to pull that off.
Unfortunately, the models are subtly different, and neither can be considered a true superset of the other.
Nor do they interact especially well at a formal level; I do not think it would be a good idea to try to implement both.

### Future-proofing evolution

In the short term, we should avoid adding more features to Swift that would make it harder to maintain source compatibility in the long term if Swift switches to a purely type-based model.
Most importantly, this means not generalizing any generic parameter or associated type to allow non-`Escapable` types if we would want that type to be a bound type in the long term.
That includes the `Element` associated type of any collection-ish protocols, such as `Iterable`.
It also includes the generic type parameter of any collection-ish generic types, including `Span`, `Ref`, and the unsafe pointers.
(`Optional` and `Result` were already generalized in Swift 6.2, and we'll just have to deal with that somehow.)

Note that it is specifically not a problem to accept [SE-0516][]'s `Iterable` protocol, despite it having a non-escapable `BorrowingIterator` associated type.
This is because that associated type would not be a bound type under the type-based model.
Iterators pick up their (non-`Element`) scope restrictions from the borrows of `self` performed for calls to `makeIterableIterator`, not from the type of the collection.
The problem is only in generalizing `Iterable` to allow non-escapable element types.

### Overall implementation plan

The type-based model will not immediately supplant the value-dependency model.
Code will continue to be compiled that uses the value-dependency model.
In the short term, the type-based model will need to be opt-in using an experimental feature flag.
The compiler will therefore need to support implementations of both models.
Sharing as much code as reasonably possible will be vital in order to keep this manageable.

Since library code can be built using either model, both models will have to have some strategy for interpreting function signatures from modules compiled under the other model.
Interpreting value-dependency signatures in the type-based model should be relatively straightforward, although it may be somewhat more conservative.
Interpreting type-based signatures in the value-based model will likely be difficult in the general case, and it may be necessary to prevent certain APIs from being used.
This should not be a problem as long as it doesn't apply to any existing library APIs.

Hopefully, it will become possible at some point to implement the value-dependency model on top of the type-based model, possibly with a small number of tweaks.
The implementations should then be fully united.

Once the Language Steering Group has decided that the type-based model has achieved adequate expressive parity with the value-dependency model, they can initiate the deprecation period for the latter.
The requirements of this process were laid out in the announcement of the [lifetime dependencies supported experimental feature](https://forums.swift.org/t/experimental-support-for-lifetime-dependencies-in-swift-6-2-and-beyond/78638).
In particular, this requires the creation of tools (perhaps compiler-based) to help programmers migrate to the new type-based feature.
Eventually, the LSG may approve the removal of the old feature and any associated implementing code.

### Checking strategy

The type-based model is essentially adding logic to Swift's type checker.
However, that logic is substantially different from the existing logic in that it must be function-global.
Even putting aside the future direction of flow-sensitive type refinement, the inferred scope restriction of a variable must consider all operations on the variable.
This is not possible with the structure-by-structure walk of the normal type checker.
Changing the type checker to operate function-global is not workable; we cannot increase the pressure on a system that is already one of the most over-burdened parts of the compiler.

Fortunately, we believe checking can be broken down into phases:

1. The main type-checking walk resolves types *without* scope restrictions for all structures in the function body.

2. After the entire body is type-checked, the scope-resolution analysis augments all of those types with scope specifiers.

3. Finally, uses of scope-restricted values are checked to ensure that they are appropriately nested within local scopes, potentially shortening or lengthening scopes when necessary to make that work.

Phase 3 is basically the "second half" of the current dependency-checking pass.
If restrictions to local scopes can be turned into SIL dependencies, it is likely that the existing pass can be largely reused for this.
The SIL passes do not need to do anything to check values that are restricted to abstract scope parameters, since those scopes necessarily include the entire current function.

Phase 2 is the bulk of the new analysis required for the type-based model.
Essentially, it will collect a set of unresolved "scope variables" for all of the bound types in the function and then build a constraint system to infer concrete scope bindings for those variables.
This constraint system will ultimately be a set of subscope relationships derived from subtype relationships between values according to their use.

It is an open question whether these unresolved scope variables are introduced during Phase 1 or Phase 2.
If they're introduced in Phase 1, they will naturally be propagated by the substitution in the constraint solver, and the constraint solver can potentially just have the subtype relationships as a secondary output that gets collected for Phase 2.
However, the constraint solver will also have to deal with the existence of all these unresolved scope variables, including sometimes introducing them.
Introducing the variables in Phase 2 would avoid that, but only by creating a bunch of its own problems.
Every resolution, substitution, and type propagation performed by the constraint solver will need to be repeated in the Phase 2 analysis, this time including the scope specifiers.

Phase 2 will initially be performed in Sema, which means it will not have a CFG or data-flow analysis available.
This means it will not be able to do flow-sensitive refinement.
That's fine for now.
If we decide refinement is necessary, we'll need to move Phase 2 into a SIL pass, which would be a major rewrite.
However, we should at least have a solid set of test cases in place.

The Phase 2 checker will collect and solve a system of subscope relationships between all of the scope variables and concrete scopes in a function.
After Phase 2, all unresolved scope variables will be eliminated and resolved down to specific scope parameters, local access scopes, and intersections thereof.
Some subscope relationships, especially those only involving scope parameters and/or the immortal global scope, can be checked and resolved during Phase 2.
Some relationships involving local scopes will need to be checked by the Phase 3 checker in addition to the conflict checking that that pass normally does.
It should be the case that all the unchecked relationships entering Phase 3 will have the form "(some local access scope) is a superscope of (some other local access scope / some scope parameter / the immortal global scope)".

### Type representations

The type and signature representations used in the compiler's AST and type-checker will need to be extended to record scope parameters and scope specifiers.
Scope specifiers are likely to be the most invasive part of this, since they will be necessary on many types that normally do not have this kind of structure, like (otherwise) non-generic nominal types.
It is an open question whether scope parameters and arguments should be modeled as part of the generic signature and substitutions, implying that non-escapable types are essentially always generic, or as a separate thing.
Unifying them makes some sense and would fit more cleanly into the existing system of substitution and mapping in and out of context.
However, keeping them separate would make it easier to isolate the handling of scopes within the typechecker.

### SIL representations

SIL should also represent scope specifiers explicitly in its type system.[^13]
This will avoid the complexity of changing the representational expectations of types between the two systems.
It also preserves the information in case SIL passes decide to use it for performance optimization.

[^13]: There was an earlier misunderstanding on this point.

`begin_access` instructions will bind local scope variables in essentially the same way that the `open_existential` and `open_pack_element` instructions bind local archetypes.
Similarly, instructions with type operands (such as substitution maps) that depend on a local scope variable will have an implicit value use of the `begin_access` that binds that variable.
This will allow parts of the SIL pipeline to reason about the dependency without having to consider types explicitly.

Because all of the unchecked constraints entering Phase 3 checking involve a specific local access scope, they can probably just be stored on the `begin_access` instruction.
It's unclear whether there's any reason to persist these constraints beyond the checking passes.

[SE-0176]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0176-enforce-exclusive-access-to-memory.md
[SE-0414]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0414-region-based-isolation.md
[SE-0446]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0446-non-escapable.md
[SE-0516]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0516-borrowing-sequence.md
[SE-0519]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0519-ref-mutableref-types.md
[SE-0524]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0524-span-temporary-allocation.md
[SE-0526]: https://github.com/swiftlang/swift-evolution/blob/main/proposals/0526-deadline.md

[SSA]: https://en.wikipedia.org/wiki/Static_single-assignment_form
