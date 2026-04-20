# Replace Parameter with Query

## Motivation

Removes a parameter that the function can derive on its own, reducing duplication between caller and callee. Do not apply if it would force the function body to access a mutable global or an element it should remain ignorant of, as that would add an unwanted dependency or break referential transparency.

## Mechanics

- If necessary, use Extract Function on the calculation of the parameter.

- Replace references to the parameter in the function body with references to the expression that yields the parameter. Test after each change.

- Use Change Function Declaration to remove the parameter.

## Example

I most often use Replace Parameter with Query when I've done some other refactorings that make a parameter no longer needed. Consider this code.

class Order...

```javascript
get finalPrice() {
  const basePrice = this.quantity * this.itemPrice;
  let discountLevel;
  if (this.quantity > 100) discountLevel = 2;
  else discountLevel = 1;
  return this.discountedPrice(basePrice, discountLevel);
}

discountedPrice(basePrice, discountLevel) {
  switch (discountLevel) {
    case 1: return basePrice * 0.95;
    case 2: return basePrice * 0.9;
  }
}
```

When simplifying a function, I'm keen to apply Replace Temp with Query, which would lead me to

class Order...

```javascript
get finalPrice() {
  const basePrice = this.quantity * this.itemPrice;
  return this.discountedPrice(basePrice, this.discountLevel);
}

get discountLevel() {
  return (this.quantity > 100) ? 2 : 1;
}
```

Once I've done this, there's no need to pass the result of `discountLevel` to `discountedPrice` -- it can just as easily make the call itself.

I replace any reference to the parameter with a call to the method instead.

class Order...

```javascript
discountedPrice(basePrice, discountLevel) {
  switch (this.discountLevel) {
    case 1: return basePrice * 0.95;
    case 2: return basePrice * 0.9;
  }
}
```

I can then use Change Function Declaration to remove the parameter.

class Order...

```javascript
get finalPrice() {
  const basePrice = this.quantity * this.itemPrice;
  return this.discountedPrice(basePrice);
}

discountedPrice(basePrice) {
  switch (this.discountLevel) {
    case 1: return basePrice * 0.95;
    case 2: return basePrice * 0.9;
  }
}
```
