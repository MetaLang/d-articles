# std.variant Is Everything Cool About D

I recently read a great article by [Matt Kline](https://bitbashing.io/about.html) on how [std::visit is everything wrong with modern C++](https://bitbashing.io/std-visit.html). My C++ skills have grown rusty from disuse (I have long since left for the greener pastures of D), but I was curious as to how things had changed in my absence.

Despite my relative unfamiliarity with post-2003 C++, I had heard about the addition of a library-based sum type in
C++17. My curiosity was mildly piqued by the news, although like many new additions to C++ in the past decade, it's something
D has had [for years](https://github.com/dlang/phobos/blob/eec6be69edec9601f9f856afcd25a797e845c181/std/variant.d). 
Given the seemingly sensational title of Mr. Kline's article, I wanted to see just what was so bad about `std::visit`,
and get a feel for how well D's equivalent measures up.

My intuition was that the author was exaggerating for the sake of an interesting article. We've all heard the
oft-repeated criticism that C++ is complex and inconsistent (even some of its [biggest proponents](https://www.youtube.com/watch?v=KAWA1DuvCnQ) think so), but really, how bad could it be? After all, while the ergonomics of templates in D are much improved over C++, the underlying mechanism is broadly the same. I was dubious that `std::visit` could be much worse in practice, if at all, than `std.variant.visit`.

For the record, my intuition was completely and utterly wrong.


## Exploring std.variant

Before we continue, let me quickly introduce D's [std.variant](https://dlang.org/phobos/std_variant.html) module. The module centres around the [Variant](https://dlang.org/phobos/std_variant.html#.Variant) type; this is not actually a sum type like C++'s `std::variant`, but a type-safe container that can contain a value of any type. It also knows the type of the value it currently contains (if you've ever implemented a type-safe union, you'll realize why that part is important). This is akin to C++'s `std::any` as opposed to `std::variant`; very unfortunate, then, that C++ chose to use the name `variant` for its implementation of a sum type instead. _C'est la vie._ The type is used as follows:

```D
import std.variant;

Variant a;
a = 42;
assert(a.type == typeid(int));
a += 1;
assert(a == 43);

float f = a.get!float; //convert a to float
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

Luckily, `std.variant` _does_ provide a sum type as well: enter [Algebraic](https://dlang.org/phobos/std_variant.html#.Algebraic). The name `Algebraic` refers to [algebraic data types](https://en.wikipedia.org/wiki/Algebraic_data_type), of which one kind is a "sum type". Another example is the tuple, which is a "product type". 

In actuality, `Algebraic` is not a separate type from `Variant`; the former is an [alias](https://dlang.org/spec/declaration.html#alias) for the latter that takes a compile-time specified list of types. The values which an `Algebraic` may take on are limited to those whose type is in that list. For example, an `Algebraic!(int, string)` can contain a value of type `int` or `string`, but if you try to assign a `string` value to an `Algebraic!(float, bool)`, you'll get an error at compile time. The result is that we effectively get an in-library sum type for free! Pretty darn cool. It's used like this:

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
assert(!index.peek!Null); //assert that `index` does not contain a value of type Null
assert(index == size_t(3));

Option!size_t index2 = a.indexOf(117);
assert(index2.peek!Null);
```

The `peek` function takes a `Variant` as a runtime argument, and a type `T` as a compile-time argument. It returns a pointer to `T` that points to the `Variant`'s contained value _iff_ the `Variant` contains a value of type `T`; otherwise, the pointer is `null`.

<sup>_**Note:** I've made use of [Universal Function Call Syntax](https://dlang.org/spec/function.html#pseudo-member) to call the free function `indexOf` as if it were a member function of `int[]`._<sup>

In addition, just like C++, D's standard library has a special `visit` function that operates on `Algebraic`. It allows the user to supply a visitor for each type that may be held, which will be executed _if_ the `Algebraic` holds data of that type at runtime. More on that in a moment.


## To recap:

- `std.variant.Variant` is the equivalent of `std::any`. It is a type-safe container that can contain a value of 
any type.

- `std.variant.Algebraic` is the equivalent of `std::variant` and is a sum type similar to what you'd find in Swift, Haskell, Rust, etc. It is a thin wrapper over `Variant` that restricts what type of values it may contain via a compile-time specified list.

- `std.variant` provides a `visit` function akin to `std::visit` which dispatches based on the type of the contained value.

With that out of the way, let's now talk about what's wrong with `std::visit` in C++, and how D makes `std.variant.visit` much more pleasant to use by leveraging its powerful toolbox of compile-time introspection and code generation features.


## The problem with std::visit

The main problems with the C++ implementation are that - aside from clunkier template syntax - metaprogramming is very arcane and convoluted, and there are very few static introspection tools included out of the box. You get the absolute basics in `std::type_traits`, but that's it (there are a couple third-party solutions, which are appropriately horrifying and verbose). This makes implementing `std::visit` much more difficult than it has to be, and also pushes that complexity down to consumers of the library, which makes _using_ it that much more difficult as well. My eyes bled at this code from Mr. Kline's article which generates a visitor struct from the provided lambda functions:

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

Now, as he points out, this can be simplified down to the following in C++17:

```C++
template<class... Ts> struct overloaded : Ts... { using Ts::operator()...; };
template<class... Ts> overloaded(Ts...) -> overloaded<Ts...>;

template <class... Fs>
auto make_visitor(Fs... fs)
{
    return overload<Fs...>(fs...);
}
```

However, this code is still quite ugly (though I suspect I could get used to the elipses syntax eventually). Despite being a massive improvement on the preceding example, it's hard to get right when writing it, and hard to understand when reading it. To write (and more importantly, _read_) code like this, you have to know about:

- Templates
- Template argument packs
- [SFINAE](http://en.cppreference.com/w/cpp/language/sfinae)
- [User-defined deduction guides](http://en.cppreference.com/w/cpp/language/class_template_argument_deduction)
- The spread operator (...)
- Operator overloading
- C++-specific metaprogramming techniques

There's a lot of complicated template expansion and code generation going on, but it's hidden behind the scenes. And boy oh boy, if you screw something up you'd better _believe_ that the compiler is going to spit some supremely perplexing errors back at you.

<sup>_**Note:** As a fun exercise, try leaving out an overload for one of the types contained in your `variant` and marvel at the truly cryptic error message your compiler prints._</sup>

Here's an example from [cppreference.com](http://en.cppreference.com/w/cpp/utility/variant/visit) showcasing the _minimal_ amount of work necessary to use `std::visit`:

```C++
template<class... Ts> struct overloaded : Ts... { using Ts::operator()...; };
template<class... Ts> overloaded(Ts...) -> overloaded<Ts...>;

using var_t = std::variant<int, long, double, std::string>;
std::vector<var_t> vec = {10, 15l, 1.5, "hello"};

for (auto& v: vec) {
    std::visit(overloaded {
        [](auto arg) { std::cout << arg << ' '; },
        [](double arg) { std::cout << std::fixed << arg << ' '; },
        [](const std::string& arg) { std::cout << std::quoted(arg) << ' '; },
    }, v);
}
```
<sup>_**Note:** I don't show it in this article, but if you want to see this example re-written in D, it's [here](https://run.dlang.io/is/CKnGCk)._</sup>

Why is this extra work forced on us by C++, just to make use of `std::visit`? Users are stuck between a rock and a hard place: either write some truly stigmata-inducing code to generate a struct with the necessary overloads, or bite the bullet and write a new struct every time you want to use `std::visit`. Neither is very appealing, and both are a one-way ticket to Boilerplate Hell. The fact that you have to jump through such ridiculous hoops and write some ugly-looking boilerplate for something that _should be_ very simple is just... ridiculous. As Mr. Kline astutely puts it:

> The rigmarole needed for `std::visit` is entirely insane.

We can do better in D.


## The D solution

This is how the typical D programming would implement `make_visitor`, using D's powerful compile-time type introspection tools and code generation abilities:

```D
struct variant_visitor(Fs...)
{
    Fs fs;
    this(Fs fs) { this.fs = fs; }
    
    import std.traits: Parameters;
    static foreach(i, Fun; Fs) //Generate a different overload of opCall for each Fs
        auto opCall(Parameters!Fun params) { return fs[i](params); }
}

auto make_visitor(Fs...)(Fs fs)
{
    return variant_visitor!Fs(fs);
}
```



And... that's it. We're done. No pain, no strain, no bleeding from the eyes. It is a few more lines than the C++ version, granted, but in my opinion, it is also much simpler than the C++ version. To write and/or read this code, you have to understand a demonstrably smaller number of concepts:

- Templates
- Template argument packs
- [static foreach](https://github.com/dlang/DIPs/blob/master/DIPs/accepted/DIP1010.md)
- Operator overloading

However, a D programmer would not write this code. Why? Because `std.variant.visit` does not take a callable struct. From [the documentation](https://dlang.org/phobos/std_variant.html#.visit):

> Applies a **delegate** or **function** to the given Algebraic depending on the held type, ensuring that all types are handled by the visiting functions. Visiting handlers are passed in the template parameter list. **_(emphasis mine)_**

So `visit` only accepts a `delegate` or `function`, and figures out which one to pass the contained value to based on the functions' signatures. 

Why do this and give the user fewer options? D is what I like to call an anti-boilerplate language. In all things, D prefers the most direct method, and thus, `visit` takes a compile-time specified list of functions as template arguments. `std.variant.visit` may give the user fewer options, but _unlike_ `std::visit`, it does not require them to painstakingly create a new struct that overloads `opCall` for each case, or to waste time writing something like `make_visitor`. 

This also highlights the difference between the two languages themselves. D may sometimes give the user fewer options (although you're rarely, if ever, stuck needing that _one_ C++ feature that D doesn't have), but it is in service of making their lives easier through faster, safer code that is often an order of magnitude faster to iterate on (D's motto is _Fast code, fast_ for a reason). With `std.variant`, there's no messing around defining structs with callable methods or unpacking tuples or wrangling arguments; just straightforward, understandable code:

```D
Algebraic!(string, int, bool) v = "D rocks!";
v.visit!(
    (string s) => writeln("string: ", s),
    (int    n) => writeln("int: ", n),
    (bool   b) => writeln("bool: ", b),
);
```

And in a puff of efficiency, we've completely obviated all this machinery that C++ requires for `std::visit`, and greatly
simplified our users' lives.

For comparison, the C++ equivalent:

```C++
template<class... Ts> struct overloaded : Ts... { using Ts::operator()...; };
template<class... Ts> overloaded(Ts...) -> overloaded<Ts...>;

std::variant<std::string, int, bool> v = "C++ rocks?";
std::visit(overloaded {
    [](const std::string& s) { std::cout << s << '\n'; },
    [](int  n) { std::cout << n << '\n'; },
    [](bool b) { std::cout << b << '\n'; }
}, v);
```
<sup>_**Note:** Unlike the C++ version, the error message you get when you accidentally leave out a function to handle one of the types is comprehendable by mere mortals. [Check it out for yourself](https://run.dlang.io/is/ROA9Ac)._</sup>

As a bonus, our D example looks very similar to the built-in pattern matching syntax that you find
in many up-and-coming languages that take inspiration from their functional forebears, but is implemented completely _in user code_.

That's powerful.


## Other considerations

As my final point - if you'll indulge me for a moment - I'd like to argue with a strawman C++ programmer of my own creation. In his article, Mr. Kline also mentions the new [if constexpr](http://en.cppreference.com/w/cpp/language/if) feature added in C++17 (which of course, D has had for over a decade now). I'd like to forestall arguments from my strawman friend such as:

> But you're cheating! You can use the new `if constexpr` to simplify the code and cut out `make_visitor` entirely, just like in your D example!

Yes, you _could_ use `if constexpr`, but you shouldn't (and Mr. Kline explicitly rejects using it in his article).There are a few problems with this approach which make it all-around inferior. One, this method is error prone and inflexible in the case where you need to add a new type to your variant. Your old code will still compile but will now be _wrong_. Two, doing it this way is uglier and more complicated than just passing functions to `visit` directly. Three, the D version would _still_ blow C++ out of the water on readability. Consider:

```C++
//C++
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
//D
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

Which version of the code would _you_ want to have to read, understand, and modify? For me, at least, it's the second - no contest.
