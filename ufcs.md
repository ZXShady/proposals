# PXXXXR0: Explicit Extension Method Syntax via Qualified Member Call

## Document #: PXXXXR0 Date: TODAY Project:Programming Language C++ Audience:EWG (Evolution Working Group) Reply to: Shady, mail.com

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

This paper proposes a new syntax for explicitly calling non-member functions using member function call syntax. The expression `obj.nested-name-specifier::function()` will perform a qualified lookup for a function in the specified namespace and prepend `obj` as its first argument.

This approach provides pure syntactic sugar for calling known free functions as extension methods. It is designed to be non-breaking, unambiguous, and simple to implement, as it does not interact with or change existing member lookup rules in any way, it is a simple rewrite.

#### Motivation and Scope

The primary goal of this proposal is to provide a fluent, member-like calling syntax for free functions that serve as "extension methods" for a type, without any of the downsides of implicit Unified Function Call Syntax (UFCS).

1. Explicit Extension Methods: Allows libraries to provide functionality for existing types, enabling users to call these functions with a fluent syntax by explicitly naming the providing namespace (e.g., vec.MyMath::normalize()).
2. Fluent Interfaces: Enables chaining for free functions (e.g., value.MyLib::transform().MyLib::filter().MyLib::process()).
3. Absolute Clarity: The syntax `x.f()` continues to find only members. The new syntax `x.N::f()` finds only free functions in namespace N. There is no possible ambiguity and it is clead that N::f cannot be a virtual function.

4. Non-Breaking: This change is purely additive. No existing code can be affected.

5. Preserved API. what the class provides is what is needed all other can be implemented as free functions on top of it, but however alot of classes implement  convenience members for simplicity like `std::string` 80% of the api of `std::string` can be free functions but they are members for simplicity, this creates unnecessary bloat, as for example the entire `std::string_view` API is the exaxt same as `std::string` yet it needed all those member functions to be api compatible if instead they were free functions it would be written once and used for any string-like type, the entire monadic api for std::optional,std::expected is the same but yet it is a member function purely for left to right syntax.



Member Function       | std::string Overloads | std::string_view Overloads | Primary Difference
----------------------|-----------------------|----------------------------|-------------------
operator[]            | 2 (const + non-const) | 1 (const only)             | string can modify.
at()                  | 2 (const + non-const) | 1 (const only)             | string can modify.
front()               | 2 (const + non-const) | 1 (const only)             | string can modify.
back()                | 2 (const + non-const) | 1 (const only)             | string can modify.
size() / length()     | 1                     | 1                          | Identical.
max_size()            | 1                     | 1                          | Identical.
empty()               | 1                     | 1                          | Identical.
(both const only).
begin() / end()       | 2 (const + non-const) | 1 (const only)             | string iterators can modify.
cbegin() / cend()     | 1                     | 1                          | Identical (both const only).
rbegin() / rend()     | 2 (const + non-const) | 1 (const only)             | string iterators can modify.
crbegin() / crend()   | 1                     | 1                          | Identical (both const only).
copy()                | 1                     | 1                          | Identical (both const only).
substr()              | 1                     | 1                          | Different return types.
compare()             | 6                    | 6                         | Similar overload sets.
find()                | 4                     | 4                          | Similar overload sets.
rfind()               | 4                     | 4                          | Similar overload sets.
find_first_of()       | 4                     | 4                          | Similar overload sets.
find_last_of()        | 4                     | 4                          | Similar overload sets.
find_first_not_of()   | 4                     | 4                          | Similar overload sets.
find_last_not_of()    | 4                     | 4                          | Similar overload sets.

There is also 0 reason to limit all of these functions to only apply on std::string & std::string_view and not other types that are string-like like a `vector<char>`,`span<char>`. 

the standard could instead provide free overloads for these

```cpp
template<class C>
C sub(C& c,std::size_t begin,std::size_t end)
{
   auto it = c.begin() + begin;
   return C(it,it+end);
}

// call
std::string s;
std::span<int> span;
std::vector<int> v;
s.std::sub(3,5);
sv.std::sub(1,9);
v.std::sub(39,4);
span.std::sub(3,6);
```

Just wrote a single function got alot of utility of it this is the main point of the paper.

6. Benefits old code, there is alot of code from C that could benefit from left to right syntax like FILE* api.

This proposal explicitly does not solve the problem of generic code that calls a member function but wants to call a non member as well. Its value is in its simplicity and explicit intent and most importantly syntax sugar!

Before/After Examples

Before this proposal:

```cpp
namespace Math {
    struct Vector { double x, y; };
    Vector normalize(const Vector& v); // free function
}

namespace Containers {
     class Array { ... };
     void copy_to(const Array& from,Array& to);
};
Math::Vector v{1.0, 2.0};
auto n1 = normalize(v);        // OK: traditional call
auto n2 = Math::normalize(v);  // OK: qualified call
// auto n3 = v.normalize();    // Error: 'normalize' is not a member
// v.Math::normalize();        // Error: invalid syntax.

Containers::Array a,b;
copy_to(a,b); // copy from "a" to "b" or from "b" to "a"? 
// For built-in types and C functions:
int* p = nullptr;
// p.free();                   // Error: int* has no members 
free(p);                       // OK: traditional call
::free(p);                     // OK: explicitly qualified
```

After this proposal:

```cpp
namespace Math {
    struct Vector { double x, y; };
    Vector normalize(const Vector& v); // free function
}

namespace Containers {
     class Array { ... };
     void copy_to(const Array& from,Array& to);
};

Math::Vector v{1.0, 2.0};
auto n1 = normalize(v);         // OK: traditional call
auto n2 = Math::normalize(v);   // OK: qualified call
auto n3 = v.Math::normalize();  // OK: new explicit extension call

Containers::Array a,b:
a.Containers::copy_to(b); // explicit function used and clear where the copy is.
// For built-in types and C functions:
int* p = nullptr;
// p.free(); // ERROR: int* has no members
p.::free();                     // OK: calls ::free(p)

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



Semantics

For a call expression of the form x.ns::f(args):

1. Qualified Lookup Only: The compiler performs a qualified lookup only for a function name `f` in the namespace `ns`. It does not perform any lookup for a member of `x` named `f`.
2. Transformation: If a function ns::f is found, the expression is transformed into a function call with the object x prepended to the argument list: ns::f(x, args). Overload resolution is performed on this new call expression.
3. Error Handling: If no function ns::f is found by qualified lookup, then it is an error.

`using` declarations are ignored.

```cpp
using ::std:: size;
obj.size(); // always call Obj::size(), will never try to call std::size.
obj.std::size(); // always call std::size();
```

This is an important differentiators, as it won't allow easier generic programming.

Rational

· Exclusive Lookup: x.ns::f() only finds free functions. x.f() only finds members. The two lookup mechanisms are completely separate and unambiguous.
· Non-Breaking: The meaning of all existing code is preserved. The change is purely additive.

· Pure Syntax Sugar: The transformation is purely syntactic. x.ns::f(args) is entirely equivalent to ns::f(x, args) in every regard.


Considered Alternatives

1. Implicit UFCS: Previous proposals tried to make x.f() find free functions. This was rejected due to complexity and breaking changes. This proposal is the antithesis of that: it requires explicit qualification.

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

Limitations and Impact

Impact on the Standard: This is a pure extension to the language.

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

· [N4174](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2014/n4174.pdf - Call syntax: x.f(y) vs. f(x,y)
· [P0251R0](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2016/p0251r0.pdf) - Unified Call Syntax