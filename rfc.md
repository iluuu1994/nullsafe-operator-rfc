# PHP RFC: Nullsafe operator

* Date: 2020-06-02
* Author: Ilija Tovilo, tovilo.ilija@gmail.com
* Status: Under discussion
* Target Version: PHP 8.0
* Implementation: <https://github.com/php/php-src/pull/5619>
* Supersedes: <https://wiki.php.net/rfc/nullsafe_calls>

## Introduction

This RFC proposes a new operator nullsafe operator `?->` with full short circuiting.

## Proposal

It is fairly common to only want to call a method or fetch a property on the result of an expression if it is not `null`. 

Currently in PHP, checking for `null` leads to deeper nesting and repetition:

```php
$country =  null;

if ($session !== null) {
    $user = $session->user;

    if ($user !== null) {
        $address = $user->getAddress();

        if ($address !== null) {
            $country = $address->country;
        }
    }
}

// do something with $country
```

With the nullsafe operator `?->` this code could instead be written as:


```php
$country = $session?->user?->getAddress()?->country;

// do something with $country
```

When the left hand side of the operator evaluates to `null` the execution of the entire chain will stop and evalute to `null`. When it is not `null` it will behave exactly like the normal `->` operator.

## Short circuiting

### Introduction

Short circuiting refers to skipping the evaluation of an expression based on some given condition. Two common examples are the operators `&&` and `||`. There are three ways the nullsafe operator `?->` could implement short circuiting.

1\. Short circuiting for neither method arguments nor chained method calls

This complete lack of short circuiting is currently only found in Hack.

```php
null?->foo(bar())->baz();
```

Both the function `bar()` and the method `baz()` are called. `baz()` will cause a "Call to a member function on null" error. Evaluating method arguments makes it the most surprising of the three options. This was also the reason the last RFC moved back to discussion.

2\. Short circuiting for method arguments but not chained method calls

This is what would normally be considered lack of short circuiting.

```php
null?->foo(bar())->baz();
```

The function `bar()` is not called, the method `baz()` is. `baz()` will cause a "Call to a member function on null" error.

3\. Short circuiting for both method arguments and chained method calls

We'll refer to this as full short circuiting.

```php
null?->foo(bar())->baz();
```

Neither the function `bar()` nor the method `baz()` are called. There will be no "Call to a member function on null" error.

### Proposal

This RFC proposes full short circuiting. When the evaluation of one element in the chain fails the execution of the entire chain is aborted and the entire chain evaluates to `null`. The following elements are considered part of the chain.

* Array access (`[]`)
* Property access (`->`)
* Nullsafe property access (`?->`)
* Static property access (`::`)
* Method call (`->`)
* Nullsafe method call (`?->`)
* Static method call (`::`)
* Assignment (`=`, `+=`, `??=`, `= &`, etc.)
* Post/pre increment (`++`, `--`)

The following elements will cause new sub-chains.

* Right hand side of an assignment
* Arguments in a function call
* The expression in `[]` of an array access

Chains are automatically inferred. Only the closest chain will terminate. The following examples will try to illustrate.

```php
   $foo = $a?->b();
// --------------- chain 1
//        -------- chain 2
// If $a is null chain 2 is aborted, method b() isn't called, null is assigned to $foo

   $a?->b($c->d());
// --------------- chain 1
//        -------  chain 2
// If $a is null chain 1 is aborted, method b() isn't called, the expression `$c->d()` is not evaluated

   $a->b($c?->d());
// --------------- chain 1
//       --------  chain 2
// If $c is null chain 2 is aborted, method d() isn't called, null is passed to `$a->b()`

   $foo?->bar = $a->b();
// -------------------- chain 1
//              ------- chain 2
// If $foo is null chain 1 is aborted, `$a->b()` is not evaluated, the assignment is skipped

   $foo?->bar++;
// ------------ chain 1
// If $foo is null, chain 1 is aborted, ++ is skipped
```

### Rationale

#### Benefits

**1\. You can see which methods/properties return null**

```php
// Without short circuiting
$foo = null;
$foo?->bar()?->baz();

// With short circuiting
$foo = null;
$foo?->bar()->baz();
```

In this example `$foo` might be `null` but `bar()` will never return `null`. Without short circuiting every subsequent method call and property access in the chain will require the nullsafe operator. With short circuiting this isn't necessary which makes it more obvious which methods/properties might return `null`.

**2\. Allows for nullsafe operator in write context**

