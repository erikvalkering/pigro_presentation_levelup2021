name: inverse
layout: true
class: inverse

---
background-image: url(Flash_Zootopia.png)
background-size: 70%
class: center, middle, title-page

# PIGRO

#### LAZY EVALUATION ON STEROIDS

## .footnote[[github.com/erikvalkering/pigro](https://github.com/erikvalkering/pigro)]

---
class: center, middle, title-page

# Part 1
#### Starting out lazy

---
# What is Lazy Evaluation anyway?

> In programming language theory, lazy evaluation, or call-by-need, is an evaluation strategy which .blue[***delays the evaluation of an expression until its value is needed***] (non-strict evaluation) and which also .orange[***avoids repeated evaluations***] (sharing). The sharing can reduce the running time of certain functions by an exponential factor over other non-strict evaluation strategies, such as call-by-name, which repeatedly evaluate the same function, blindly, regardless of whether the function can be memoized.

#### ***(Emphasis mine)***

## .footnote[[https://en.wikipedia.org/wiki/Lazy_evaluation](https://en.wikipedia.org/wiki/Lazy_evaluation)]

---

# Making Functions ***Lazy***

```c++
auto lazy(auto f) {
    using result_t = decltype(f());

    auto cache = std::optional<result_t>{};
    return [=]() mutable {
        if (!cache)
            cache = f();

        return *cache;
    };
}
```

--

### Usage:

```cpp
auto long_computation() -> int;

auto lazy_computation = lazy(long_computation);

auto answer_to_life = lazy_computation(); // may take a while...
assert(answer_to_life == 42);

// ...
auto universe_and_everything = lazy_computation(); // instantaneous!
```

---
name: function dependencies
class: disable-highlighting

# Making Functions ***Reactive***

```cpp
auto lazy(auto f, `auto... dependencies`) {
    using result_t = decltype(f(`dependencies()...`));
*   using dependencies_t = decltype(std::tuple{dependencies()...});

    auto cache = std::optional<result_t>{};
*   auto dependencies_cache = std::optional<dependencies_t>{};
    return [=]() mutable {
*       const auto args = std::tuple{dependencies()...};
        if (!cache || `args != dependencies_cache`) {
*           cache = std::apply(f, args);
*           dependencies_cache = args;
        }

        return *cache;
    };
}
```

---
template: function dependencies
class: enable-highlighting
count: false

---
count: false

# Making Functions ***Reactive***

### Usage:

```c++
auto draw_mouse_cursor(const point_2d pos, const image &icon) -> ui_object;
auto get_mouse_pos() -> point_2d;
auto load_image(const std::string_view filename) -> image;

auto arrow = [] { return load_image("arrow.png"); };
auto mouse_cursor = lazy(draw_mouse_cursor, get_mouse_pos, arrow);

// Rendering loop for a graphics editor
while (true) {
    mouse_cursor();
}
```

--

### Issues:
.ri-checkbox-blank-fill[] .failure[Verbosity:] the need to create a lambda for `arrow`

.ri-checkbox-blank-fill[] .failure[Performance:] the `image` is loaded repeatedly from disk

---
name: value dependencies
class: disable-highlighting

# Values as dependencies

```cpp
auto lazy(auto f, `std::invocable auto... dependencies`) {
    // ...as before...
}
```

```cpp
*auto ensure_invocable(std::invocable auto dependency) {
*   return dependency;
*}
*
*auto ensure_invocable(auto dependency) {
*   return [=] { return dependency; };
*}
```

```cpp
*auto lazy(auto f, auto... dependencies) {
*    return lazy(f, ensure_invocable(dependencies)...);
*}
```

---
template: value dependencies
class: enable-highlighting
count: false

> `std::invocable` is a C++20 **_concept_** which will match callable functions.

---
count: false

# Values as dependencies

### Usage (before):
```c++
auto draw_mouse_cursor(const point_2d pos, const image &icon) -> ui_object;
auto get_mouse_pos() -> point_2d;
auto load_image(const std::string_view filename) -> image;

auto arrow = [] { return load_image("arrow.png"); };
auto mouse_cursor = lazy(draw_mouse_cursor, get_mouse_pos, arrow);

// Rendering loop for a graphics editor
while (true) {
    mouse_cursor();
}
```

---
name: value dependencies verbosity
class: disable-highlighting

# Values as dependencies

### Usage:
```c++
auto draw_mouse_cursor(const point_2d pos, const image &icon) -> ui_object;
auto get_mouse_pos() -> point_2d;
auto load_image(const std::string_view filename) -> image;


auto mouse_cursor = lazy(draw_mouse_cursor, get_mouse_pos, `load_image("arrow.png")`);

// Rendering loop for a graphics editor
while (true) {
    mouse_cursor();
}
```

---
template: value dependencies verbosity
class: enable-highlighting
count: false

--

### Issues:

.ri-task-fill[] .success[Verbosity:] _**values**_ are now directly supported

--

.ri-checkbox-blank-fill[] .failure[Performance:] the `image` is loaded repeatedly from disk

