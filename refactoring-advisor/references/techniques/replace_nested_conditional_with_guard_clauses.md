# Replace Nested Conditional with Guard Clauses

```
function getPayAmount() {
  let result;
  if (isDead)
    result = deadAmount();
  else {
    if (isSeparated)
      result = separatedAmount();
    else {
      if (isRetired)
        result = retiredAmount();
      else
        result = normalPayAmount();
    }
  }
  return result;
}
```

```
function getPayAmount() {
  if (isDead) return deadAmount();
  if (isSeparated) return separatedAmount();
  if (isRetired) return retiredAmount();
  return normalPayAmount();
}
```

## Motivation

Replace nested if-then-else structures with guard clauses that handle special cases and return early, leaving the main path unnested.

## Mechanics

- Select outermost condition that needs to be replaced, and change it into a guard clause.
- Test.
- Repeat as needed.
- If all the guard clauses return the same result, use Consolidate Conditional Expression.

## Example

Here's some code to calculate a payment amount for an employee. It's only relevant if the employee is still with the company, so it has to check for the two other cases.

```js
function payAmount(employee) {
  let result;
  if(employee.isSeparated) {
    result = {amount: 0, reasonCode: "SEP"};
  }
  else {
    if (employee.isRetired) {
      result = {amount: 0, reasonCode: "RET"};
    }
    else {
      // logic to compute amount
      lorem.ipsum(dolor.sitAmet);
      consectetur(adipiscing).elit();
      sed.do.eiusmod = tempor.incididunt.ut(labore) && dolore(magna.aliqua);
      ut.enim.ad(minim.veniam);
      result = someFinalComputation();
    }
  }
  return result;
}
```

The primary purpose of this code only applies if the special conditions aren't the case. Guard clauses make this intention clearer.

I begin with the topmost condition.

```js
function payAmount(employee) {
  let result;
  if (employee.isSeparated) return {amount: 0, reasonCode: "SEP"};
  if (employee.isRetired) {
    result = {amount: 0, reasonCode: "RET"};
  }
  else {
    // logic to compute amount
    lorem.ipsum(dolor.sitAmet);
    consectetur(adipiscing).elit();
    sed.do.eiusmod = tempor.incididunt.ut(labore) && dolore(magna.aliqua);
    ut.enim.ad(minim.veniam);
    result = someFinalComputation();
  }
  return result;
}
```

I test that change and move on to the next one.

```js
function payAmount(employee) {
  let result;
  if (employee.isSeparated) return {amount: 0, reasonCode: "SEP"};
  if (employee.isRetired)   return {amount: 0, reasonCode: "RET"};
  // logic to compute amount
  lorem.ipsum(dolor.sitAmet);
  consectetur(adipiscing).elit();
  sed.do.eiusmod = tempor.incididunt.ut(labore) && dolore(magna.aliqua);
  ut.enim.ad(minim.veniam);
  result = someFinalComputation();
  return result;
}
```

At which point the result variable isn't really doing anything useful, so I remove it.

```js
function payAmount(employee) {
  if (employee.isSeparated) return {amount: 0, reasonCode: "SEP"};
  if (employee.isRetired)   return {amount: 0, reasonCode: "RET"};
  // logic to compute amount
  lorem.ipsum(dolor.sitAmet);
  consectetur(adipiscing).elit();
  sed.do.eiusmod = tempor.incididunt.ut(labore) && dolore(magna.aliqua);
  ut.enim.ad(minim.veniam);
  return someFinalComputation();
}
```

## Example: Reversing the Conditions

We often do Replace Nested Conditional with Guard Clauses by reversing the conditional expressions.

```js
function adjustedCapital(anInstrument) {
  let result = 0;
  if (anInstrument.capital > 0) {
    if (anInstrument.interestRate > 0 && anInstrument.duration > 0) {
      result = (anInstrument.income / anInstrument.duration) * anInstrument.adjustmentFactor;
    }
  }
  return result;
}
```

I make the replacements one at a time, reversing the condition as I put in the guard clause.

```js
function adjustedCapital(anInstrument) {
  let result = 0;
  if (anInstrument.capital <= 0) return result;
  if (anInstrument.interestRate > 0 && anInstrument.duration > 0) {
    result = (anInstrument.income / anInstrument.duration) * anInstrument.adjustmentFactor;
  }
  return result;
}
```

The next conditional is a bit more complicated, so I do it in two steps. First, I simply add a `not`.

```js
function adjustedCapital(anInstrument) {
  let result = 0;
  if (anInstrument.capital <= 0) return result;
  if (!(anInstrument.interestRate > 0 && anInstrument.duration > 0)) return result;
  result = (anInstrument.income / anInstrument.duration) * anInstrument.adjustmentFactor;
  return result;
}
```

Leaving `not`s in a conditional like that twists my mind around at a painful angle, so I simplify it:

```js
function adjustedCapital(anInstrument) {
  let result = 0;
  if (anInstrument.capital <= 0) return result;
  if (anInstrument.interestRate <= 0 || anInstrument.duration <= 0) return result;
  result = (anInstrument.income / anInstrument.duration) * anInstrument.adjustmentFactor;
  return result;
}
```

Both of those lines have conditions with the same result, so I apply Consolidate Conditional Expression.

```js
function adjustedCapital(anInstrument) {
  let result = 0;
  if (   anInstrument.capital      <= 0
      || anInstrument.interestRate <= 0
      || anInstrument.duration     <= 0) return result;
  result = (anInstrument.income / anInstrument.duration) * anInstrument.adjustmentFactor;
  return result;
}
```

The `result` variable is doing two things here. Its first setting to zero indicates what to return when the guard clause triggers; its second value is the final computation. I can get rid of it, which eliminates its double usage.

```js
function adjustedCapital(anInstrument) {
  if (   anInstrument.capital      <= 0
      || anInstrument.interestRate <= 0
      || anInstrument.duration     <= 0) return 0;
  return (anInstrument.income / anInstrument.duration) * anInstrument.adjustmentFactor;
}
```