```php
$foo = null;
$foo?->bar = 'bar';
var_dump($foo);

// Without short circuiting:
// Fatal error: Can't use nullsafe result value in write context

// With short circuiting:
// NULL
```

Without short circuiting the assignment to a nullsafe property would be illegal because it produces an r-value (a value that cannot be assigned to). With short circuiting if a nullsafe operation on the left hand side of the assignment fails the assignment is simply skipped.

**3\. Mixing with other operators**

```php
$foo = null;
$baz = $foo?->bar()['baz'];
var_dump($baz);

// Without short circuiting: 
// Notice: Trying to access array offset on value of type null
// NULL

// With short circuiting 
// NULL
```

Since with short circuiting the array access `['baz']` will be completely skipped no notice is emitted. This might be less of a problem once we have a nullsafe array access operator `?[]`. 

#### Drawbacks

**1\. More rules**

Short circuiting must define which elements belong to the short circuiting chain and which do not. Not all of them might be immediately obvious but they should be intuitive for the most part.

**2\. Complexity**

It's also very likely that the implementation of the nullsafe operator with short circuiting will be slightly more complicated than without it. No short circutiing poses it's own set of complications though (like checking that `?->` can't be used in write context).

## Other languages

Lets look the most popular high-level programming languages (according to the [Stack Overflow 2020 survey](https://insights.stackoverflow.com/survey/2020#technology-programming-scripting-and-markup-languages)) and our sister language Hack to see how the nullsafe operator is implemented.

| Language     | Has nullsafe operator | Symbol | Has short circuiting |
|--------------|-----------------------|--------|----------------------|
| [JavaScript] | ✓                     | ?.     | ✓                    |
| [Python]     | ✗                     |        |                      |
| Java         | ✗                     |        |                      |
| [C#]         | ✓                     | ?.     | ✓                    |
| [TypeScript] | ✓                     | ?.     | ✓                    |
| [Kotlin]     | ✓                     | ?.     | ✗                    |
| [Ruby]       | ✓                     | &.     | ✗                    |
| [Swift]      | ✓                     | ?.     | ✓                    |
| [Rust]       | ✗                     |        |                      |
| Objective-C  | ✗\*                   |        |                      |
| [Dart]       | ✓                     | ?.     | ✗                    |
| Scala        | ✗†                    |        |                      |
| [Hack]       | ✓                     | ?->    | ✗‡                   |

\* In Object-C accessing properties and calling methods on `nil` is always ignored  
† Possible via [DSL](https://github.com/ThoughtWorksInc/Dsl.scala/blob/master/keywords-NullSafe/src/main/scala/com/thoughtworks/dsl/keywords/NullSafe.scala)  
‡ Hack evaluates method arguments even if the left hand side of `?->` is `null`

[JavaScript]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Optional_chaining
[Python]: https://www.python.org/dev/peps/pep-0505/
[C#]: https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/operators/null-conditional-operators
[TypeScript]: https://www.typescriptlang.org/docs/handbook/release-notes/typescript-3-7.html#optional-chaining
[Kotlin]: https://kotlinlang.org/docs/reference/null-safety.html#safe-calls
[Ruby]: http://mitrev.net/ruby/2015/11/13/the-operator-in-ruby/
[Swift]: https://docs.swift.org/swift-book/LanguageGuide/OptionalChaining.html
[Rust]: https://doc.rust-lang.org/stable/rust-by-example/error/option_unwrap/and_then.html#combinators-and_then
[Dart]: https://dart.dev/guides/language/language-tour#other-operators
[Hack]: https://docs.hhvm.com/hack/expressions-and-operators/member-selection#null-safe-member-access

8/13 languages have a nullsafe operator. 4/8 of those implement the nullsafe operator with short circuiting.

## Syntax choice

The `?` in `?->` denotes the precise place in the code that the short circuiting occurs. It closesly resembles the syntax of every other language that implements a nullsafe operator. 

## Backward Incompatible Changes

There are no known backward incompatible changes in this RFC.

## Future Scope

Since PHP 7.4 a notice is emitted on array access on `null` (`null["foo"]`). Thus the operator `?[]` could also be useful (`$foo?["foo"]`). Unfortunately, this code introduces a parser ambiguity because of the ternary operator and short array syntax (`$foo?["foo"]:["bar"]`). Because of this complication the `?[]` operator is not part of this RFC.

The same also goes for a nullsafe function call syntax (`$callableOrNull?()`).

## Vote

...