---
name: value dependencies performance
class: disable-highlighting
count: false

# Values as dependencies

### Usage:
```c++
auto draw_mouse_cursor(const point_2d pos, const image &icon) -> ui_object;
auto get_mouse_pos() -> point_2d;
auto load_image(const std::string_view filename) -> image;


auto mouse_cursor = lazy(draw_mouse_cursor, get_mouse_pos, `lazy(load_image, "arrow.png")`);

// Rendering loop for a graphics editor
while (true) {
    mouse_cursor();
}
```
### Issues:
.ri-task-fill[] .success[Verbosity:] _**values**_ are now directly supported

.ri-task-fill[] .success[Performance:] `load_image()` is now called lazily

---
template: value dependencies performance
class: enable-highlighting
count: false

---
name: value dependencies bonus
class: disable-highlighting

# Reactivity all the way down

### Usage:
```c++
auto draw_mouse_cursor(const point_2d pos, const image &icon) -> ui_object;
auto get_mouse_pos() -> point_2d;
auto load_image(const std::string_view filename) -> image;
*auto get_drawing_mode() -> drawing_mode;

*auto get_mouse_icon_filename(const drawing_mode mode) {
*    return mode == drawing_mode::drawing ? "crosshair.png" : "arrow.png";
*}

*auto mode = lazy(get_drawing_mode);
*auto filename = lazy(get_mouse_icon_filename, mode);
*auto icon = lazy(load_image, filename);
auto mouse_cursor = lazy(draw_mouse_cursor, get_mouse_pos, `icon`);

// Rendering loop for a graphics editor
while (true) {
    mouse_cursor();
}
```

---
template: value dependencies bonus
class: enable-highlighting
count: false

--

### New issues:
.ri-checkbox-blank-fill[] .failure[Performance:] the `image` is constantly being compared, even if the drawing mode never changes

---
class: center, middle, title-page
# Part 2
#### Injecting some steroids
---
name: comparisons
class: disable-highlighting

# So what exactly is going on?

```cpp
auto lazy(auto f, std::invocable auto... dependencies) {
    using result_t = decltype(f(dependencies()...));
    using dependencies_t = decltype(std::tuple{dependencies()...});

    auto cache = std::optional<result_t>{};
    auto dependencies_cache = std::optional<dependencies_t>{};
    return [=]() mutable {
        const auto args = std::tuple{dependencies()...};
        if (!cache || `args != dependencies_cache`) {
            cache = std::apply(f, args);
            dependencies_cache = args;
        }

        return *cache;
    };
}
```

---
template: comparisons
class: enable-highlighting
count: false

--

> All the dependencies are unconditionally compared, even if they did **not** change.

---
name: lazy result
class: disable-highlighting

# Let's fix that (1) - Core Algorithm
```cpp
auto `lazy_core`(auto f, `lazy_function auto... dependencies`) {
    using result_t = decltype(f(`dependencies(nullptr).value`...));


    auto cache = std::optional<result_t>{};

    return [=](`std::nullptr_t`) mutable {

        if (!cache || `(dependencies(nullptr).is_changed || ...)`) {
            cache = `f(dependencies(nullptr).value...)`;
            `return LazyResult{*cache, true};`
        }

        return `LazyResult{*cache, false}`;
    };
}
```

> **Note:** In the `pigro` library, the double call to `dependencies(nullptr)` is optimized to a single call.

---
template: lazy result
class: enable-highlighting
count: false

---
# Let's fix that (2) - Some Helpers

```cpp
template<typename T, std::same_as<bool> B>
struct LazyResult {
    T value;
    B is_changed;
};

template<typename F>
concept lazy_function = requires(F f) {
    { f(nullptr).value };
    { f(nullptr).is_changed };
};

template<typename F>
concept lazy_function_with_facade = lazy_function<F> && std::invocable<F>;

auto lazy_value(auto value, auto is_changed) {
    return [=](std::nullptr_t) {
        return LazyResult{value, is_changed};
    };
}
```

### Usage:
```cpp
auto f = lazy_value(42, true);    // always considered to be changed
auto g = lazy_value(1729, false); // always considered up-to-date
```

---
# Let's fix that (3) - Ensuring laziness

```cpp
auto ensure_lazy_function(lazy_function auto dependency) {
    return dependency;
}

auto ensure_lazy_function(lazy_function_with_facade auto dependency) {
    return dependency;
}

auto ensure_lazy_function(std::invocable auto dependency) {
    return lazy(
        [=](int) mutable { return dependency(); },
        lazy_value(0, true)
    );
}

auto ensure_lazy_function(auto dependency) {
    return lazy_value(dependency, false);
}
```

---
# Let's fix that (4) - Wrapping Up
```cpp
auto facade(lazy_function auto f) {
    return [=]<typename... Args>(Args... args) mutable {
        if constexpr (std::same_as<std::tuple<Args...>, std::tuple<std::nullptr_t>>)
            return f(nullptr);
        else if (sizeof...(Args) == 0)
            return f(nullptr).value;
    };
}

auto lazy(auto f, auto... dependencies) {
    return facade(lazy_core(f, ensure_lazy_function(dependencies)...));
}
```

