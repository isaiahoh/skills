# Consolidate Conditional Expression

```
if (anEmployee.seniority < 2) return 0;
if (anEmployee.monthsDisabled > 12) return 0;
if (anEmployee.isPartTime) return 0;
```

```
if (isNotEligableForDisability()) return 0;

function isNotEligableForDisability() {
  return ((anEmployee.seniority < 2)
          || (anEmployee.monthsDisabled > 12)
          || (anEmployee.isPartTime));
}
```

## Motivation

Combine a series of conditional checks that all produce the same result into a single expression using `and`/`or` operators, then extract it into a named function.

## Mechanics

- Ensure that none of the conditionals have any side effects. If any do, use Separate Query from Modifier on them first.
- Take two of the conditional statements and combine their conditions using a logical operator. Sequences combine with `or`, nested `if` statements combine with `and`.
- Test.
- Repeat combining conditionals until they are all in a single condition.
- Consider using Extract Function on the resulting condition.

## Example

Perusing some code, I see the following:

```js
function disabilityAmount(anEmployee) {
  if (anEmployee.seniority < 2) return 0;
  if (anEmployee.monthsDisabled > 12) return 0;
  if (anEmployee.isPartTime) return 0;
  // compute the disability amount
```

It's a sequence of conditional checks which all have the same result. Since the result is the same, I should combine these conditions into a single expression. For a sequence like this, I do it using an `or` operator.

```js
function disabilityAmount(anEmployee) {
  if ((anEmployee.seniority < 2)
      || (anEmployee.monthsDisabled > 12)) return 0;
  if (anEmployee.isPartTime) return 0;
  // compute the disability amount
```

I test, then fold in the other condition:

```js
function disabilityAmount(anEmployee) {
  if ((anEmployee.seniority < 2)
      || (anEmployee.monthsDisabled > 12)
      || (anEmployee.isPartTime)) return 0;
  // compute the disability amount
```

Once I have them all together, I use Extract Function on the condition.

```js
function disabilityAmount(anEmployee) {
  if (isNotEligableForDisability()) return 0;
  // compute the disability amount

  function isNotEligableForDisability() {
    return ((anEmployee.seniority < 2)
            || (anEmployee.monthsDisabled > 12)
            || (anEmployee.isPartTime));
  }
```

## Example: Using ands

The example above showed combining statements with an `or`, but I may run into cases that need `and`s as well. Such a case uses nested `if` statements:

```js
if (anEmployee.onVacation)
  if (anEmployee.seniority > 10)
    return 1;
return 0.5;
```

I combine these using `and` operators.

```js
if ((anEmployee.onVacation)
    && (anEmployee.seniority > 10)) return 1;
return 0.5;
```

If I have a mix of these, I can combine using `and` and `or` operators as needed. When this happens, things are likely to get messy, so I use Extract Function liberally to make it all understandable.
