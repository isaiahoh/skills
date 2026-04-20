# Inline Function

```
function getRating(driver) {
  return moreThanFiveLateDeliveries(driver) ? 2 : 1;
}

function moreThanFiveLateDeliveries(driver) {
  return driver.numberOfLateDeliveries > 5;
}
```

```
function getRating(driver) {
  return (driver.numberOfLateDeliveries > 5) ? 2 : 1;
}
```

## Motivation

Replaces a function call with the function's body when the body is as clear as the name, eliminating needless indirection.

## Mechanics

- Check that this isn't a polymorphic method.

  If this is a method in a class, and has subclasses that override it, don't inline it.

- Find all the callers of the function.

- Replace each call with the function's body.

- Test after each replacement.

  The entire inlining doesn't have to be done all at once. If some parts are tricky, do them gradually as opportunity permits.

- Remove the function definition.

  If you encounter complexities (recursion, multiple return points, inlining into another object without accessors), don't do this refactoring.

## Example

In the simplest case, this refactoring is trivial.

```js
function rating(aDriver) {
  return moreThanFiveLateDeliveries(aDriver) ? 2 : 1;
}
function moreThanFiveLateDeliveries(aDriver) {
  return aDriver.numberOfLateDeliveries > 5;
}
```

Take the return expression of the called function and paste it into the caller to replace the call.

```js
function rating(aDriver) {
  return aDriver.numberOfLateDeliveries > 5 ? 2 : 1;
}
```

It can be a bit more involved when the declared argument name differs from the passed-in argument:

```js
function rating(aDriver) {
  return moreThanFiveLateDeliveries(aDriver) ? 2 : 1;
}

function moreThanFiveLateDeliveries(dvr) {
  return dvr.numberOfLateDeliveries > 5;
}
```

You have to fit the code when you inline -- substituting `aDriver` for `dvr`:

```js
function rating(aDriver) {
  return aDriver.numberOfLateDeliveries > 5 ? 2 : 1;
}
```

It can be even more involved with multi-statement bodies:

```js
function reportLines(aCustomer) {
  const lines = [];
  gatherCustomerData(lines, aCustomer);
  return lines;
}
function gatherCustomerData(out, aCustomer) {
  out.push(["name", aCustomer.name]);
  out.push(["location", aCustomer.location]);
}
```

Inlining `gatherCustomerData` into `reportLines` isn't a simple cut and paste. To be cautious, move one line at a time using Move Statements to Callers on the first line (cut, paste, and fit).

```js
function reportLines(aCustomer) {
  const lines = [];
  lines.push(["name", aCustomer.name]);
  gatherCustomerData(lines, aCustomer);
  return lines;
}
function gatherCustomerData(out, aCustomer) {
  out.push(["name", aCustomer.name]);
  out.push(["location", aCustomer.location]);
}
```

Then continue with the other lines until done.

```js
function reportLines(aCustomer) {
  const lines = [];
  lines.push(["name", aCustomer.name]);
  lines.push(["location", aCustomer.location]);
  return lines;
}
```

Always be ready to take smaller steps. If you run into complications, go one line at a time. If something breaks, revert to the last green code and repeat with smaller steps.
