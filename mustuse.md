# @mustuse as function return value annotation


There are two scenarios we need to handle:

- existing legacy library code, which the programmer has no right to modify
- new code, which the programmer has full control from the root of the inheritance tree

## @mustuse is neither covariant nor contravariant, it is invariant!

@mustuse as a function attribute only enforces there must be a receiver of the function call result, and do not throw away
the returned result. It has nothing to do with the (sub-)types of the returned value.

Let's consider the following example:
```
abstract class AbsPaths {
  // no annotation
  abstract AbsPaths remove(int i);  // return an AbsPaths object with i-th element removed
}

class ImperativePaths : AbsPaths {  // will modify `this` object itself
  // no @mustuse annotation, with the implicit assumption that the caller will continue to use `this` object
  overide AbsPaths remove(int i) {
    ... // remove the i-th element of `this` object
    return this;
  }
}

class FunctionalPaths : AbsPaths {  // will NOT modify `this` object, but return new (separate) object on modification
  @mustuse  // should have this annotation
  overide AbsPaths remove(int i) {
    AbsPaths other = new ImmutablePaths();
    ...  // `other` is the object with the i-th element of `this` object being removed; `this` object did not change
    return other;
  }
}

void main() {
  AbsPaths paths = new ImperativePaths();  // or FunctionalPaths() interchangeably
  AbsPaths shortPaths = paths.remove(i);   // and this line should always work, as long as the return value is not discarded
}
```
Here both `ImperativePaths` and `FunctionalPaths` inherit from `AbsPaths`, if one branch has @mustuse and the other one
does not, and the programmer is not forced to explicitly take the return value and use it, these two derived classes cannot be used
interchangeably.

### @mustuse propagation: transitive closure
In the following class inheritance tree:
```
----------Base-------
|         |         |
Derived1  Derived2  Derived3 <-user only manually maked Derived3.remove() as @mustuse here
|         |         |
GrandDr1  GrandDr2  GrandDr3
|
...
```
If the programmer only manually marked Derived3.remove() as @mustuse, then everything else (Base, Derived1, Derived2,
Derived3, GrandDr1, GrandDr2 GrandDr3, ...)'s .remove() method will all become @mustuse (as seen by the compiler internally).

With transitive closure @mustuse rule, 

Pros:

1. the method interface is consistent, e.g. at any call-site of `AbsPaths.remove(i)`, its return value must be received.
the programmer only need to remember one interface, instead of looking through docs/code for all the branches in the 
inheritance tree, and check for a particular class, whether its `remove(i)` method is @mustuse or not.
2. the programmer can easily switch between different derived implementation classes of AbsPaths to maximize efficiency /
performance, without worrying about potential breakage.

Cons:

1. a few more key-strokes on every call-site of @mustuse marked method.

## Prior work

[Paul Backus proposal:](https://forum.dlang.org/post/cqlwlnpcbtbkzqnhicwc@forum.dlang.org) 

>In other words, you cannot introduce @mustuse in a derived class; it must be present in the base class.
>
>This is somewhat problematic for D, because we have a universal base class, Object, and neither it nor any of its methods are @mustuse.

His reasoning is logically correct by itself, with the constraint that legacy code is un-modifiable.
However, his proposal is not very useful because of this constraint:

1. it won't help the existing library code where there is no @mustuse presence today. But, these library code are where the programmer 
want the compiler help most,  as demonstrated by the OP user who brought up this issue on the forum. (This is also why Paul talked about
Object, although it's not a very good example; instead we can think about std.lib.AbsPath above).
2. if the new rule have to fully honor the legacy (with deficiency) code, how we can improve for future D?
Also honoring legacy code, does not mean we should not even check for potential problems.
Even typically we cannot modify the the legacy code, at least we want the compiler help to check where are the potential problems
are; the compiler can give warning messages, and if they are manually verified, these findings should be formally 
logged as bugs, and be fixed in the next release.

And if we follow this line of reasoning, it also beg the question: whether one can remove @mustuse in a derived class. 
E.g. let ImperativePaths (mutable implementation) inherit
from FunctionalPaths (immutable implementation), and the caller can just use the `this` object as the result of the computation. Again,
this logic is correct by itself, but it make the whole code base brittle: what if the library author decided later one day that s/he want to
modify the class ImperativePaths again to implement another immutable implementation?


## Introduce two variants: @mustuse and @mustuse_remedy_legacy

1. @mustuse_remedy_legacy: this annotation is to allow programmer flag existing legacy library code that s/he has no 
right to change, but want to set a flag and let the compiler to help to find violations; the compiler only output warnings 
instead of errors. This annotation will be implicitly propagated by the compiler throughout the whole inheritance tree.

2. @mustuse: proper, the default. For programmer to use in new code, or library code s/he can change (from the root of the
inheritance tree). Violations of this annotation will cause compiler error. This annotation must be explicit (just as the keyword
`override`).

and:

a) in both cases, this function property will be transitive closure, i.e. be propagated upwards and downwards, in all directions.

b) in both cases, removing the annotation is not allowed in the derived class if any supper class method carry such annotation.

