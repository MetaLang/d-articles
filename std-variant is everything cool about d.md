# std.variant Is Everything Cool About D

I recently read a great article by [Matt Kline](https://bitbashing.io/about.html) on how [std::visit is everything wrong with modern C++](https://bitbashing.io/std-visit.html). Being quite
out of practice with C++ (I have long since left for the greener pastures of D), I was curious as to how things had changed
in my absence with all the new features added in the last few major revisions to the language.

Despite my relative unfamiliarity with post-2003 C++, I had heard about the addition of a library-based sum type in
C++17. My curiosity was mildly piqued by the news, but like many new additions to C++ in the past decade, it's something
D has had [for years](https://github.com/dlang/phobos/blob/eec6be69edec9601f9f856afcd25a797e845c181/std/variant.d). 
Given the oft-repeated criticism that C++ is a convoluted, complex mess, and the seemingly sensational title of Mr. 
Kline's article, I wanted to see just what was so bad about `std::visit`, and get a feel for how well D's equivalent 
measures up.

My intuition going in was that the author was exaggerating for the sake of an interesting article. After all, while
the ergonomics of D templates are much improved over C++, the underlying mechanics are broadly the same. I was dubious
that the code could be much worse than if `std::visit` were implemented in D.

For the record, my intuition was completely and utterly wrong.


## Exploring std.variant

Before we continue, let me quickly introduce D's [std.variant](https://dlang.org/phobos/std_variant.html) module. The module centres around the [Variant](https://dlang.org/phobos/std_variant.html#.Variant)
type, which is not actually a sum type like C++'s `std::variant`, but a type-safe container that can contain a value of 
any type. This is akin to C++'s `std::any` as opposed to `std::variant`, with the unfortunate coincidence that C++ used that same
name for its implementation of a sum type instead. The `Variant` type is used as follows:

```D
import std.variant;

Variant a;
a = 42;
assert(a.type == typeid(int));
a += 1;
assert(a == 43);

float f = a.get!float; //Coerce a to float
assert(f == 43);
a /= 2;
f /= 2;
assert(a == 21 && f == 21.5);

a = "D rocks!";
assert(a.type == typeid(string));

Variant b = new Object();
Variant c = b;
assert(c is b); //c and b point to the same object
b /= 2; //Error: no possible match found for Variant / int
```

That said, `std.variant` _does_ provide a sum type as well: enter [Algebraic](https://dlang.org/phobos/std_variant.html#.Algebraic). The name `Algebraic` refers to 
[algebraic data types](https://en.wikipedia.org/wiki/Algebraic_data_type), of which one kind is a "sum type". Another
example is the tuple, called a "product type". In actuality, `Algebraic` is not a separate type from `Variant`; the
former is an [alias](https://dlang.org/spec/declaration.html#alias) for the latter that takes a compile-time specified list of which types it may contain.
This effectively gives us an in-library sum type for free! Pretty darn cool. It's used like this:

```D
alias Null = typeof(null); //for convenience
alias Option(T) = Algebraic!(T, Null);

Option!size_t indexOf(int[] haystack, int needle) {
    foreach (size_t i, int n; haystack)
        if (n == needle)
            return Option!size_t(i);
    return Option!size_t(null);
}

int[] a = [4, 2, 210, 42, 7];
Option!size_t index = a.indexOf(42); //call indexOf like a method using UFCS
assert(!index.peek!Null && index == size_t(3));

Option!size_t index2 = a.indexOf(117);
assert(index2.peek!Null);
```


## To recap:

- `std.variant.Variant` is the equivalent of `std::any::any`. It is a type-safe container that can contain a value of 
any type.

- `std.variant.Algebraic` is the equivalent of `std::variant::variant` and is a sum type similar to what you'd find in 
Swift, Haskell, Rust, etc. It is a thin wrapper over `Variant` that restricts what types it may contain to a comile-time
specified list.

With that out of the way, let's now talk about what's wrong with C++'s implementation of `std::visit`, and how 
D greatly improves on it using its powerful toolbox of compile-time, introspective features.


## Problems with std::visit and how D fixes them

The main problem with the C++ implementation is that - aside from clunkier template syntax - metaprogramming is very arcane
and convoluted, and there are almost no static introspection tools included out of the box, save for the absolute
basics in `std::type_traits` (there are a few third-party solutions, which are appropriately horrifying and verbose).
This makes implementing `std::visit` much more difficult than it has to be, and also pushes that complexity down to the
consumer of the library; my eyes bled at this code from Mr. Kline's article:

```C++
template <class... Fs>
struct overload;

template <class F0, class... Frest>
struct overload<F0, Frest...> : F0, overload<Frest...>
{
    overload(F0 f0, Frest... rest) : F0(f0), overload<Frest...>(rest...) {}

    using F0::operator();
    using overload<Frest...>::operator();
};

template <class F0>
struct overload<F0> : F0
{
    overload(F0 f0) : F0(f0) {}

    using F0::operator();
};

template <class... Fs>
auto make_visitor(Fs... fs)
{
    return overload<Fs...>(fs...);
}
```

Now as he points out, this can be simplified down to the following in C++17:

```C++
template<class... Ts> struct overloaded : Ts... { using Ts::operator()...; };
template<class... Ts> overloaded(Ts...) -> overloaded<Ts...>;
```

But this is still ugly, and it is still very complicated to write and understand. There's a lot of moving parts here.
The fact that someone implementing `make_visitor` has to jump through such ridiculous hoops for something so simple is
just... ridiculous. As Mr. Kline so eloquently puts it: _"The rigmarole needed for `std::visit` is entirely insane."_

We can do better in D:

```D
struct variant_visitor(Fs...)
{
    Fs fs;
    this(Fs fs) { this.fs = fs; }
    
    import std.traits;
    static foreach(i, Fun; Fs) //Generate a different overload of opCall for each Fs
        ReturnType!Fun opCall(Parameters!Fun params) { return fs[i](params); }
}

auto make_visitor(Fs...)(Fs fs)
{
    return variant_visitor!Fs(fs);
}
```

And... that's it. We're done. No pain, no strain, and easy, understandable code. We can then put our much-simplified
implementation to work:

```D
Algebraic!(string, int, bool) v = "D rocks!";
auto visitor = make_visitor(
    (string s) => writeln("string: ", s),
    (int    n) => writeln("int: ", n),
    (bool   b) => writeln("bool: ", b),
);
v.visit(visitor);
```

Except for one thing: this code won't compile. Why? Because the version of `visit` in D's standard library does not 
accept a callable struct. Sorry to mislead you, but D has no need for such an awkward construction. D is what I like to 
call an anti-boilerplate language. In all things, D prefers the more direct method, thus, `visit` takes a compile-time 
specified list of functions as template arguments. No messing around defining structs with callable methods and 
unpacking tuples and wrangling arguments; just pure, simple, straightforward code:

```D
Algebraic!(string, int, bool) v = "D rocks!";
v.visit!(
    (string s) => writeln("string: ", s),
    (int    n) => writeln("int: ", n),
    (bool   b) => writeln("bool: ", b),
);
```

And in a puff of efficiency, we've completely obviated all this machinery necessary to use `std::visit` and greatly
simplified our users' lives. As a bonus, this looks very similar to the built-in pattern matching syntax that you find
in many up-and-coming languages, but implemented completely _in user code_. That's very powerful.


## Other considerations

_"But you're cheating! You can just use the new `if constexpr` to simplify the code and cut out `make_visitor`
entirely, just like in your D example!"_

Yes, that's true. However, for one thing, doing it that way is still more 
complicated and uglier than just passing functions to `visit` directly, and two, the D version would _still_ blow C++ 
out of the water on readability. Consider:

```C++
visit([](auto& arg) {
    using T = std::decay_t<decltype(arg)>;

    if constexpr (std::is_same_v<T, string>) {
        printf("string: %s\n", arg.c_str());
    }
    else if constexpr (std::is_same_v<T, int>) {
        printf("integer: %d\n", arg);
    }
    else if constexpr (std::is_same_v<T, bool>) {
        printf("bool: %d\n", arg);
    }
}, v);
```

vs.

```D
v.visit!((arg) {
    alias T = typeof(arg);

    static if (is(T == string)) {
        writeln("string: ", arg);
    }
    else static if (is(T == int)) {
        writeln("int: ", arg);
    }
    else static if (is(T == bool)) {
        writeln("bool: ", arg);
    }
});
```

Which version of the code would _you_ want to have to read, understand, and modify? For me, at least, it's the second -
no contest.
