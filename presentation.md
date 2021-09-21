name: inverse
layout: true
class: inverse

---
background-image: url(Flash_Zootopia.png)
background-size: 70%
class: center, middle, title-page

# PIGRO

#### Lazy Evaluation on Steroids

## .footnote[[github.com/erikvalkering/pigro](https://github.com/erikvalkering/pigro)]

---
name: caching

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

--

### Usage:

```cpp
auto long_computation() -> int;

auto lazy_computation = lazy(long_computation);

auto answer_to_live = lazy_computation(); // may take a while...
assert(answer_to_live == 42);

// ...
auto universe_and_everything = lazy_computation(); // instantaneous!
```

---
name: dependencies
class: disable-highlighting

# Simple function dependencies

```cpp
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
template: dependencies
class: enable-highlighting
count: false

---
name: dependencies usage
count: false

# Simple function dependencies

### Usage:

```c++
auto get_mouse_pos() -> point_2d;
auto draw_mouse_cursor(const point_2d pos) -> void;

auto mouse_cursor = lazy(draw_mouse_cursor, get_mouse_pos);

// Rendering loop
while (true) {
    mouse_cursor();
}
```

---
template: dependencies usage
count: false

### Issues:
- Verbose: the need to create a lambda for `arrow`
- Slow: the image is loaded repeatedly from disk
- (Slow): the image is constantly being compared, even if it didn't change
