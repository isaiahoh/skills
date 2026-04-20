# Move Field

```
class Customer {
  get plan() {return this._plan;}
  get discountRate() {return this._discountRate;}
}
```

```
class Customer {
  get plan() {return this._plan;}
  get discountRate() {return this.plan.discountRate;}
}
```

## Motivation

Move a field to the record or class where it logically belongs, so that data that changes together lives together and only needs to be updated in one place. With bare records that lack encapsulation, refactor usage patterns to use accessors before attempting the move.

## Mechanics

- Ensure the source field is encapsulated.

- Test.

- Create a field (and accessors) in the target.

- Run static checks.

- Ensure there is a reference from the source object to the target object.

  An existing field or method may give you the target. If not, see if you can easily create a method that will do so. Failing that, you may need to create a new field in the source object that can store the target. This may be a permanent change, but you can also do it temporarily until you have done enough refactoring in the broader context.

- Adjust accessors to use the target field.

  If the target is shared between source objects, consider first updating the setter to modify both target and source fields, followed by Introduce Assertion to detect inconsistent updates. Once you determine all is well, finish changing the accessors to use the target field.

- Test.

- Remove the source field.

- Test.

## Example

I'm starting here with this customer and contract:

class Customer...

```js
constructor(name, discountRate) {
  this._name = name;
  this._discountRate = discountRate;
  this._contract = new CustomerContract(dateToday());
}
get discountRate() {return this._discountRate;}
becomePreferred() {
  this._discountRate += 0.03;
  // other nice things
}
applyDiscount(amount) {
  return amount.subtract(amount.multiply(this._discountRate));
}
```

class CustomerContract...

```js
constructor(startDate) {
  this._startDate = startDate;
}
```

I want to move the discount rate field from the customer to the customer contract.

The first thing I need to do is use Encapsulate Variable to encapsulate access to the discount rate field.

class Customer...

```js
constructor(name, discountRate) {
  this._name = name;
  this._setDiscountRate(discountRate);
  this._contract = new CustomerContract(dateToday());
}
get discountRate() {return this._discountRate;}
_setDiscountRate(aNumber) {this._discountRate = aNumber;}
becomePreferred() {
  this._setDiscountRate(this.discountRate + 0.03);
  // other nice things
}
applyDiscount(amount) {
  return amount.subtract(amount.multiply(this.discountRate));
}
```

I use a method to update the discount rate, rather than a property setter, as I don't want to make a public setter for the discount rate.

I add a field and accessors to the customer contract.

class CustomerContract...

```js
constructor(startDate, discountRate) {
  this._startDate = startDate;
  this._discountRate = discountRate;
}
get discountRate()    {return this._discountRate;}
set discountRate(arg) {this._discountRate = arg;}
```

I now modify the accessors on customer to use the new field. When I did that, I got an error: "Cannot set property 'discountRate' of undefined". This was because `_setDiscountRate` was called before I created the contract object in the constructor. To fix that, I first reverted to the previous state, then used Slide Statements to move the `_setDiscountRate` after creating the contract.

class Customer...

```js
constructor(name, discountRate) {
  this._name = name;
  this._contract = new CustomerContract(dateToday());
  this._setDiscountRate(discountRate);
}
```

I tested that, then changed the accessors again to use the contract.

class Customer...

```js
get discountRate() {return this._contract.discountRate;}
_setDiscountRate(aNumber) {this._contract.discountRate = aNumber;}
```

Since I'm using JavaScript, there is no declared source field, so I don't need to remove anything further.

### Changing a Bare Record

This refactoring is generally easier with objects, since encapsulation provides a natural way to wrap data access in methods. If I have many functions accessing a bare record, then, while it's still a valuable refactoring, it is decidedly more tricky.

I can create accessor functions and modify all the reads and writes to use them. If the field that's being moved is immutable, I can update both the source and the target fields when I set its value and gradually migrate reads. Still, if possible, my first move would be to use Encapsulate Record to turn the record into a class so I can make the change more easily.

## Example: Moving to a Shared Object

Now, let's consider a different case. Here's an account with an interest rate:

class Account...

```js
constructor(number, type, interestRate) {
  this._number = number;
  this._type = type;
  this._interestRate = interestRate;
}
get interestRate() {return this._interestRate;}
```

class AccountType...

```js
constructor(nameString) {
  this._name = nameString;
}
```

I want to change things so that an account's interest rate is determined from its account type.

The access to the interest rate is already nicely encapsulated, so I'll just create the field and an appropriate accessor on the account type.

class AccountType...

```js
constructor(nameString, interestRate) {
  this._name = nameString;
  this._interestRate = interestRate;
}
get interestRate() {return this._interestRate;}
```

But there is a potential problem when I update the accesses from account. Before this refactoring, each account had its own interest rate. Now, I want all accounts to share the interest rates of their account type. If all the accounts of the same type already have the same interest rate, then there's no change in observable behavior, so I'm fine with the refactoring. But if there's an account with a different interest rate, it's no longer a refactoring. If my account data is held in a database, I should check the database to ensure that all accounts have the rate matching their type. I can also Introduce Assertion in the account class.

class Account...

```js
constructor(number, type, interestRate) {
  this._number = number;
  this._type = type;
  assert(interestRate === this._type.interestRate);
  this._interestRate = interestRate;
}
get interestRate() {return this._interestRate;}
```

I might run the system for a while with this assertion in place to see if I get an error. Or, instead of adding an assertion, I might log the problem. Once I'm confident that I'm not introducing an observable change, I can change the access, removing the update from the account completely.

class Account...

```js
constructor(number, type) {
  this._number = number;
  this._type = type;
}
get interestRate() {return this._type.interestRate;}
```
