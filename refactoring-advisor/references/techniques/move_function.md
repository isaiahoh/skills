# Move Function

```
class Account {
  get overdraftCharge() {...}

class AccountType {
    get overdraftCharge() {...}
```

## Motivation

Move a function to the context (class, module, or scope) whose data and behavior it references most, reducing coupling and improving encapsulation.

## Mechanics

- Examine all the program elements used by the chosen function in its current context. Consider whether they should move too.

  If a called function should also move, move it first. That way, moving a cluster of functions begins with the one that has the least dependency on the others in the group.

  If a high-level function is the only caller of subfunctions, inline those subfunctions into the high-level method, move, and re-extract at the destination.

- Check if the chosen function is a polymorphic method.

  In an object-oriented language, take account of super- and subclass declarations.

- Copy the function to the target context. Adjust it to fit in its new home.

  If the body uses elements in the source context, either pass those elements as parameters or pass a reference to the source context.

  Moving a function often means coming up with a different name that works better in the new context.

- Perform static analysis.

- Figure out how to reference the target function from the source context.

- Turn the source function into a delegating function.

- Test.

- Consider Inline Function on the source function.

  The source function can stay indefinitely as a delegating function. But if its callers can just as easily reach the target directly, it is better to remove the middle man.

## Example: Moving a Nested Function to Top Level

A function calculates the total distance for a GPS track record.

```js
function trackSummary(points) {
  const totalTime = calculateTime();
  const totalDistance = calculateDistance();
  const pace = totalTime / 60 / totalDistance;
  return {
    time: totalTime,
    distance: totalDistance,
    pace: pace
  };

  function calculateDistance() {
    let result = 0;
    for (let i = 1; i < points.length; i++) {
      result += distance(points[i-1], points[i]);
    }
    return result;
  }

  function distance(p1,p2) { ... }
  function radians(degrees) { ... }
  function calculateTime() { ... }
}
```

The goal is to move `calculateDistance` to the top level so distances can be calculated for tracks without the rest of the summary.

**Copy the function to the top level with a temporary name** so the original and copy are easy to distinguish during the move.

```js
function trackSummary(points) {
  const totalTime = calculateTime();
  const totalDistance = calculateDistance();
  const pace = totalTime / 60 / totalDistance;
  return {
    time: totalTime,
    distance: totalDistance,
    pace: pace
  };

  function calculateDistance() {
    let result = 0;
    for (let i = 1; i < points.length; i++) {
      result += distance(points[i-1], points[i]);
    }
    return result;
  }
  ...

  function distance(p1,p2) { ... }
  function radians(degrees) { ... }
  function calculateTime() { ... }
}

function top_calculateDistance() {
  let result = 0;
  for (let i = 1; i < points.length; i++) {
    result += distance(points[i-1], points[i]);
  }
  return result;
}
```

Static analysis flags two undefined symbols in the new function: `distance` and `points`. The natural fix for `points` is to pass it as a parameter.

```js
function top_calculateDistance(points) {
  let result = 0;
  for (let i = 1; i < points.length; i++) {
    result += distance(points[i-1], points[i]);
  }
  return result;
}
```

`distance` could also be passed as a parameter, but it makes sense to move it along with `calculateDistance`. Here is the relevant code:

function trackSummary...

```js
function distance(p1,p2) {
  // haversine formula see http://www.movable-type.co.uk/scripts/latlong.html
  const EARTH_RADIUS = 3959; // in miles
  const dLat = radians(p2.lat) - radians(p1.lat);
  const dLon = radians(p2.lon) - radians(p1.lon);
  const a = Math.pow(Math.sin(dLat / 2),2)
          + Math.cos(radians(p2.lat))
          * Math.cos(radians(p1.lat))
          * Math.pow(Math.sin(dLon / 2), 2);
  const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1-a));
  return EARTH_RADIUS * c;
}
function radians(degrees) {
  return degrees * Math.PI / 180;
}
```

`distance` only uses `radians`, and `radians` uses nothing from its current context. Rather than passing the functions as parameters, move them too. **First nest them inside `calculateDistance`** so static analysis and tests can catch any complications.

```js
function trackSummary(points) {
  const totalTime = calculateTime();
  const totalDistance = calculateDistance();
  const pace = totalTime / 60 / totalDistance;
  return {
    time: totalTime,
    distance: totalDistance,
    pace: pace
  };

  function calculateDistance() {
    let result = 0;
    for (let i = 1; i < points.length; i++) {
      result += distance(points[i-1], points[i]);
    }
    return result;

    function distance(p1,p2) { ... }
    function radians(degrees) { ... }
  }
```

All is well, so **copy them over to `top_calculateDistance`** as well. The copy does not change runtime behavior but gives another opportunity for static analysis -- had `distance`'s call to `radians` been missed, the linter would catch it here.

