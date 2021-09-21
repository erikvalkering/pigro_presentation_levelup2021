name: inverse
layout: true
class: middle, inverse

---

class: center

# PIGRO

Lazy Evaluation on Steroids

## .footnote[[github.com/erikvalkering/pigro](https://github.com/erikvalkering/pigro)]

```
constexpr auto lazy(auto &&f, concepts::lazy_function auto... deps) {
  using result_t = decltype(std::invoke(std::forward<decltype(f)>(f), deps(nullptr).value...));

  auto cache = std::optional<result_t>{};
  return compressed_tuple{ std::forward<decltype(f)>(f), deps... } << [=](std::nullptr_t, auto &&f, auto... deps) mutable {
      // TODO: instead of deps(nullptr), do something like access(deps), which does the nullptr trick
      const auto args = std::tuple{ deps(nullptr)... };

      auto changed = !cache || any(args, is_changed);
      if (changed) {
          const auto values = transform(args, value);
          const auto result = std::apply(f, values);

          changed = cache != result;
          cache = std::move(result);
      }

      return LazyResult{
          *cache,
          changed,
      };
  };
}
```
