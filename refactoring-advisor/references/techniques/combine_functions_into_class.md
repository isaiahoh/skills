# Combine Functions into Class

```
function base(aReading) {...}
function taxableCharge(aReading) {...}
function calculateBaseCharge(aReading) {...}
```

```
class Reading {
  base() {...}
  taxableCharge() {...}
  calculateBaseCharge() {...}
}
```

## Motivation

Groups functions that operate closely on a common body of data into a class, making the shared environment explicit and simplifying function calls. Prefer a class over Combine Functions into Transform when the source data may be mutated, since a class keeps derivations consistent with updated data.

## Mechanics

- Apply Encapsulate Record to the common data record that the functions share.

  If the common data isn't already grouped into a record structure, use Introduce Parameter Object to create one.

- Take each function that uses the common record and use Move Function to move it into the new class.

  Remove arguments to the function call that are now members.

- Each bit of logic that manipulates the data can be extracted with Extract Function and then moved into the new class.

## Example

Monthly tea meter readings produce records like:

```js
reading = {customer: "ivan", quantity: 10, month: 5, year: 2017};
```

Various places calculate things on this data. One calculates the base charge:

client 1...

```js
const aReading = acquireReading();
const baseCharge = baseRate(aReading.month, aReading.year) * aReading.quantity;
```

Another calculates the taxable charge (allowing some essential tea tax-free):

client 2...

```js
const aReading = acquireReading();
const base = (baseRate(aReading.month, aReading.year) * aReading.quantity);
const taxableCharge =  Math.max(0, base - taxThreshold(aReading.year));
```

The base charge formula is duplicated. Elsewhere someone already extracted it:

client 3...

```js
const aReading = acquireReading();
const basicChargeAmount = calculateBaseCharge(aReading);

function calculateBaseCharge(aReading) {
  return  baseRate(aReading.month, aReading.year) * aReading.quantity;
}
```

Top-level functions like this are easy to miss. Give the function a closer connection to its data by turning the data into a class.

Use Encapsulate Record to turn the record into a class:

```js
class Reading {
  constructor(data) {
    this._customer = data.customer;
    this._quantity = data.quantity;
    this._month = data.month;
    this._year = data.year;
  }
  get customer() {return this._customer;}
  get quantity() {return this._quantity;}
  get month()    {return this._month;}
  get year()     {return this._year;}
}
```

Apply the new class to the data as soon as it's acquired:

client 3...

```js
const rawReading = acquireReading();
const aReading = new Reading(rawReading);
const basicChargeAmount = calculateBaseCharge(aReading);
```

Use Move Function to move `calculateBaseCharge` into the new class:

class Reading...

```js
get calculateBaseCharge() {
  return  baseRate(this.month, this.year) * this.quantity;
}
```

client 3...

```js
const rawReading = acquireReading();
const aReading = new Reading(rawReading);
const basicChargeAmount = aReading.calculateBaseCharge;
```

Rename it to something better. The client can't tell whether `baseCharge` is a field or a derived value (Uniform Access Principle):

```js
get baseCharge() {
  return  baseRate(this.month, this.year) * this.quantity;
}
```

client 3...

```js
const rawReading = acquireReading();
const aReading = new Reading(rawReading);
const basicChargeAmount = aReading.baseCharge;
```

Alter the first client to call the method:

client 1...

```js
const rawReading = acquireReading();
const aReading = new Reading(rawReading);
const baseCharge = aReading.baseCharge;
```

Now update the taxable charge client to use the new base charge property:

client 2...

```js
const rawReading = acquireReading();
const aReading = new Reading(rawReading);
const taxableCharge =  Math.max(0, aReading.baseCharge - taxThreshold(aReading.year));
```

Use Extract Function on the taxable charge calculation:

```js
function taxableChargeFn(aReading) {
  return  Math.max(0, aReading.baseCharge - taxThreshold(aReading.year));
}
```

client 3...

```js
const rawReading = acquireReading();
const aReading = new Reading(rawReading);
const taxableCharge = taxableChargeFn(aReading);
```

Then apply Move Function:

class Reading...

```js
get taxableCharge() {
  return  Math.max(0, this.baseCharge - taxThreshold(this.year));
}
```

client 3...

```js
const rawReading = acquireReading();
const aReading = new Reading(rawReading);
const taxableCharge = aReading.taxableCharge;
```

Since all the derived data is calculated on demand, there's no problem with updating the stored data. In general, prefer immutable data, but when there is a reasonable chance the data will be updated, a class is very helpful.