```js
function top_calculateDistance(points) {
  let result = 0;
  for (let i = 1; i < points.length; i++) {
    result += distance(points[i-1], points[i]);
  }
  return result;

  function distance(p1,p2) { ... }
  function radians(degrees) { ... }
}
```

Now **turn the original `calculateDistance` into a delegating function** that calls the top-level copy. This is the crucial time to run tests.

```js
function trackSummary(points) {
  const totalTime = calculateTime();
  const totalDistance = calculateDistance();
  const pace = totalTime / 60 / totalDistance;
  return {
    time: totalTime,
    distance: totalDistance,
    pace: pace
  };

  function calculateDistance() {
    return top_calculateDistance(points);
  }
```

With tests passing, decide whether to keep the delegating function. Here the callers are few and localized, so **inline the delegation** -- replace the call to `calculateDistance` with a direct call to the top-level function.

```js
function trackSummary(points) {
  const totalTime = calculateTime();
  const totalDistance = top_calculateDistance(points);
  const pace = totalTime / 60 / totalDistance;
  return {
    time: totalTime,
    distance: totalDistance,
    pace: pace
  };
```

Now **choose a good final name**. Since the top-level function has the highest visibility, it deserves the best name. `totalDistance` is a good choice, but the local variable inside `trackSummary` shadows it. There is no reason to keep that variable, so use Inline Variable to remove it, then rename the function.

```js
function trackSummary(points) {
  const totalTime = calculateTime();
  const pace = totalTime / 60 / totalDistance(points);
  return {
    time: totalTime,
    distance: totalDistance(points),
    pace: pace
  };

function totalDistance(points) {
  let result = 0;
  for (let i = 1; i < points.length; i++) {
    result += distance(points[i-1], points[i]);
  }
  return result;
```

If the variable needed to stay, it could be renamed to something like `totalDistanceCache`.

Since `distance` and `radians` do not depend on anything inside `totalDistance`, move them to top level too, leaving all four functions flat.

```js
function trackSummary(points) { ... }
function totalDistance(points) { ... }
function distance(p1,p2) { ... }
function radians(degrees) { ... }
```

## Example: Moving between Classes

class Account...

```js
get bankCharge() {
  let result = 4.5;
  if (this._daysOverdrawn > 0) result += this.overdraftCharge;
  return result;
}

get overdraftCharge() {
  if (this.type.isPremium) {
    const baseCharge = 10;
    if (this.daysOverdrawn <= 7)
      return baseCharge;
    else
      return baseCharge + (this.daysOverdrawn - 7) * 0.85;
  }
  else
    return this.daysOverdrawn * 1.75;
}
```

Upcoming changes will give different account types different overdraft-charge algorithms, so it is natural to move `overdraftCharge` to the account type class.

**Examine what the method uses.** `daysOverdrawn` must remain on Account because it varies per individual account. `overdraftCharge` is the method to move.

**Copy the method body to AccountType and adjust it to fit.** `isPremium` becomes a simple call on `this`. For `daysOverdrawn`, decide whether to pass the value or the whole account. Pass the simple value for now; this can change later if more account data is needed.

class AccountType...

```js
overdraftCharge(daysOverdrawn) {
  if (this.isPremium) {
    const baseCharge = 10;
    if (daysOverdrawn <= 7)
      return baseCharge;
    else
      return baseCharge + (daysOverdrawn - 7) * 0.85;
  }
  else
    return daysOverdrawn * 1.75;
}
```

**Replace the original method body with a delegating call.**

class Account...

```js
get bankCharge() {
  let result = 4.5;
  if (this._daysOverdrawn > 0) result += this.overdraftCharge;
  return result;
}

get overdraftCharge() {
  return this.type.overdraftCharge(this.daysOverdrawn);
}
```

**Decide whether to keep the delegation or inline it.** Inlining results in:

class Account...

```js
get bankCharge() {
  let result = 4.5;
  if (this._daysOverdrawn > 0)
    result += this.type.overdraftCharge(this.daysOverdrawn);
  return result;
}
```

**Alternative: pass the whole account** when there is a lot of data to forward, or when the data needed varies by account type.

class Account...

```js
get bankCharge() {
  let result = 4.5;
  if (this._daysOverdrawn > 0) result += this.overdraftCharge;
  return result;
}

get overdraftCharge() {
  return this.type.overdraftCharge(this);
}
```

class AccountType...

```js
overdraftCharge(account) {
  if (this.isPremium) {
    const baseCharge = 10;
    if (account.daysOverdrawn <= 7)
      return baseCharge;
    else
      return baseCharge + (account.daysOverdrawn - 7) * 0.85;
  }
  else
    return account.daysOverdrawn * 1.75;
}
```
