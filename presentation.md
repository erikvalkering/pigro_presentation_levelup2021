name: inverse
layout: true
class: inverse

---

class: center, middle

# PIGRO

Lazy Evaluation on Steroids

## .footnote[[github.com/erikvalkering/pigro](https://github.com/erikvalkering/pigro)]

---

# Simple caching

```c++
auto lazy(auto f) {
    using result_t = decltype(f());

    auto cache = std::optional<result_t>{};
    return [=]() mutable {
        if (!cache) {
            cache = f(args...);
        }

        return *cache;
    };
}
```

---

count: false

# Simple caching

```c++
auto lazy(auto f) {
    using result_t = decltype(f());

    auto cache = std::optional<result_t>{};
    return [=]() mutable {
        if (!cache) {
            cache = f(args...);
        }

        return *cache;
    };
}
```

### Usage:

```c++
auto long_computation() -> int;

auto lazy_computation = lazy(long_computation);

auto answer_to_live = lazy_computation(); // may take a while...
assert(answer_to_live == 42);

// ...
auto universe_and_everything = lazy_computation(); // instantaneous!
```

---

# Simple function dependencies

```c++
auto lazy(auto f, `auto... deps`) {
    using result_t = decltype(f(`deps()...`));
*   using deps_t = decltype(std::tuple{deps()...});

    auto cache = std::optional<result_t>{};
*   auto deps_cache = std::optional<deps_t>{};
    return [=]() mutable {
*       const auto args = std::tuple{deps()...};
        if (!cache || `args != deps_cache`) {
*           cache = std::apply(f, args);
*           deps_cache = args;
        }

        return *cache;
    };
}
```

---

count: false

# Simple function dependencies

```c++
auto lazy(auto f, `auto... deps`) {
    using result_t = decltype(f(`deps()...`));
*   using deps_t = decltype(std::tuple{deps()...});

    auto cache = std::optional<result_t>{};
*   auto deps_cache = std::optional<deps_t>{};
    return [=]() mutable {
*       const auto args = std::tuple{deps()...};
        if (!cache || `args != deps_cache`) {
*           cache = std::apply(f, args);
*           deps_cache = args;
        }

        return *cache;
    };
}
```