---
# Even smarter state propagation
```cpp
auto lazy_core(auto f, lazy_function auto... dependencies) {
    using result_t = decltype(f(dependencies(nullptr).value...));

    auto cache = std::optional<result_t>{};
    return [=](std::nullptr_t) mutable {

        if (!cache || (dependencies(nullptr).is_changed || ...)) {
            cache = f(dependencies(nullptr).value...);


            return LazyResult{*cache, true};
        }

        return LazyResult{*cache, false};
    };
}
```

---
name: less changes
class: disable-highlighting
count: false

# Even smarter state propagation
```cpp
auto lazy_core(auto f, lazy_function auto... dependencies) {
    using result_t = decltype(f(dependencies(nullptr).value...));

    auto cache = std::optional<result_t>{};
    return [=](std::nullptr_t) mutable {
*       auto changed = !cache || (dependencies(nullptr).is_changed || ...);
        if (`changed`) {
*           auto result = f(dependencies(nullptr).value...);

*           changed = result != cache;
*           cache = result;
        }

        return LazyResult{*cache, `changed`};
    };
}
```

---
template: less changes
class: enable-highlighting
count: false

---
# Overview
.smaller.left-column[
```cpp
template<typename F>
concept lazy_function = requires(F f) {
    { f(nullptr).value };
    { f(nullptr).is_changed };
};

template<typename T, std::same_as<bool> B>
struct LazyResult {
    T value;
    B is_changed;
};

auto lazy_core(auto f, lazy_function auto... dependencies) {
    using result_t = decltype(f(dependencies(nullptr).value...));

    auto cache = std::optional<result_t>{};
    return [=](std::nullptr_t) mutable {
        auto changed = !cache || (dependencies(nullptr).is_changed || ...);
        if (changed) {
            auto result = f(dependencies(nullptr).value...);

            changed = result != cache;
            cache = result;
        }

        return LazyResult{*cache, changed};
    };
}

template<typename F>
concept lazy_function_with_facade = lazy_function<F> && std::invocable<F>;

auto facade(lazy_function auto f) {
    return [=]<typename... Args>(Args... args) mutable {
        if constexpr (std::same_as<std::tuple<Args...>, std::tuple<std::nullptr_t>>)
            return f(nullptr);
        else if (sizeof...(Args) == 0)
            return f(nullptr).value;
    };
}
```
]

.smaller.right-column[
```cpp
auto lazy_value(auto value, auto is_changed) {
    return [=](std::nullptr_t) {
        return LazyResult{value, is_changed};
    };
}

auto ensure_lazy_function(lazy_function auto dependency) {
    return dependency;
}

auto ensure_lazy_function(lazy_function_with_facade auto dependency) {
    return dependency;
}

auto ensure_lazy_function(std::invocable auto dependency) {
    return lazy(
        [=](int) mutable { return dependency(); },
        lazy_value(0, true)
    );
}

auto ensure_lazy_function(auto dependency) {
    return lazy_value(dependency, false);
}












auto lazy(auto f, auto... dependencies) {
    return facade(lazy_core(f, ensure_lazy_function(dependencies)...));
}
```
]


---
# Comparison with hand-coded solution
.smaller.left-column[
#### HAND-CODED
```cpp
auto should_draw_mouse_cursor = false;
auto cache_pos = std::optional<point_2d>{};
auto cache_icon = std::optional<image>{};
auto cache_filename = std::optional<std::string>{};
auto cache_mode = std::optional<drawing_mode>{};

// Rendering loop for a graphics editor
while (true) {
    const auto pos = get_mouse_pos();
    if (cache_pos != pos) {
        cache_pos = pos;
        should_draw_mouse_cursor = true;
    }

    const auto mode = get_drawing_mode();
    if (cache_mode != mode) {
        cache_mode = mode;

        const auto filename = get_mouse_icon_filename(mode);
        if (cache_filename != filename) {
            cache_filename = filename;

            const auto icon = load_image(filename);
            if (cache_icon != icon) {
                cache_icon = icon;

                should_draw_mouse_cursor = true;
            }
        }
    }

    if (should_draw_mouse_cursor /* && cache_pos && cache_icon */) {
        draw_mouse_cursor(*cache_pos, *cache_icon);
    }
}
```
]

.smaller.right-column[
#### LAZY & REACTIVE
```cpp
auto mode = lazy(get_drawing_mode);
auto filename = lazy(get_mouse_icon_filename, mode);
auto icon = lazy(load_image, filename);
auto mouse_cursor = lazy(draw_mouse_cursor, get_mouse_pos, icon);

// Rendering loop for a graphics editor
while (true) {
    mouse_cursor();
}
```
]

---
# Conclusion and Further Work

- Under 70 lines of code
- https://godbolt.org/z/q8oG5da1W
