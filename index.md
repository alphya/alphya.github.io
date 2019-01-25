#高階関数とは？
wikipediaには、こうあります

>高階関数（こうかいかんすう、英: higher-order function）とは、第一級関数をサポートしているプログラミング言語において、関数（手続き）を引数にしたり、あるいは関数（手続き）を戻り値とするような関数のことである。 

関数を扱う関数ですね！
例えば、関数`f(x)`を引数にとって微分をする関数は、次のようになります。

```cpp

// 関数 f(x) の x = a での微分係数の近似値を返します 7点近似です
template <typename F>
constexpr decltype(auto) diff(F && f, const double a) noexcept(noexcept(f(a)))
{
    // 値を変えながら試したところ、double だとhはこのくらいの値がいいようです（要検証）
	constexpr auto h = 1e-7;  
	return (f(a + 3 * h) - 9 * f(a + 2 * h) + 45 * f(a + h) - 45 * f(a - h) 
        + 9 * f(a - 2 * h) - f(a - 3 * h)) / (60 * h);
}
```

#高階関数にいろいろなものを渡したい！
上の`diff`は、今のところメンバ関数に対応してません。例えば以下の`mystruct::f`のような：

```cpp
struct mystruct
{
    // constexpr mystruct() = default; // 冗長でした
    constexpr decltype(auto) f(double x) noexcept { return x * x; }
}
```

日ごろから移植性の高いコードを書こうとしているあなたは、もちろんメンバ関数も引数に取れるようにすることでしょう。メンバ関数を呼ぶためには、メンバのポインタ型の値と、クラスのインスタンスが必要です（staticでないメンバなら）。これに対応した`diff`実装例は、例えば以下のようになるでしょう：

```cpp
//通常関数、スタティックメンバ関数、etc用
template <class F>
constexpr decltype(auto) diff(F && f, const double a) noexcept(noexcept(f(a)))
{
	constexpr auto h = 1e-7;  
	return (f(a + 3 * h) - 9 * f(a + 2 * h) + 45 * f(a + h) - 45 * f(a - h) 
        + 9 * f(a - 2 * h) - f(a - 3 * h)) / (60 * h);
}

//メンバ関数用  (2018/11/05 16:27 右辺値のインスタンスに対応)
template <class C, typename Ret, class Object, typename ... Args>
constexpr decltype(auto) diff(Ret(C::* && f)(Args...), Object && obj, const double a) noexcept(noexcept((obj.*f)(a)))
{
	constexpr auto h = 1e-7;
	return ((obj.*f)(a + 3 * h) - 9 * (obj.*f)(a + 2 * h) + 45 * (obj.*f)(a + h) - 45 * 
        (obj.*f)(a - h) + 9 * (obj.*f)(a - 2 * h) - (obj.*f)(a - 3 * h)) / (60 * h);
}
```

