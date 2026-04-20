# Extract Variable

```
return order.quantity * order.itemPrice -
  Math.max(0, order.quantity - 500) * order.itemPrice * 0.05 +
  Math.min(order.quantity * order.itemPrice * 0.1, 100);
```

```
const basePrice = order.quantity * order.itemPrice;
const quantityDiscount = Math.max(0, order.quantity - 500) * order.itemPrice * 0.05;
const shipping = Math.min(basePrice * 0.1, 100);
return basePrice - quantityDiscount + shipping;
```

## Motivation

Introduces a named local variable for a complex sub-expression, making the surrounding logic easier to follow and debug.

## Mechanics

- Ensure that the expression you want to extract does not have side effects.

- Declare an immutable variable. Set it to a copy of the expression you want to name.

- Replace the original expression with the new variable.

- Test.

- If the expression appears more than once, replace each occurrence with the variable, testing after each replacement.

## Example

Start with a simple calculation:

```js
function price(order) {
  //price is base price - quantity discount + shipping
  return order.quantity * order.itemPrice -
    Math.max(0, order.quantity - 500) * order.itemPrice * 0.05 +
    Math.min(order.quantity * order.itemPrice * 0.1, 100);
}
```

Recognize that the base price is the multiple of quantity and item price. Create and name a variable for it:

```js
function price(order) {
  //price is base price - quantity discount + shipping
  const basePrice = order.quantity * order.itemPrice;
  return order.quantity * order.itemPrice -
    Math.max(0, order.quantity - 500) * order.itemPrice * 0.05 +
    Math.min(order.quantity * order.itemPrice * 0.1, 100);
}
```

Replace the expression with the variable:

```js
function price(order) {
  //price is base price - quantity discount + shipping
  const basePrice = order.quantity * order.itemPrice;
  return basePrice -
    Math.max(0, order.quantity - 500) * order.itemPrice * 0.05 +
    Math.min(order.quantity * order.itemPrice * 0.1, 100);
}
```

The same expression appears later, so replace it there too:

```js
function price(order) {
  //price is base price - quantity discount + shipping
  const basePrice = order.quantity * order.itemPrice;
  return basePrice -
    Math.max(0, order.quantity - 500) * order.itemPrice * 0.05 +
    Math.min(basePrice * 0.1, 100);
}
```

Extract the quantity discount:

```js
function price(order) {
  //price is base price - quantity discount + shipping
  const basePrice = order.quantity * order.itemPrice;
  const quantityDiscount = Math.max(0, order.quantity - 500) * order.itemPrice * 0.05;
  return basePrice -
    quantityDiscount +
    Math.min(basePrice * 0.1, 100);
}
```

Finally, extract shipping. The comment can be removed since the code now says the same thing:

```js
function price(order) {
  const basePrice = order.quantity * order.itemPrice;
  const quantityDiscount = Math.max(0, order.quantity - 500) * order.itemPrice * 0.05;
  const shipping = Math.min(basePrice * 0.1, 100);
  return basePrice - quantityDiscount + shipping;
}
```

## Example: With a Class

The same code in the context of a class:

```js
class Order {
  constructor(aRecord) {
    this._data = aRecord;
  }

  get quantity()  {return this._data.quantity;}
  get itemPrice() {return this._data.itemPrice;}

  get price() {
    return this.quantity * this.itemPrice -
      Math.max(0, this.quantity - 500) * this.itemPrice * 0.05 +
      Math.min(this.quantity * this.itemPrice * 0.1, 100);
  }
}
```

Here the names apply to the `Order` as a whole, not just the price calculation. So extract the names as methods rather than variables:

```js
class Order {
  constructor(aRecord) {
    this._data = aRecord;
  }
  get quantity()  {return this._data.quantity;}
  get itemPrice() {return this._data.itemPrice;}

  get price() {
    return this.basePrice - this.quantityDiscount + this.shipping;
  }
  get basePrice()        {return this.quantity * this.itemPrice;}
  get quantityDiscount() {return Math.max(0, this.quantity - 500) * this.itemPrice * 0.05;}
  get shipping()         {return Math.min(this.basePrice * 0.1, 100);}
}
```

Objects give you a reasonable amount of context for logic to share other bits of logic and data. With a larger class it becomes very useful to call out common hunks of behavior as their own abstractions with their own names.