Actually this rule is very simple: there is only *one* consistent interface for any method, @mustuse is part of that method interface;
for legacy code, @mustuse_remedy_legacy will cause compiler to generate warning message, and for new code @mustuse will cause 
the compiler to generate error message. That's all.


### @mustuse_remedy_legacy for legacy code base and pre-built binaries
Let's revisit the motivating example:
```
paths.remove(i); // compiles fine but does nothing
paths = paths.remove(i); // works - what I erroneously thought the previous line was doing
```

Suppose the `paths`' type is `AbsPaths`, and the programmer have no right to modify it (e.g. in std.lib, or even pre-built binaries), 
the compiler can only issue warning (instead of error) messages. But the programmer must be made aware of where
these potential problems are located.

And, once the programmer discovered one such misuse problem, s/he can try to find and fix all such potential problems by defining 
a helper class RemedyPaths as follows:
```
// class Paths is located in the source file that the programmer has no right to modify
class RemedyPaths : std.lib.AbsPaths {  // helper class to trace all the occurrences of the @mustuse violation
  @mustuse_remedy_legacy
  override Paths remove(int i) {return null;}
}
```
and the compiler will find out all the occurrences of the same issues in the code visited by the compilation, and issue 
warnings (not errors), so the programmer have a chance to visit all the code locations where `remove(i)`'s function
return value are discarded.

### Informative detailed compiler warning messages

Since this kind of warning message is transitive closure by design, so for the *implicit* markings made by the 
compiler: as a debugging aid the warning message should indicate the **originating source of the annotation** 
to make it clear to the programmers, e.g.:
```
warning: foo.d:123, AbsPaths.remove(int i)'s return value is discarded, violates the originating 
annotation from RemedyPaths.d:456 @mustuse_remedy_legacy.
```

### Summary
With universal (i.e. transitive closure) @mustuse, 

1. when the library author has decided to return a value from a function, typically it's represent the computation result or status report,
which the function caller should either use or check instead of discard. That is good engineering practice. It forces the programmer
to pay attention to the returned value, instead of assuming the semantics of the function e.g. based purely on the function name.
```
  ResultType result = someFunction();
  ... // use or check `result`
```
it will save lots of debugging time, at the expense of just a few more key-strokes.

2. the library author can to implement either ImperativePaths or FunctionalPaths, and the library users can choose 
them interchangably for the maximal efficiency / performance without worrying about code breakage.


## History: command query separation principle

Not discarding function return value has its root from the command query separation principle. 
As an informal exercise: lets' derive command query separation principle from DbC.

The contract in DbC mostly exhibits as assertions in the code.

The programmer can insert assertions at any place of the code, without changing the code's original semantics (i.e the behavior when the assertions are turned-off, e.g. in release mode). In an OOP language, most of the time the assertions are checking some properties of an object, hence any method that can be called in an assertion must be a query (i.e a query can be called on an object for any number times without changing that object's internal state). So now we have query method.

But we do need to change an object's state in imperative programming, then those methods are classified as commands. After changing an object state, the command must NOT return any value. Why? because otherwise, the programmer may accidentally want to call that command and check the returned value in some assertions ... then you know what happens in the end: the program behaves differently when assertions are turn on in debug mode and off in release mode.

Therefore, we have this:

> every method should either be a command that performs an action, or a query that returns data to the caller, but not both.


## References:
- [Liskov substitution principle](https://en.wikipedia.org/wiki/Liskov_substitution_principle) 
- [Command query separation principle](https://en.wikipedia.org/wiki/Command%E2%80%93query_separation#:~:text=Command-query%20separation%20(CQS),the%20caller%2C%20but%20not%20) 
- [@mustuse DIP1038.md](https://github.com/dlang/DIPs/blob/master/DIPs/accepted/DIP1038.md#mustuse-as-a-function-attribute) 