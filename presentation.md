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
name: caching

# Caching

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
name: function dependencies
class: disable-highlighting

# Function dependencies

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
template: function dependencies
class: enable-highlighting
count: false

---
count: false

# Function dependencies

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
name: function dependencies usage
count: false

# Function dependencies

### Usage:

```c++
auto get_mouse_pos() -> point_2d;
auto draw_mouse_cursor(const point_2d pos, `const image &icon`) -> void;
*auto load_image(const std::string_view filename) -> image;

*auto arrow = [] { return load_image("arrow.png"); };
auto mouse_cursor = lazy(draw_mouse_cursor, get_mouse_pos, `arrow`);

// Rendering loop
while (true) {
    mouse_cursor();
}
```

---
template: function dependencies usage
count: false

### Issues:
.hide-li[
- .ri-checkbox-blank-fill[] Verbosity: the need to create a lambda for `arrow`
- .ri-checkbox-blank-fill[] Performance: the image is loaded repeatedly from disk
]

---
# Value dependencies

```cpp
auto lazy(auto f, `std::invocable auto... deps`) {
    // ...as before...
}
```

```cpp
auto ensure_invocable(auto dep) {
    if constexpr (std::invocable<decltype(dep)>)
        return dep;
    else
        return [=] { return dep; };
}
```

```cpp
auto lazy(auto f, auto... deps) {
    return lazy(f, ensure_invocable(deps)...);
}
```

---
# Value dependencies

### Usage:
```c++
auto get_mouse_pos() -> point_2d;
auto draw_mouse_cursor(const point_2d pos, const image &icon) -> void;
auto load_image(const std::string_view filename) -> image;

auto mouse_cursor = lazy(draw_mouse_cursor, get_mouse_pos, `load_image("arrow.png")`);

// Rendering loop
while (true) {
    mouse_cursor();
}
```
--
### Issues:

.ri-task-fill[] .success[Verbosity:] _**values**_ are now directly supported

--

.ri-checkbox-blank-fill[] .failure[Performance:] the `image` is loaded repeatedly from disk

---
count: false

# Value dependencies

### Usage:
```c++
auto get_mouse_pos() -> point_2d;
auto draw_mouse_cursor(const point_2d pos, const image &icon) -> void;
auto load_image(const std::string_view filename) -> image;

auto mouse_cursor = lazy(draw_mouse_cursor, get_mouse_pos, `lazy(load_image, "arrow.png")`);

// Rendering loop
while (true) {
    mouse_cursor();
}
```
### Issues:
.ri-task-fill[] .success[Verbosity:] _**values**_ are now directly supported

.ri-task-fill[] .success[Performance:] `load_image()` is now called lazily
]

---

count: false

# Value dependencies

### Usage:
```c++
auto get_mouse_pos() -> point_2d;
auto draw_mouse_cursor(const point_2d pos, const image &icon) -> void;
auto load_image(const std::string_view filename) -> image;
*auto get_drawing_mode() -> drawing_mode;

*auto get_mouse_icon(const drawing_mode mode) {
*    if (mode == drawing_mode::drawing)
*        return load_image("crosshair.png");
*    else
*        return load_image("arrow.png");
*}

*auto icon = lazy(get_mouse_icon, get_drawing_mode);
auto mouse_cursor = lazy(draw_mouse_cursor, get_mouse_pos, `icon`);

// Rendering loop
while (true) {
    mouse_cursor();
}
```
--

### New issues:
.ri-checkbox-blank-fill[] .failure[Performance:] the `image` is constantly being compared, even if the drawing mode never changes
- .ri-checkbox-blank-fill[] Performance: the image is constantly being compared, even if it didn't change
]
