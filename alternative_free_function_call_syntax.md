# PXXXXR0: Alternative free function call syntax.

Date: TODAY 

Audience:EWG

Reply to: Shady

### Table of Contents

1. [Abstract](#abstract)
2. [Motivation](#motivation)
3. [Proposal](#proposal)
4. [Examples](#examples)
5. [Technical Specification](#technical-specification)
6. [Considered Alternatives](#considered-alternatives)
7. [Possible Issues](#possible-issues)
8. [Limitations and Impact](#limitations-and-impact)
9. [Implementation](#implementation)
10. [References](#references)

#### Abstract

This paper proposes a new syntax for explicitly calling non-member functions using member function call syntax. The expression `obj.namespace::function()` will perform a qualified lookup for a function in the specified namespace and prepend `obj` as its first argument as if it was a rewrite expression.

This approach provides pure syntactic sugar for calling free functions as member functions acting like extension methods. It is designed to be non-breaking, unambiguous, and should be simple to implement, as it does not interact with or change existing member lookup rules in a major way unlike ADL, it is a simple rewrite.

#### Motivation

The need for some form of Unified Function Call Syntax (UFCS) in C++ has been discussed for years. Papers on UFCS and Extension Methods, pipeline syntax, and the existence of `std::ranges::views` using `operator|` demonstrate strong demand for more fluent function call syntax.

This paper provides a way to call free functions in a way that visually resembles member function calls, while avoiding the pitfalls of implicit lookup, major breaking changes or virtual call ambiguity.

1. Explicit Extension Methods: Allows libraries to provide functionality for existing types, enabling users to call these functions with a fluent syntax by explicitly naming the providing namespace (e.g., `matrix.Math::translate(position)`).

2. Better tooling for IDEs
    Today, IDEs struggle to offer useful auto completions for free functions like `std::size(x)` because they lack context on the object it will operate on :

    ```cpp
    X x;
    std::?? // IDE can’t suggest std::size(x) has to suggest everything
    ```

    After this proposal

    ```cpp
    X x;
    x.std:: // IDE can now suggest only applicable functions
    ```
    The later allows the IDE to filter out candidates much better and provide helpful auto complete, as you made your intent clear by asking for what functions can operate on `x`.

3. Fluent Interfaces
    Enables clean chaining of function calls:
    `range.MyLib::transform().MyLib::filter(odd).MyLib::process();`

4. Clarity: 
    The syntax `x.f()` continues to find only members. The new syntax `x.N::f()` finds only free functions in namespace `N`. There is no ambiguity for the reader and it is clear that `N::f` cannot be a virtual function.

5. No Major Breaking: 
    This is a purely additive feature. No existing code is impacted. No current behavior changes.

6. Promotes Non-Member, Non-Friend Interfaces & Generic Reuse.
     A significant class of member functions of many types are convenience functions that do not require access to private class state. This proposal provides a clear syntax for implementing such functions as non-members, reducing class API bloat and dramatically increasing reusability across types.

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
    Total                 | 46 * 2 == 92 OVERLOADS!

    Thats 92 overloads! For what is essentialy algorithms already in the C++ standard library like `std::find`,`std::cbegin` and others. This would get worse as the Standard Library ends up with more types with similiar interfaces (e.g `std::span` and `std::zstring_view`) more duplication will occur.

    The power of this proposal is enabling a single, generic function to replace many specific members. This extends beyond strings to any compatible range, while not having second class citizen calling syntax or the inability to chain.

    This approach eliminates API duplication and allows new user-defined types to automatically gain this functionality simply by providing `begin()`, `end()`, and appropriate constructors with free functions while not sacrificing readability

7. Benefits old code without any rewrites like C apis unlike Extension Methods.

> [!IMPORTANT]
> This proposal explicitly does not solve the problem of generic code that calls a member function but wants to call a non member as well. Its value is in its simplicity and explicit intent and elegance.


#### Examples

<table>
<tr>
<th>
Before
</th>
<th align="left">
After
</th>
</tr>

<tr>
<td  valign="top">

<pre lang="cpp">
namespace Math {
    struct Vector { double x, y; };
    // free function
    Vector normalize(const Vector& v); 
}
namespace Containers {
     class Array { ... };
     void copy_to(const Array& from,Array& to);
}

Math::Vector v{1.0, 2.0};
auto n1 = normalize(v);        // OK: call via ADL.
auto n2 = Math::normalize(v);  // OK: qualified call

Containers::Array a,b;
// ambigious, is it copy from "a" to "b" or from "b" to "a"? 
copy_to(a,b);                  // OK: ADL call

int* p = nullptr;
free(p);                       // OK: call
::free(p);                     // OK: qualified call

</pre>
</td>
<td  valign="top">
<pre lang="cpp">
namespace Math {
    struct Vector { double x, y; };
    // free function
    Vector normalize(const Vector& v);
}
namespace Containers {
     class Array { ... };
     void copy_to(const Array& from,Array& to);
}

Math::Vector v{1.0, 2.0};
auto n1 = normalize(v);         // OK: traditional call
auto n2 = Math::normalize(v);   // OK: qualified call
auto n3 = v.Math::normalize();  // OK: new explicit extension call

Containers::Array a,b:
a.Containers::copy_to(b); // explicit function used and clear where the copy is happening.

int* p = nullptr;
// p.free(); // ERROR: int* has no members
p.::free();                     // OK: calls ::free(p)

</pre>
</td>

</table>

The syntax provides a clear visual distinction and works for any namespace effectivly making every free function usable as an extension method by just specfiying the namespace.

Example 2: Code reuse without sacrificing chaining

<table>
<tr>
<th>
Before
</th>
<th align="left">
After
</th>
</tr>

<tr>
<td  valign="top">

<pre lang="cpp">
namespace std {
  // A single generic 'sub' algorithm, written once
  template &ltclass Rng>
  auto sub(const Rng& r, std::size_t pos, std::size_t count) {
      auto it = r.begin() + pos;
      return Rng(it, it + std::min(count, r.size() - pos));
  }
  template &ltclass Rng,class Rng2>
  auto starts_with(const Rng& obj,const Rng2& other ) {
       // MISTAKE! no qualification for 'sub'. an overload can be injected here
       return sub(obj,0,other.size()) == other; 
  }
}

// Used across wildly different types
std::string s = "hello world";
std::string_view sv = s;
std::vector<int> v = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
std::span<const int> sp = v;

auto part1 = std::sub(s,6, 5);    // "world"
auto part2 = std::sub(sv,6, 5);   // "world"
auto part3 = std::sub(v,6, 4);    // {7, 8, 9, 10}
auto part4 = std::sub(sp,2, 3);   // {3, 4, 5}

// does not read like English.
std::starts_with(s,sv); // true!
std::starts_with(sp,v); // true!

// awkard chaining inside out reading
std::starts_with(std::sub(sv,6,5),"world");

</pre>
</td>
<td  valign="top">
<pre lang="cpp">
namespace std {
  // A single generic 'sub' algorithm, written once
  template &ltclass Rng>
  auto sub(const Rng& r, std::size_t pos, std::size_t count) {
      auto it = r.begin() + pos;
      return Rng(it, it + std::min(count, r.size() - pos));
  }
  template &ltclass Rng,class Rng2>
  auto starts_with(const Rng& obj,const Rng2& other ) {
       return obj.std::sub(0,other.size()) == other; 
  }
}


std::string s = "hello world";
std::string_view sv = s;
std::vector<int> v = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
std::span<const int> sp = v;

// All these work with the same fluent syntax
auto part1 = s.std::sub(6, 5);    // "world"
auto part2 = sv.std::sub(6, 5);   // "world"
auto part3 = v.std::sub(6, 4);    // {7, 8, 9, 10}
auto part4 = sp.std::sub(2, 3);   // {3, 4, 5}

// Reads like English!
s.std::starts_with(sv); // true!
sp.std::starts_with(v); // true!

// free chaining left-to-right reading.
sv.std::sub(6,5).std::starts_with("world");
</pre>
</td>

</table>

#### Technical Specification

The author has little wording experience.

#### Proposal

The proposal utilizes the existing qualified name grammar for `operator.` but defines a new path for previously invalid syntax to enable calling free functions (or any functor) as member functions.

For a call expression of the form `x.ns::f(args)`: 

> [!NOTE]
> The namespace cannot be omitted even if the current scope is the same namespace.

A rewrite will happen as if you had written `ns::f(x,args)` all usual rules apply.

However `using` declarations are ignored.

```cpp
using ::std::size;
obj.size(); // always call Obj::size(), will never try to call std::size.
obj.std::size(); // always call std::size();
```
This is an important difference in this paper vs others as it won't allow easier generic programming. it could later be added with the `using` declaration as an opt in to full UFCS with member chosen first then global function second semantics.

#### Rational for design.

* Qualified Lookup, no ADL: `x.ns::f()` only finds free functions. `x.f()` only finds members. The two lookup mechanisms are completely separate and unambiguous to the reader no ADL can occur since the namespace must be explicitly qualified.

* Non-Breaking: The meaning of all existing code is preserved. The change is purely additive.

* Pure Syntax Sugar: The transformation is purely syntactic. `x.ns::f(args)` is entirely equivalent to `ns::f(x, args)` in every regard.

#### Considered Alternatives

1. Implicit UFCS: Previous proposals [N4174](#N4174) tried to make `x.f()` find free functions. This can lead to more complex overload resolution and breaking changes. This proposal is the antithesis of that: it requires explicit qualification.

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

    this is all good, until `std::string_view` itself gets a member named `starts_with` which takes a `const char*` that instead will be chosen.

    This is fine since both of the functions do the same thing in this case but what if they don't? you will get silent breakage and class writers can't gurantee what their api provides, and this is why this paper chosed an explicit syntax, it avoids the issues with implicit lookup and keeps the class interface stable and visually seperate from extensions because for example an API can provide a gurantee that all accesses of its const member functions are thread-safe, extension methods would break that as both `x.ext()` and `x.mem()` look the same, while this proposal provides explicit intent `x.Utils::ext()` is clear that it is not part of the official API and therefore the gurantee does not have to necessarily appy.

3. Overloading operators like `std::ranges`'s `operator|`
   *It works* but it is a workaround, and it results in bad debug codegen and worsens compile times that are already bad and requires weird functors to be used and it does not allow usage of other free functions you did not write.


This example was modified from this [cppreference](https://en.cppreference.com/w/cpp/ranges.html) example.

```cpp
#include <ranges>
constexpr auto even = [](int i) { return 0 == i % 2; };
constexpr auto square = [](int i) { return i * i; };
void nooptimize(volatile void*);
int main()
{
    auto const ints = {0, 1, 2, 3, 4, 5};
#if PIPE
    for (int i : ints | std::views::filter(even) | std::views::transform(square))
#else
    for (int i : std::views::transform(std::views::filter(ints, even), square))
#endif
    nooptimize(&i);
}
```

[Godbolt](https://godbolt.org/z/GT4edGYrj) shows doing `-DPIPE=1` increases the amount of assembly to 1213 from 1077 lines just for 
using the nicer syntax, and worse compile times due to the magic needed to implement it and burden on library developers

while with the paper 

```cpp
    ints.std::views::filter(even)
        .std::views::transform(square);
```

would result in quick debug codegen and same readability while compiling faster.

#### Possible Issues

The syntax `x.ns::f()` is the exact same as `x.Base::f()` wouldn't there be ambiguities if there is both a namespace and a base class named like that?

Lets see this code
```cpp
namespace Bar {
template<int,typename T> bool Baz(T,int) = delete;
};

template<class T>
bool f(T t)
{
    return t.Bar::Baz<0>(0); // the compiler can see that Bar::Baz is an entity at this point of definition
}
namespace Test {
struct Bar {
    static constexpr auto Baz = 0;
};
struct Foo : Bar {};
}
int main(){
    f(Test::Foo());
}
```

What is expected to happen if the paper is adopted? the compiler thinks `Bar::Baz<0>(0)` is a (wrong) chained comparison or a function call? Well even without this proposal the behavior seems inconsistent between compilers that implemented or didn't implement [CWG1089](#cwg1089).

[Godbolt](https://godbolt.org/z/81WP9qxq7) shows GCC and MSVC failing to compile they are looking up Bar::Baz in the namespace instead of the Base class. while Clang does look up the base class.


Proposed resolution.

This proposal aligns with the suggested resolution for CWG1089: if a name in a qualified-id within a template definition resolves to a template parameter or **a namespace**, that name is not subject to further lookup at instantiation time.

This resolution would allow types to call templated functions without manual disambiguation.

```cpp
template<class... Ts>
auto get_0(std::tuple<Ts...> t)
{
    return t.std::get<0>(); 
// old behavior:doesn't behave as expected, treats as member lookup instead of looking for namespace std.

// new behavior find that namespace std is an entity at this point of definition resolve to it and stop lookup.
}
```

If the old behavior was needed one could write

```cpp
using T = decltype(t)::std;
return t.T::get<0>();
```


The resolution would also make writing some functions possible and always correct.

```cpp
template<class T>
void call_f_nonvirtual(T* p)
{
    p->T::f(); // call the class direct f always.
}
struct T { void f(); }

struct S : T { void f(); }

S s;
call_f_nonvirtual(&s); // calls T::f! expected S::f.
```

Someone naively would think that this always call the type's member function "f" but it doesn't, it instead first says search for a base class called named T if not found see the typename parameter T, this is surprising and not really that useful to be the default and makes writing the above function with the intended semantics  impossible per the  current rules.


#### Limitations and Impact

Impact on the Standard: 
    This is an extension to the language that can later aid in library design, There is no existing code that will be broken.

Limitations:

1. This proposal does not aid in writing generic templates, as the namespace must be explicitly specified. Its primary benefit is syntactic convenience for non-generic code
    
    ```cpp
    // This works today and is not fully generic
    template <typename T>
    void func(T t) {
        t.foo(); // Only finds t.foo();
    }

    // This proposal does NOT enable this:
    template <typename T>
    void func(T t) {
        t.foo(); // Looks a member foo then an adl foo
    }
    ```

2. This proposal does not have a way currently to call hidden friends as members
    
    ```cpp
    struct S {
        friend void f(S& s) {}
    };

    S s;
    s.f(); // no member named f;
    s.::f(); // no global namespace member named f
    ```

The author is fine with those limitations currently.


#### Implementation

The author has made a [clang fork](https://github.com/ZXShady/llvm-project/blob/alternative_free_function_calling_syntax/
) that impelments the paper with resolving CWG1089.

[Test file](https://github.com/ZXShady/llvm-project/blob/alternative_free_function_calling_syntax/clang/ZXShady/alternative_free_function_calling_syntax.cpp)


#### References

* [N4174](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2014/n4174.pdf) - Call syntax: x.f(y) vs. f(x,y)
* [P3021R0](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2023/p3021r0.pdf) - Unified Call Syntax Herb Sutter 
* [N1585](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2004/n1585.pdf) - Extension Methods
* [UTFS Histroy](https://brevzin.github.io/c++/2019/04/13/ufcs-history/) - Barry Revzin UFCS Histroy.

* [CWG1089](https://www.open-std.org/jtc1/sc22/wg21/docs/cwg_active.html#1089) -  Template parameters in member selections.