#STLではどうなってるの？
高階関数といえば、`std::bind`が代表的ですね。[ここ](https://faithandbrave.hateblo.jp/entry/2016/09/07/173344)によると、
>C++11から入ったstd::mem_fn()関数、std::bind()関数、std::result_ofメタ関数などは、どれもINVOKEコンセプトという仕様のもとに定義されています。

とのことです。INVOKEコンセプトとは、[cpprefjpの記事](https://cpprefjp.github.io/reference/concepts/Invoke.html)によると、

>第1引数がメンバ関数へのポインタの場合でも非静的メンバデータへのポインタの場合でも，第2引数がクラスオブジェクトへの参照の場合でもポインタの場合でもポインタっぽいものの場合でも，なんか知らんけどそれっぽく上手くいく ように取り計らった操作のこと

らしいです。詳しい定義は上記の記事にあります。例えば、INVOKEコンセプトでメンバ関数を呼び出したければ、第1引数にT型のメンバ関数へのポインタ、第2引数にT型のオブジェクト、それ以降に渡したい引数を渡します。このように、関数っぽいものなら、それっぽく呼び出せるようです。
　INVOKEの実装の参考になりそうなものを探したところ、どうやらINVOKEに従って関数呼び出しを行う`std::invoke`というものがSTLにあり、ヘッダ`<functional>`で定義されているようです。そして、この実装例は[cppreferenceのページ](https://ja.cppreference.com/w/cpp/utility/functional/invoke)にあります。
　しかし、これは`constexpr`に対応していません。[N4169](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2014/n4169.html)によると、 "`<functional>`ヘッダへのconstexprの追加が、別個の提案によって一貫して適用されることを前提としている" かららしいです（google翻訳）。ちなみに、このページに紹介されている`constexpr`実装へのリンクは切れています。正しくは[ここ](https://github.com/tomaszkam/proposals/blob/master/implementation/invoke/invoke_cpp11.hpp)です。高階関数を作る際には、このコードを参考にすれば、きっと汎用性に富んだINVOKEなインターフェイスを実装することができるでしょう！

#INVOKE実装例
こうなりました（闇）。上記の`invoke`の`constexpr`実装を`diff`に改変したものです。しかし、ここまでやる必要性があるのかはわかりませんね...

```cpp
#include <functional>
#include <limits>
#include <utility>

//        Copyright Tomasz Kamiński 2013 - 2014.
// Distributed under the Boost Software License, Version 1.0.
//   (See accompanying file ../LICENSE_1_0.txt or copy at
//          http://www.boost.org/LICENSE_1_0.txt)

#ifndef TYPE_TRAITS_HPP
#define TYPE_TRAITS_HPP

#include <type_traits>

namespace type_traits
{
	template<typename T>
	struct target_type
	{
		typedef void type;
	};

	template<typename Class, typename Member>
	struct target_type<Member Class::*>
	{
		typedef Class type;
	};

	//Is reference to pointer target or derived
	template<typename Object, typename Pointer>
	struct is_target_reference :
		public std::integral_constant<
		bool,
		std::is_reference<Object>::value &&
		std::is_base_of<
		typename target_type<Pointer>::type,
		typename std::decay<Object>::type
		>::value
		>
	{};

	namespace detail
	{
		//MPL or
		constexpr bool predicate_or()
		{
			return false;
		}

		template<typename Pred, typename... Preds>
		constexpr bool predicate_or(Pred&& pred, Preds&&... preds)
		{
			return pred || predicate_or(preds...);
		}

		template<typename Object, typename TargetType>
		struct is_wrapper_compatible_with_member_pointer_impl
			: std::integral_constant<bool, predicate_or(
				std::is_convertible<Object, TargetType&>{},
				std::is_convertible<Object, TargetType const&>{},
				std::is_convertible<Object, TargetType&&>{},
				std::is_convertible<Object, TargetType const&&>{}) >
		{};

		template<typename Object>
		struct is_wrapper_compatible_with_member_pointer_impl<Object, void>
			: std::false_type
		{};
	}

	template<typename Object, typename Pointer>
	struct is_wrapper_compatible_with_member_pointer
		: detail::is_wrapper_compatible_with_member_pointer_impl<Object, typename target_type<Pointer>::type>
	{};

	namespace detail
	{
		template<typename Object, typename Pointer>
		constexpr bool is_pointer_compatible_with_member_pointer_impl(typename std::decay<decltype(*std::declval<Object>())>::type*)
		{
			return is_wrapper_compatible_with_member_pointer<decltype(*std::declval<Object>()), Pointer>::value;
		}

		template<typename Object, typename Pointer>
		constexpr bool is_pointer_compatible_with_member_pointer_impl(...)
		{
			return false;
		}
	}

	template<typename Object, typename Pointer>
	struct is_pointer_compatible_with_member_pointer
		: public std::integral_constant<bool, detail::is_pointer_compatible_with_member_pointer_impl<Object, Pointer>(0)>
	{};
}

#endif //TYPE_TRAITS_HPP

// ここから、invokeをdiffに改変
// コピペするのなら、改変前のほうをお勧めします
// 可変長に対応していました

// なお、diff関数の後置で戻り値型を指定している部分で、
// INVOKEコンセプトのためのオーバーロード解決を行っているようです。

// 微分します 7点近似です
// メンバ関数用
template<typename Functor, typename Object>
constexpr auto diff(Functor&& functor, Object&& object, const double a)
->  typename std::enable_if<
	std::is_member_function_pointer<
	typename std::decay<Functor>::type
	>::value &&
	type_traits::is_target_reference<
	Object&&,
	typename std::decay<Functor>::type
	>::value,
	decltype((std::forward<Object>(object).*functor)(a))
>::type
{
	constexpr auto h = 1e-7;
	return ((object.*functor)(a + 3 * h) - 9 * (object.*functor)(a + 2 * h) 
		+ 45 * (object.*functor)(a + h) - 45 * (object.*functor)(a - h)
		+ 9 * (object.*functor)(a - 2 * h) - (object.*functor)(a - 3 * h)) / (60 * h); 
}

// これは何でしょう？
template<typename Functor, typename Object>
constexpr auto diff(Functor&& functor, Object&& object, const double a)
->  typename std::enable_if<
	std::is_member_function_pointer<
	typename std::decay<Functor>::type
	>::value &&
	!type_traits::is_target_reference<
	Object&&,
	typename std::decay<Functor>::type
	>::value,
	decltype(((*std::forward<Object>(object)).*functor)(a))
>::type
{
	constexpr auto h = 1e-7;
	return (((*object).*functor)(a + 3 * h) - 9 * ((*object).*functor)(a + 2 * h) 
		+ 45 * ((*object).*functor)(a + h) - 45 * ((*object).*functor)(a - h)
		+ 9 * ((*object).*functor)(a - 2 * h) - ((*object).*functor)(a - 3 * h)) / (60 * h); 
}

// これを呼んだところで、何をすればいいんでしょうか？
template<typename Functor, typename Object>
constexpr auto diff(Functor&& functor, Object&& object)
->  typename std::enable_if<
	std::is_member_object_pointer<
	typename std::decay<Functor>::type
	>::value &&
	type_traits::is_target_reference<
	Object&&,
	typename std::decay<Functor>::type
	>::value,
	decltype((std::forward<Object>(object).*functor))
>::type
{
	return std::forward<Object>(object).*functor;
}

// これも...
template<typename Functor, typename Object>
constexpr auto diff(Functor&& functor, Object&& object)
->  typename std::enable_if <
	std::is_member_object_pointer<
	typename std::decay<Functor>::type
	>::value &&
	!type_traits::is_target_reference<
	Object&&,
	typename std::decay<Functor>::type
	>::value,
	decltype((*std::forward<Object>(object)).*functor)
>::type
{
	return (*std::forward<Object>(object)).*functor;
}

//通常関数、スタティックメンバ関数用
template<typename Functor>
constexpr auto diff(Functor&& functor, const double a)
->  typename std::enable_if<
	!std::is_member_pointer<
	typename std::decay<Functor>::type
	>::value,
	decltype(std::forward<Functor>(functor)(a))
>::type
{
	    constexpr auto h = 1e-7;
		return (functor(a + 3 * h) - 9 * functor(a + 2 * h) + 45 * functor(a + h) 
			- 45 * functor(a - h) + 9 * functor(a - 2 * h) - functor(a - 3 * h)) / (60 * h);
}
```

`diff`の自分が使ったテスト用のコードの例：

```cpp
constexpr auto f(double x)
{
	return 300 * x + 600;
};

struct s
{
	constexpr s() = default;
	constexpr decltype(auto) f(double x)
	{
		return x * x * x;
	}
	static constexpr decltype(auto) s_f(double x)
	{
		return 15 * x * x;
	}
};

auto main() -> int
{
	constexpr auto f_ = [](double x)
	{
		return - 30000 / x;
	};

	s ss;

	constexpr auto result_lambda = diff(f_, 10);
	constexpr auto result_func = diff(f, 10);
	constexpr auto result_member_func = diff(&s::f, ss, 10);
	constexpr auto result_static_member_func = diff(&s::s_f, 10);

	std::cout << "理論値: 300" << "\n"
		<< result_lambda << "\n" 
		<< result_func << "\n"
		<< result_member_func << "\n"
		<< result_static_member_func << "\n";
}

/* output
理論値: 300
300
300
300
300
*/
```

上のコードの、何をしているかよくわからないところは、きっとメンバ変数か何かを呼び出したいのでしょう......？呼ばれることはない思いますが！

#おわりに
高階関数を実装する際は、全部実装しないまでも、INVOKEっぽくメンバ関数のためのオーバーロードも作っておけば、ちょっと汎用性が上がるかもしれませんね！

