---
layout: post
title: A terse introduction to value-templated functions in C++ and a constant-caller gadget
---

In C++, one use of value-templates is to effectively specify a family of function without so much extra typing.

For example, this template function specifies a family of functions:

```
template <int constant_factor>
int multiply_by(int inp) {
    return inp * constant_factor;
}
```

Each member in this family of functions takes an integer argument and returns an integer. Members of the family would include `multiply_by<10>` which returns 10 times the input, and `multiply_by<0>` which, incidentally, always returns 0. Typing these expressions in a c++ program is the cue to the compiler to instantiate the named function (that is, create a version of the function with the specified constant). (Side note: these functions are just as proper functions as those that are not instantiated from a template! For example, you can refer to them in a function pointer!)

At the time of instantiation, the compiler knows the value of `constant_factor`. And this is not just true in practice due to a common optimization. Rather, the programmer fully intended this. (The programmer also, by the way, is also telling the compiler it's OK to generate separate machine code for each instantiation!). In this case, this means that, `multiply_by<16>` is likely to be optimized to make use of bit-shifts and `multiply_by<0>` will be optimized to ignore the input and return `0`. (With inlining, the expression `multiply_by<0>(some_int_variable)` will almost certainly be replaced by the constant `0`).

Having constants available at compile time is good for more than just encouraging the compiler to go hog-wild on optimizing arithmetic.

* A conditional that depends on a compile-time constant can be eliminated by the compiler in some circumstances. (Reducing branches can significantly speed up a program!). 

* A constant integer can be used as a `case:` label in a switch statement.

* Types can depend on constant integers! (Techniques for that would include e.g. `std::conditional` and full specialization). For example, maybe an integer type is determined by a constant `bool` and a constant `char`; the former representing signedness and the latter representing width. 

* An array can be *statically* allocated using a constant integer as a length.

All so far has been good, but quickly one comes upon the questions: how do we *make* these constant integers? It turns out that the usual answer is surprisingly bland, since at some level, the only real option is to type them into the source code!

We quickly end up with code like this:

```
template <bool elem_signedness, char elem_width, bool must_check_for_null>
void process_dynamic_array(void * head, size_t len) {
  // whatever...
}
void process_dynamic_array(void * head, size_t len, bool elem_signedness, char elem_width, bool must_check_for_nulls) {
  ASSERT(elem_width == 1 || elem_width == 2 || elem_width == 4 || elem_width == 8);
  switch (elem_width) {
    case 1:
    if (elem_signedness) {
      if (must_check_for_nulls) {
        process_dynamic_array<true, 1, true>(head, len);
      } else {
        process_dynamic_array<true, 1, false>(head, len);
      }
    } else {
      if (must_check_for_nulls) {
        process_dynamic_array<false, 1, true>(head, len);
      } else {
        process_dynamic_array<false, 1, false>(head, len);
      }
    }
    case 2:
    if (elem_signedness) {
      if (must_check_for_nulls) {
        process_dynamic_array<true, 2, true>(head, len);
      } else {
        process_dynamic_array<true, 2, false>(head, len);
      }
    } else {
      if (must_check_for_nulls) {
        process_dynamic_array<false, 2, true>(head, len);
      } else {
        process_dynamic_array<false, 2, false>(head, len);
      }
    }
    case 4:
    if (elem_signedness) {
      if (must_check_for_nulls) {
        process_dynamic_array<true, 4, true>(head, len);
      } else {
        process_dynamic_array<true, 4, false>(head, len);
      }
    } else {
      if (must_check_for_nulls) {
        process_dynamic_array<false, 4, true>(head, len);
      } else {
        process_dynamic_array<false, 4, false>(head, len);
      }
    }
    case 8:
    if (elem_signedness) {
      if (must_check_for_nulls) {
        process_dynamic_array<true, 8, true>(head, len);
      } else {
        process_dynamic_array<true, 8, false>(head, len);
      }
    } else {
      if (must_check_for_nulls) {
        process_dynamic_array<false, 8, true>(head, len);
      } else {
        process_dynamic_array<false, 8, false>(head, len);
      }
    }
  }
}

```

So obviously, things get unwieldy quick!

Now, on to the solution! 

To handle just one level at a time, I created this gadget:

```
    template <typename T, T... u>
    struct constant_call {
        typedef T value_type;
        // TODO: Create a version that returns the result of f(T).
        template <typename Callback>
        static void call(T v, Callback f) {
            const char zero = 0;
            bool found = false;
            static_assert(sizeof...(u), "constant_call with 0 options will always fail.");
            char dummy[sizeof...(u)] = { ( v == u ? (found = true, f(std::integral_constant<T, u>()), zero) : zero)... };
	    // ^See "Brace-enclosed initializers" at https://en.cppreference.com/w/cpp/language/parameter_pack
            (void)(dummy); // Don't warn about unused "dummy". It was only there to iterate over u...
            if (!found) throw "Bad value!";
        }
    };
```

That lets us write code like this:

```
void process_dynamic_array(void * head, size_t len, bool elem_signedness, char elem_width, bool must_check_for_nulls) {
  constant_call<char, 1, 2, 4, 8>::call(width, [&] (auto width_t) {
    constant_call<bool, true, false>::call(elem_signedness, [&] (auto signedness_t) {
      constant_call<bool, true, false>::call(must_check_for_nulls, [&] (auto check_nulls_t) {
        process_dynamic_array<signedness_t(), width_t(), check_nulls_t()>(head, len);
      });
    });
  });
}
```

But we can definitely do better! This gadget allows us to chain the arguments, consolidating into one callback!

```
    template <typename... constant_calls>
    struct chain;

    template <>
    struct chain<> {
        template <typename Callback, typename... Args>
        void call_impl(Callback c, Args... args) const {
            c(args...);
        }
        template <typename T, T... u>
        chain<constant_call<T, u...>> add(T v) const {
            return { *this, std::move(v) };
        }
        template <typename Callback>
        void call(Callback&& c) const {
            call_impl(std::forward<Callback>(c));
        }
    };

    template <typename calls_head, typename... calls_tail>
    class chain<calls_head, calls_tail...> {
        typedef typename calls_head::value_type value_t;
        typedef chain<calls_tail...> predecessor_t;
        const predecessor_t & predecessor;
        const value_t v;
    public:
        template <typename Callback, typename... Args>
        void call_impl(Callback&& c, Args... args) const {
            calls_head::call(v, [&] (auto new_arg) {
                    predecessor.call_impl(std::forward<Callback>(c), new_arg, args...);
                });
        }
        chain() = delete;
        chain(const predecessor_t & predecessor, value_t v) :
            predecessor(predecessor), v(std::move(v))
        {
        }
        template <typename T, T... u>
        chain<constant_call<T, u...>, calls_head, calls_tail...> add(T v) const {
            return { *this, std::move(v) };
        }
        template <typename Callback>
        void call(Callback&& c) const {
            call_impl(std::forward<Callback>(c));
        }
    };
```

That will let us write code like this:

```
void process_dynamic_array(void * head, size_t len, bool elem_signedness, char elem_width, bool must_check_for_nulls) {
  chain<>()
  .add<char, 1, 2, 3, 8>(width)
  .add<bool, true, false>(elem_signedness)
  .add<bool, true, false>(must_check_for_nulls)
  .call([&] (auto width_t, auto signedness_t, auto check_nulls_t) {
    process_dynamic_array<signedness_t(), width_t(), check_nulls_t()>(head, len);
  });
}
```