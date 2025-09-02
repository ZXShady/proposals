# PXXXXR0: Alternative free function call syntax.

Date: TODAY 

Audience:EWG

Reply to: Shady, [shadyaa@outlook.com](https://www.youtube.com/watch?v=dQw4w9WgXcQ)

### Table of Contents

1. [Abstract](#Abstract)
2. Motivation and Scope
3. Before/After Examples
4. Design & Proposal
5. Technical Specification
6. Considered Alternatives
7. Limitations and Impact
8. Acknowledgements
9. References

#### Abstract

This paper proposes a new syntax for explicitly calling non-member functions using member function call syntax. The expression `obj.namespace::function()` will perform a qualified lookup for a function in the specified namespace and prepend `obj` as its first argument.

This approach provides pure syntactic sugar for calling free functions as extension methods. It is designed to be non-breaking, unambiguous, and simple to implement, as it does not interact with or change existing member lookup rules in any way, it is a simple rewrite.

#### Motivation and Scope

The primary goal of this proposal is to provide a fluent, member-like calling syntax for free functions that serve as "extension methods lite" for a type, without any of the downsides of implicit Unified Function Call Syntax (UFCS).

It is clear that some sort of UFCS needs to be in C++ from numerous papers like multiple UFCS papers and Pipeline rewrite operator ans the `std::ranges` `operator|` chains, it is clear that there is demand for some sort of UFCS in C++.

This paper brings

1. Explicit Extension Methods: Allows libraries to provide functionality for existing types, enabling users to call these functions with a fluent syntax by explicitly naming the providing namespace (e.g., `vec.MyMath::normalize()`).
2. Fluent Interfaces: Enables chaining for free functions (e.g., `value.MyLib::transform().MyLib::filter().MyLib::process())`.

3. Absolute Clarity: The syntax `x.f()` continues to find only members. The new syntax `x.N::f()` finds only free functions in namespace N. There is no possible ambiguity and it is clear that N::f cannot be a virtual function.

4. Non-Breaking: This change is purely additive. No existing code can be affected.

5. Promotes Non-Member, Non-Friend Interfaces & Generic Reuse. A significant class of member functions of many types are convenience functions that do not require access to private class state. This proposal provides a clear and fluent syntax for implementing such functions as non-members, reducing class API bloat and dramatically increasing reusability across types.

The classic example is the similarity between the purely const interfaces of `std::string` and `std::string_view`. `std::string_view` had to reimplement this entire interface as members, leading to significant duplication for operations that just act on *data* but their member-only implementation locks them to specific standard types.

This proposal would allow a single set of generic, non-member functions to provide this functionality for any compatible type, enabling a fluent syntax without requiring every type to implement its own members.

Shared const Member Functions: `std::string` and `std::string_view`

Member Function       | Shared Overloads
----------------------|-----------------
length()              | 1
max_size()            | 1
empty()               | 1
cbegin() / cend()     | 1
crbegin() / crend()   | 1
copy()                | 1
substr()              | 1
starts_with()         | 4
ends_with()           | 4
compare()             | 9
find()                | 4
rfind()               | 4
find_first_of()       | 4
find_last_of()        | 4
find_first_not_of()   | 4
find_last_not_of()    | 4
Total                 | 46 * 2 == 92

The power of this proposal is enabling a single, generic function to replace many specific members. This extends beyond strings to any compatible range:

```cpp
// A single generic 'sub' algorithm, written once
namespace std {
  template <class Rng>
  auto sub(const Rng& r, std::size_t pos, std::size_t count) {
      auto it = r.begin() + pos;
      return Rng(it, it + std::min(count, r.size() - pos));
  }
  template <class Rng,class Rng2>
  auto starts_with(const Rng& obj,const Rng2& other ) {
       // implementation unimportant
       return obj.std::sub(0,other.size()) == other; 
  }
}

// Used fluently across wildly different types
std::string s = "hello world";
std::string_view sv = s;
std::vector<int> v = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
std::span<const int> sp = v;

// All these work with the same fluent syntax
auto part1 = s.std::sub(6, 5);    // "world"
auto part2 = sv.std::sub(6, 5);   // "world"
auto part3 = v.std::sub(6, 4);    // {7, 8, 9, 10}
auto part4 = sp.std::sub(2, 3);   // {3, 4, 5}

s.std::starts_with(sv); // true!
sp.std::starts_with(v); // true!

// free chaining
sv.std::sub(6,5).std::starts_with("world");

// without chaining
starts_with(sub(sv,6,5))
```

This approach eliminates API duplication and allows new user-defined types to automatically gain this functionality simply by providing begin(), end(), and appropriate constructors with free functions while not having second class citizen syntax.


6. Benefits old code, there is alot of code from C that could benefit from left to right syntax like FILE* api.

```cpp
FILE* f = "File".::fopen("rb");
long offset;
int origin;
f.::fseek(offset,origin);
f.::fclose();    
```

This proposal explicitly does not solve the problem of generic code that calls a member function but wants to call a non member as well. Its value is in its simplicity and explicit intent and elegance.

Before/After Examples
+-----------------------------------------------------------------------------+-----------------------------------------------------------------------------+
| Before                                                                      | After                                                                       |
+-----------------------------------------------------------------------------+-----------------------------------------------------------------------------+
| namespace Math {                                                            | namespace Math {                                                            |
|     struct Vector { double x, y; };                                         |     struct Vector { double x, y; };                                         |
|     Vector normalize(const Vector& v);                                      |     Vector normalize(const Vector& v);                                      |
| }                                                                           | }                                                                           |
|                                                                             |                                                                             |
| Math::Vector v{1.0, 2.0};                                                   | Math::Vector v{1.0, 2.0};                                                   |
| auto n1 = normalize(v);                                                     | auto n1 = normalize(v);                                                     |
| auto n2 = Math::normalize(v);                                               | auto n2 = Math::normalize(v);                                               |
| // auto n3 = v.normalize();    // Error                                     | auto n3 = v.Math::normalize();  // OK                                       |
| // v.Math::normalize();        // Error                                     | // Transformed to: Math::normalize(v)                                       |
+-----------------------------------------------------------------------------+-----------------------------------------------------------------------------+
| namespace Containers {                                                      | namespace Containers {                                                      |
|      class Array { ... };                                                   |      class Array { ... };                                                   |
|      void copy_to(const Array& from,Array& to);                             |      void copy_to(const Array& from,Array& to);                             |
| };                                                                          | };                                                                          |
|                                                                             |                                                                             |
| Containers::Array a,b;                                                      | Containers::Array a,b;                                                      |
| copy_to(a,b); // Direction unclear                                          | a.Containers::copy_to(b); // Explicit and clear                             |
|                                                                             | // Transformed to: Containers::copy_to(a, b)                                |
+-----------------------------------------------------------------------------+-----------------------------------------------------------------------------+
| int* p = nullptr;                                                           | int* p = nullptr;                                                           |
| // p.free();                   // Error                                     | p.::free();                    // OK                                        |
| free(p);                       // OK                                        | free(p);                        // Still OK                                 |
| ::free(p);                     // OK                                        | ::free(p);                      // Still OK                                 |
|                                                                             | // Transformed to: ::free(p)                                                |
+-----------------------------------------------------------------------------+-----------------------------------------------------------------------------+
| std::string_view sv = "hello world";                                        | std::string_view sv = "hello world";                                        |
| auto result = starts_with(substr(sv, 6, 5), "world"); // Cumbersome        | auto result = sv.std::substr(6,5).std::starts_with("world"); // Fluent      |
|                                                                             | // Transformed to: std::starts_with(std::substr(sv,6,5), "world")           |
+-----------------------------------------------------------------------------+-----------------------------------------------------------------------------+

Another example is having monadic functions for every optional like type.

```cpp
namespace std {
   // Written once
   // Allows pointers, unique_ptr,shared_ptr,optional,expected to be used in a nice syntax no need to write it for every class.
   auto value_or(auto& opt,auto default_)
   {
      return opt ? *opt : default_;
   }
}

std::optional<int> opt;
opt.std::value_or(0);
int* pointer;
pointer.std::value_or(0); 
```

The syntax provides a clear visual distinction and works for any namespace.

Design & Proposal

Syntax

The proposal utilizes the existing grammar for postfix-expression but defines a new semantic path for previously invalid syntax to enable calling free functions as member functions.

For a call expression of the form `x.ns::f(args)`: Note the namespace cannot be omitted even if the current scope is the same namespace.

1. Qualified Lookup Only: The compiler performs a qualified lookup only for a function name `f` in the namespace `ns`. It does not perform any lookup for a member of `x` named `f`.
2. Transformation: If a function ns::f is found, the expression is transformed into a function call with the object x prepended to the argument list: ns::f(x, args). Overload resolution is performed on this new call expression.
3. Error Handling: If no function ns::f is found by qualified lookup, then it is an error.

`using` declarations are ignored.

```cpp
using ::std:: size;
obj.size(); // always call Obj::size(), will never try to call std::size.
obj.std::size(); // always call std::size();
```
This is an important differentiator as it won't allow easier generic programming. it could later be added.

Rational for Design.

· Exclusive Lookup: x.ns::f() only finds free functions. x.f() only finds members. The two lookup mechanisms are completely separate and unambiguous.

· Non-Breaking: The meaning of all existing code is preserved. The change is purely additive.

· Pure Syntax Sugar: The transformation is purely syntactic. x.ns::f(args) is entirely equivalent to ns::f(x, args) in every regard.

Considered Alternatives

1. Implicit UFCS: Previous proposals tried to make x.f() find free functions. This was rejected due to complexity and breaking changes. This proposal is the antithesis of that: it requires explicit qualification.

2. Extension Methods: Previous proposals tried to make `x.f()` find free functions if the 1st argument used special constructs or names.

a simple example that can break is for example adding `starts_with` to `std::string_view`.

```cpp
// imaginary syntax for extensions
size_t starts_with(const this std::string_view& a,std::string_view b)
{
    return a.substr(0,b.size()) == b;
}

std::string_view s;
s.starts_with("Hello");
```


this is all fine and good, until `std::string_view` itself gets a member named `starts_with` which takes a `const char*` that instead will be chosen.

This is fine since both of the functions do the same thing in this case but what if they don't you will get silent breakage and class writers can't gurantee what their api provides, and this is why this paper chosed an explicit syntax for that, it avoids all the issues.

An API can provide a gurantee that all accesses of its const member functions are thread-safe, extension methods would break that as both `x.ext()` and `x.mem()` look the same, while this proposal provides explicit intent `x.Utils::ext()` is clear that it is not part of the official api.

3. Overloading operators like `std::ranges`
   *It works* but it is a workaround, and it results in bad debug codegen and worsens compile times that are already bad.

Example from cppreference

https://en.cppreference.com/w/cpp/ranges.html

```cpp
#include <iostream>
#include <ranges>
    constexpr auto even = [](int i) { return 0 == i % 2; };
    constexpr auto square = [](int i) { return i * i; };
void functional()
{
    auto const ints = {0, 1, 2, 3, 4, 5};

 #if PIPE
     for (int i : ints | std::views::filter(even) | std::views::transform(square))
#else
    for (int i : std::views::transform(std::views::filter(ints, even), square))
#endif
        std::cout << i << ' ';
}
 
int main(){}
```

[Godbolt](https://godbolt.org/z/ohsrbTxxK) shows doing DFLIP=1 increases the amount of assembly by a 100+ just for 
using the nicer syntax, and worse compile times due to the magic needed to implement it.

#### Limitations and Impact

Impact on the Standard: This is a pure extension to the language that can later aid in library design.

Limitations: This mechanism does not aid in writing generic templates, as the namespace must be explicitly specified. Its primary benefit is syntactic convenience and fluency for non-generic code.

Example of Limitation:

```cpp
// This works today and is not fully generic
template <typename T>
void func(T t) {
    t.foo((; // Only finds t.foo();
}

// This proposal does NOT enable this:
template <typename T>
void func(T t) {
    t.foo(); // Looks a member foo then an adl foo
}
```

References

· [N4174](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2014/n4174.pdf) - Call syntax: x.f(y) vs. f(x,y)
· [P0251R0](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2016/p0251r0.pdf) - Unified Call Syntax