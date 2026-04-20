# Parameterize Function

## Motivation

Replaces two or more functions that carry out very similar logic with different literal values by using a single function with parameters for the varying values.

## Mechanics

- Select one of the similar methods.

- Use Change Function Declaration to add any literals that need to turn into parameters.

- For each caller of the function, add the literal value.

- Test.

- Change the body of the function to use the new parameters. Test after each change.

- For each similar function, replace the call with a call to the parameterized function. Test after each one.

  If the original parameterized function doesn't work for a similar function, adjust it for the new function before moving on to the next.

## Example

An obvious example is something like this:

```javascript
function tenPercentRaise(aPerson) {
  aPerson.salary = aPerson.salary.multiply(1.1);
}
function fivePercentRaise(aPerson) {
  aPerson.salary = aPerson.salary.multiply(1.05);
}
```

Hopefully it's obvious that I can replace these with

```javascript
function raise(aPerson, factor) {
  aPerson.salary = aPerson.salary.multiply(1 + factor);
}
```

But it can be a bit more involved than that. Consider this code:

```javascript
function baseCharge(usage) {
  if (usage < 0) return usd(0);
  const amount =
        bottomBand(usage) * 0.03
        + middleBand(usage) * 0.05
        + topBand(usage) * 0.07;
  return usd(amount);
}

function bottomBand(usage) {
  return Math.min(usage, 100);
}

function middleBand(usage) {
  return usage > 100 ? Math.min(usage, 200) - 100 : 0;
}

function topBand(usage) {
  return usage > 200 ? usage - 200 : 0;
}
```

The logic is clearly similar -- but is it similar enough to support creating a parameterized method for the bands? It is, but may be less obvious than the trivial case above.

When looking to parameterize related functions, my approach is to take one of the functions and add parameters to it, with an eye to the other cases. With range-oriented things like this, usually the place to start is with the middle range. So I'll work on `middleBand` to change it to use parameters, and then adjust other callers to fit.

`middleBand` uses two literal values: `100` and `200`. These represent the bottom and top of this middle band. I begin by using Change Function Declaration to add them to the call. While I'm at it, I'll also change the name of the function to something that makes sense with the parameterization.

```javascript
function withinBand(usage, bottom, top) {
  return usage > 100 ? Math.min(usage, 200) - 100 : 0;
}

function baseCharge(usage) {
  if (usage < 0) return usd(0);
  const amount =
        bottomBand(usage) * 0.03
        + withinBand(usage, 100, 200) * 0.05
        + topBand(usage) * 0.07;
  return usd(amount);
}
```

I replace each literal with a reference to the parameter:

```javascript
function withinBand(usage, bottom, top) {
  return usage > bottom ? Math.min(usage, 200) - bottom : 0;
}
```

then:

```javascript
function withinBand(usage, bottom, top) {
  return usage > bottom ? Math.min(usage, top) - bottom : 0;
}
```

I replace the call to the bottom band with a call to the newly parameterized function.

```javascript
function baseCharge(usage) {
  if (usage < 0) return usd(0);
  const amount =
        withinBand(usage, 0, 100) * 0.03
        + withinBand(usage, 100, 200) * 0.05
        + topBand(usage) * 0.07;
  return usd(amount);
}
```

To replace the call to the top band, I need to make use of infinity.

```javascript
function baseCharge(usage) {
  if (usage < 0) return usd(0);
  const amount =
        withinBand(usage, 0, 100) * 0.03
        + withinBand(usage, 100, 200) * 0.05
        + withinBand(usage, 200, Infinity) * 0.07;
  return usd(amount);
}
```

With the logic working the way it does now, I could remove the initial guard clause. But although it's logically unnecessary now, I like to keep it as it documents how to handle that case.
