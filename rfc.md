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

### Short circuiting

This RFC proposes full short circuiting. This means when the evaluation of one element in the chain fails the execution of the entire chain is aborted and the entire chain evaluates to `null`. The following elements are considered part of the chain.

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

## Syntax choice

The syntax has been chosen to indicate the precise place in the code that the short circuiting occurs.

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

## Explanation of short circuiting choice

A previous RFC for [null safe calls](https://wiki.php.net/rfc/nullsafe_calls) suggested not using short-circuiting. 

The position of this RFC is that full short-circuiting of the rest of the expression (**Iluja, I a not sure that is the right term**) is the correct choice, for the following reasons.


### Making the code easy to reason about, and avoids surprises.

Without full short-circuiting, for the following code

```
function f()
{
    echo "Hello, I am a side effect.";
}

$r = $x?->a(f())
```

The function f() would still be called, even when $x is null. 

The position of this RFC is that trying to understand that code is quite hard, and that full short-circuiting of the rest of the expression is the correct choice as otherwise side-effects in function calls for the code after the nullsafe operator will be highly surprising.


### Fits well when using other operators

The 'full short-circuit' choice is also the correct choice when dealing with other operators. Consider this code:

```php
$foo = null;
$baz = $foo?->bar()['baz'];
var_dump($baz);
```

Without full short circuiting, it could be equivalent to:

```php
// Without short circuiting:
if ($foo !== null) {
   $baz = $foo->bar()[];
}
else {
    $tmp = null;
    $baz = $tmp['baz'];
}
var_dump($baz);
// Notice: Trying to access array offset on value of type null
// NULL
```

And with full short-circuiting, it would be equivalent to:

```php
// With short circuiting:
if ($foo !== null) {
    $baz = $foo->bar()[];
    
} else {
    $baz = null;
}
var_dump($baz);
// NULL
```

With short circuiting the array access `['baz']` will be completely skipped no notice is emitted.


### Allows for nullsafe operator in write context

```php
$foo?->bar = 'bar';
var_dump($foo);
```


Without full short circuiting, it could be equivalent to:

```php
// Without short circuiting:

if ($foo !== null) {
    $foo->bar = 'bar';
} else {
    $tmp = null;
    $tmp->bar = 'bar';
}

// PHP Warning:  Creating default object from empty value
```


With full short circuiting, it would be equivalent to:

```php

if ($foo !== null) {
    $foo->bar = 'bar';
} 
// no else statement as nothing to do

```

Without full short circuiting the assignment to a nullsafe property would be illegal because it produces an r-value (a value that cannot be assigned to). With short circuiting if a nullsafe operation on the left hand side of the assignment fails, the assignment is simply skipped.

## Backward Incompatible Changes

There are no known backward incompatible changes in this RFC.

## Future Scope

Since PHP 7.4 a notice is emitted on array access on `null` (`null["foo"]`). Thus the operator `?[]` could also be useful (`$foo?["foo"]`). Unfortunately, this code introduces a parser ambiguity because of the ternary operator and short array syntax (`$foo?["foo"]:["bar"]`). Because of this complication the `?[]` operator is not part of this RFC.

Suggestions for a nullsafe function call syntax (`$callableOrNull?()`) are also outside the scope of what is being proposed by this RFC.

## Vote

...
