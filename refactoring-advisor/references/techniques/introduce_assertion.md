# Introduce Assertion

```
if (this.discountRate)
  base = base - (this.discountRate * base);
```

```
assert(this.discountRate >= 0);
if (this.discountRate)
  base = base - (this.discountRate * base);
```

## Motivation

Make implicit assumptions explicit by adding assertions that state conditions assumed to always be true. Assertions must not affect program correctness if removed -- use first-class validation (not assertions) for external input.

## Mechanics

- When you see that a condition is assumed to be true, add an assertion to state it.
- Since assertions should not affect the running of a system, adding one is always behavior-preserving.

## Example

Here's a simple tale of discounts. A customer can be given a discount rate to apply to all their purchases:

class Customer...

```js
applyDiscount(aNumber) {
  return (this.discountRate)
    ? aNumber - (this.discountRate * aNumber)
    : aNumber;
}
```

There's an assumption here that the discount rate is a positive number. I can make that assumption explicit by using an assertion. But I can't easily place an assertion into a ternary expression, so first I'll reformulate it as an if-then statement.

class Customer...

```js
applyDiscount(aNumber) {
  if (!this.discountRate) return aNumber;
  else return aNumber - (this.discountRate * aNumber);
}
```

Now I can easily add the assertion.

class Customer...

```js
applyDiscount(aNumber) {
  if (!this.discountRate) return aNumber;
  else {
    assert(this.discountRate >= 0);
    return aNumber - (this.discountRate * aNumber);
  }
}
```

In this case, I'd rather put this assertion into the setting method. If the assertion fails in `applyDiscount`, my first puzzle is how it got into the field in the first place.

class Customer...

```js
set discountRate(aNumber) {
  assert(null === aNumber || aNumber >= 0);
  this._discountRate = aNumber;
}
```

An assertion like this can be particularly valuable if it's hard to spot the error source -- which may be an errant minus sign in some input data or some inversion elsewhere in the code.
