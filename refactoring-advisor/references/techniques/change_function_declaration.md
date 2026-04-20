# Change Function Declaration

```
function circum(radius) {...}
```

```
function circumference(radius) {...}
```

Also known as: Rename Function, Add Parameter, Remove Parameter, Change Signature

## Motivation

Changes a function's name or parameter list to better communicate its purpose or adjust how it couples to the rest of the system.

## Mechanics

The simple mechanics work when you can change the declaration and all callers in one go. The migration mechanics allow changing callers more gradually -- use them when there are many callers, they are awkward to get to, the function is polymorphic, or the change is complicated.

### Simple Mechanics

- If you're removing a parameter, ensure it isn't referenced in the body of the function.

- Change the method declaration to the desired declaration.

- Find all references to the old method declaration, update them to the new one.

- Test.

  It's often best to separate changes: if you want to both change the name and add a parameter, do these as separate steps. If you run into trouble, revert and use the migration mechanics instead.

### Migration Mechanics

- If necessary, refactor the body of the function to make it easy to do the following extraction step.

- Use Extract Function on the function body to create the new function.

  If the new function will have the same name as the old one, give the new function a temporary name that's easy to search for.

- If the extracted function needs additional parameters, use the simple mechanics to add them.

- Test.

- Apply Inline Function to the old function.

- If you used a temporary name, use Change Function Declaration again to restore it to the original name.

- Test.

- If you're changing a method on a class with polymorphism, you'll need to add indirection for each binding. If the method is polymorphic within a single class hierarchy, you only need the forwarding method on the superclass; otherwise, you need forwarding methods on each implementation class.

- If you are refactoring a published API, pause the refactoring after creating the new function. Deprecate the original function and wait for clients to migrate. Remove the original when all clients have switched.

## Example: Renaming a Function (Simple Mechanics)

A function with an overly abbreviated name:

```js
function circum(radius) {
  return 2 * Math.PI * radius;
}
```

Change the declaration:

```js
function circumference(radius) {
  return 2 * Math.PI * radius;
}
```

Then find all callers of `circum` and change the name to `circumference`.

Static typing and a good IDE usually allow renaming functions automatically. Without static typing, searching can produce false positives.

The same approach works for adding or removing parameters. It's often better to do name changes and parameter changes as separate steps.

If there are too many callers, or the names aren't unique across classes, use the migration mechanics instead.

## Example: Renaming a Function (Migration Mechanics)

The same abbreviated function:

```js
function circum(radius) {
  return 2 * Math.PI * radius;
}
```

Apply Extract Function to the entire function body:

```js
function circum(radius) {
  return circumference(radius);
}
function circumference(radius) {
  return 2 * Math.PI * radius;
}
```

Test, then apply Inline Function to the old function. Find all calls of the old function and replace each one with a call of the new one. Test after each change. Once all are replaced, remove the old function.

With a published API, you can pause after creating `circumference` and deprecate `circum`. Wait for callers to switch, then delete `circum`. At minimum, new code gets a better name.

## Example: Adding a Parameter

A book class with the ability to take a reservation:

class Book...

```js
addReservation(customer) {
  this._reservations.push(customer);
}
```

A priority queue is needed, requiring an extra parameter. Using migration mechanics, first extract the body with a temporary name:

class Book...

```js
addReservation(customer) {
  this.zz_addReservation(customer);
}

zz_addReservation(customer) {
  this._reservations.push(customer);
}
```

Add the parameter to the new declaration and its call:

class Book...

```js
addReservation(customer) {
  this.zz_addReservation(customer, false);
}

zz_addReservation(customer, isPriority) {
  this._reservations.push(customer);
}
```

In JavaScript, add an assertion to check the new parameter is used by the caller -- this catches mistakes when changing callers:

class Book...

```js
zz_addReservation(customer, isPriority) {
  assert(isPriority === true || isPriority === false);
  this._reservations.push(customer);
}
```

Now change callers one at a time using Inline Function on the original. Then rename the new function back to the original.

## Example: Changing a Parameter to One of Its Properties

A function that determines if a customer is based in New England:

```js
function inNewEngland(aCustomer) {
  return ["MA", "CT", "ME", "VT", "NH", "RI"].includes(aCustomer.address.state);
}
```

One of its callers:

caller...

```js
const newEnglanders = someCustomers.filter(c => inNewEngland(c));
```

`inNewEngland` only uses the customer's home state. Refactor it to take a state code as a parameter, making it usable in more contexts by removing the dependency on the customer.

First, use Extract Variable on the desired new parameter:

```js
function inNewEngland(aCustomer) {
  const stateCode = aCustomer.address.state;
  return ["MA", "CT", "ME", "VT", "NH", "RI"].includes(stateCode);
}
```

Use Extract Function to create the new function with a temporary name:

```js
function inNewEngland(aCustomer) {
  const stateCode = aCustomer.address.state;
  return xxNEWinNewEngland(stateCode);
}

function xxNEWinNewEngland(stateCode) {
  return ["MA", "CT", "ME", "VT", "NH", "RI"].includes(stateCode);
}
```

Apply Inline Variable on the input parameter in the original function:

```js
function inNewEngland(aCustomer) {
  return xxNEWinNewEngland(aCustomer.address.state);
}
```

Use Inline Function to fold the old function into its callers one at a time:

caller...

```js
const newEnglanders = someCustomers.filter(c => xxNEWinNewEngland(c.address.state));
```

Once all callers are inlined, rename the new function back to the original:

caller...

```js
const newEnglanders = someCustomers.filter(c => inNewEngland(c.address.state));
```

top level...

```js
function inNewEngland(stateCode) {
  return ["MA", "CT", "ME", "VT", "NH", "RI"].includes(stateCode);
}
```
