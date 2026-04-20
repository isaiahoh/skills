# Replace Temp with Query

```
const basePrice = this._quantity * this._itemPrice;
if (basePrice > 1000)
  return basePrice * 0.95;
else
  return basePrice * 0.98;
```

```
get basePrice() {this._quantity * this._itemPrice;}

...

if (this.basePrice > 1000)
  return this.basePrice * 0.95;
else
  return this.basePrice * 0.98;
```

## Motivation

Replaces a temporary variable with a query method, so the calculation is reusable and no longer needs to be passed as a parameter when extracting functions. Only apply to temps that are assigned once and whose calculation yields the same result each time it is evaluated--snapshot variables like `oldAddress` are not suitable.

## Mechanics

- Check that the variable is determined entirely before it's used, and the code that calculates it does not yield a different value whenever it is used.

- If the variable isn't read-only, and can be made read-only, do so.

- Test.
- Extract the assignment of the variable into a function.

  If the variable and the function cannot share a name, use a temporary name for the function.

  Ensure the extracted function is free of side effects. If not, use Separate Query from Modifier.

- Test.
- Use Inline Variable to remove the temp.

## Example

Here is a simple class:

class Order...

```js
constructor(quantity, item) {
  this._quantity = quantity;
  this._item = item;
}

get price() {
  var basePrice = this._quantity * this._item.price;
  var discountFactor = 0.98;
  if (basePrice > 1000) discountFactor -= 0.03;
  return basePrice * discountFactor;
}
```

I want to replace the temps `basePrice` and `discountFactor` with methods.

Starting with `basePrice`, I make it `const` and run tests. This is a good way of checking that I haven't missed a reassignment--unlikely in a short function but common in larger ones.

class Order...

```js
get price() {
  const basePrice = this._quantity * this._item.price;
  var discountFactor = 0.98;
  if (basePrice > 1000) discountFactor -= 0.03;
  return basePrice * discountFactor;
}
```

I then extract the right-hand side of the assignment to a getter.

class Order...

```js
get price() {
  const basePrice = this.basePrice;
  var discountFactor = 0.98;
  if (basePrice > 1000) discountFactor -= 0.03;
  return basePrice * discountFactor;
}

get basePrice() {
  return this._quantity * this._item.price;
}
```

I test, and apply Inline Variable.

class Order...

```js
get price() {
  var discountFactor = 0.98;
  if (this.basePrice > 1000) discountFactor -= 0.03;
  return this.basePrice * discountFactor;
}
```

I then repeat the steps with `discountFactor`, first using Extract Function. In this case the extracted function must contain both assignments to `discountFactor`. I also set the original variable to `const`.

class Order...

```js
get price() {
  const discountFactor = this.discountFactor;
  return this.basePrice * discountFactor;
}

get discountFactor() {
  var discountFactor = 0.98;
  if (this.basePrice > 1000) discountFactor -= 0.03;
  return discountFactor;
}
```

Then, I inline:

```js
get price() {
  return this.basePrice * this.discountFactor;
}
```
