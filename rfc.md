# PHP RFC: Nullsafe operator

* Date: 2020-05-25
* Author: Ilija Tovilo, tovilo.ilija@gmail.com
* Status: Under discussion
* Target Version: PHP 8.0
* Implementation: <https://github.com/php/php-src/pull/5619>
* Supersedes: <https://wiki.php.net/rfc/nullsafe_calls>

## Introduction

This RFC proposes a new operator nullsafe operator `?->` with full short-ciruiting.

## Proposal

It is fairly common to only want to call a method or fetch a property on the result of an expression it it is not `null`. This is usually done with if statements / ternary expressions and temporary variables.

```php
$bar = $foo !== null ? $foo->bar() : null;
$baz = $bar !== null ? $bar->baz : null;

// or

if ($foo === null) {
    return;
}
$bar = $foo->bar();

if ($bar === null) {
    return;
}
$baz = $bar->baz;
```

With the nullsafe operator `?->` this code could instead be expressed
like this:

```php
$baz = $foo?->bar()?->baz;
```

When the left hand side of the operator evaluates to `null` the execution of the entire chain will stop and evalute to `null`. When it not `null` it will behave exactly like the normal `->` operator.

### Short circuiting

This RFC proposes full short circuiting. This means when the evaluation of one element in the chain fails the execution of the entire chain is aborted and the entire chain evaluates to `null`. The following elements are considert part of the chain.

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
   $foo?->bar = $a?->b()->c();
// -------------------------- chain 1
//              ------------- chain 2
// If $foo is null chain 1 is aborted, `$a?->b()->c()` is not evaluated, the assignment is skipped
// If $a is null chain 2 is aborted, null is assigned to `$foo->bar`
// If `->b()` returns null a "Call to a member function c() on null" error is thrown

   $a?->b($c?->d);
// -------------- chain 1
//        ------  chain 2
// If $a is null chain 1 is aborted, `->b()` is not called, `$c?->d` is not evaluated
// If $c is null chain 2 is aborted, null is passed to `->b()`

   $foo++;
// ------ chain 1
// If $foo is null, chain 1 is aborted, ++ is skipped
```

## Backward Incompatible Changes

There are no known backward incompatible changes in this RFC.

## Future Scope

Since PHP 7.4 a notice is emitted on array access on `null` (`null["foo"]`). Thus the operator `?[]` could also be useful (`$foo?["foo"]`). Unfortunately, this code introduces a parser ambiguity because of the ternary operator and short array syntax (`$foo?["foo"]:["bar"]`). Because of this complication this operator is not part of this RFC.

## Vote

...
