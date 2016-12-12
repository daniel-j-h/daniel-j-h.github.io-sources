+++
Categories = []
Description = ""
Tags = ["Cxx"]
date = "2016-12-12T20:36:17+01:00"
title = "Strong Typedefs in C++ by Inheriting Constructors"

+++

## Motivation

Suppose we want to represent a `Tree` type as either a `Leaf` type or a recursive `Node` type which holds a left and right `Tree` itself.
One way to accomplish this is making use of a type-safe sumtype such as Boost.Variant.
In addition we want our `Tree` type to be a strong typedef instead of a weak alias for a variant.

This can can be done in C++11 and C++14 by inheriting from the variant and then inheriting its constructors, too.

```c++
using TreeBase = boost::variant<Leaf, boost::recursive_wrapper<Node>>;

struct Tree : TreeBase {
  using TreeBase::TreeBase;

  // ...
};
```

Our `Tree` is now a strong typedef (opaque typedef, newtype). Great!

Except that GCC trunk crashes with an internal compiler error.
And LLVM rejects this with compile-time errors, too.

As it turns out inheriting constructors has its quirks and defects for which we need C++17 to fix a total of _eight_ issues documented here:

> http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/p0136r1.html

Quoting the summary for one of those issues regarding inheriting constructors:

> Default arguments are a common mechanism for applying SFINAE to constructors. However, default arguments are not carried over when base class constructors are inherited; instead, an overload set of constructors with various numbers of arguments is created in the derived class. This seems problematic.

If we take a look at Boost.Variant (pseudocode below) we see how this is "problematic":

```c++
struct variant {
  template <typename T>
  variant(const T& t, typename enable_if<is_constructible_from<const T&>::value>::type* = 0) { }

  template <typename T>
  variant(T&& t, typename enable_if<is_constructible_from<T&&>>::type* = 0) { }
};
```

Inheriting `variant`'s constructors now creates multiple constructors in the derived class completely messing with the SFINAE approach, as described in the issue summary.

And this is where my journey ends.
I really hoped for an easy way to create strong typedefs via inheriting constructors.
But as long as there is no language support for it, it seems like a lost cause.


## Further Reading

- [Rewording inheriting constructors (core issue 1941 et al)](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/p0136r1.html)
- [Boost.Variant's Constructors](https://github.com/boostorg/variant/blob/f739850467d544d51721723d2b606764469f6777/include/boost/variant/variant.hpp#L1757-L1792)
- [Small Self-Contained Reproducible Example](https://gist.github.com/daniel-j-h/12c8b7f1c59b5a76c7e75dab38eb06fe#file-crash-cc)
- [GCC ICE Ticket](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=78767)
- [Boost.Variant Ticket](https://svn.boost.org/trac/boost/ticket/12680)

## Summary

Before C++17 you can not tell if inheriting constructors is possible or not other than by reading the library code.
Thanks to Agustín Bergé who brought this to my attention.

If you are curious have a look at Haskell's pattern matching or Rust's and Swift's enums.
