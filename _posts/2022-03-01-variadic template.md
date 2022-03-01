# Variadic template

Variadic template 是 c++11标准引入的新功能，在c++11之前，只存在 Variadic function，以 `sum` 函数为例：
```c
int sum(int n_args, ...)
{
    int val = 0;
    va_list ap;
    int i;
    va_start(ap, n_args);
    for (i = 0; i < n_args; i++) {
        var += va_arg(ap, int);
    }
    va_end(ap);
    return val;
```
Variadic function 需要使用`va_`系列的宏函数操作 variadic argument，并且是类型不安全的。Variadic function 在运行期确定参数的类型，但是实际上，可以在编译器进行确定。
因此 c++11 引入 Variadic template 用于接受任意数量的参数，并在编译期确定参数类型，同时是类型安全的。

下面介绍一个简单的例子，实现上述的 sum 函数。
```cpp
// base case
template <typename T> 
T sum(T t) {
	return t;
}

template <typename T, typename... Ts> 
T sum(T t, Ts... ts) {
	return t + sum(ts...);
}

```
`sum` 函数可以接受任意数量的参数，只要这些参数的类型可以使用`+`运算符
`typename... Ts` 被成为 template parameter pack, `Ts... ts` 被成为 function parameter pack。
这里 template function 被组织成递归的形式，`T sum(T t)` 为 base case，`T sum(T t, Ts... ts)` 会不断递归，每次规模减少一个参数，直到遇见 base case。
对于 `sum(1,2,3,4,5)` 其递归形式如下：
```cpp
T sum(T, Args...) [T = int, Args = <int, int, int, int>]
T sum(T, Args...) [T = int, Args = <int, int, int>]
T sum(T, Args...) [T = int, Args = <int, int>]
T sum(T, Args...) [T = int, Args = <int>]
T sum(T) [T = int]
```

## Variadic template structure
下面介绍一下 `tuple`, 与`vector`不同，`tuple` 可以存储不同类型的数据，相当于在编译器可以不断的向 `tuple`类型中增加新的项，使用 variadic template 可以实现该效果
```cpp
namespace Tuple {
	// base case 
	template <class... Args> struct tuple {};

    // Important!!! tuple<T, Args...> derived from tuple<Args...>, which means tuple<T, Args...> has base class part tuple<Args...>
	template <class T, class... Args>
	struct tuple<T, Args...> : tuple<Args...> {
		tuple(T t, Args... ts) : tuple<Args...>(ts...), tail(t) {}
		T tail;
	};
    
    // elem_type_holder 存储第k个项的类型
	template <std::size_t, class> struct elem_type_holder;

	template <class T, class... Ts>
	struct elem_type_holder<0, tuple<T, Ts...>> {
		typedef T type;
	};

	template <std::size_t K, class T, class... Ts>
	struct elem_type_holder<K, tuple<T, Ts...>> {
		typedef typename elem_type_holder<K - 1, tuple<Ts...>>::type type;
	};

    // 使用 enable_if in type_trait, if k == 0, get 返回 tail,否则递归
	template <std::size_t k, class... Ts>
	typename std::enable_if_t<
		k == 0, typename elem_type_holder<0, tuple<Ts...>>::type&>
		get(tuple<Ts...>& t) {
		return t.tail;
	}

	template <std::size_t k, class T, class... Args>
	typename std::enable_if_t<
		k != 0, typename elem_type_holder<k, tuple<Args...>>::type&> get(tuple<T, Args...>& t) {
		tuple<Args...> base = t;
		return get<K - 1, args>(base);
	}
}
```
