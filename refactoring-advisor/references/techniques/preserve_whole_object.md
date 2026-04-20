# Preserve Whole Object

## Motivation

Replaces passing several values derived from a record with passing the whole record, letting the called function derive what it needs. Do not apply if the called function should not have a dependency on the whole (e.g., they are in different modules).

## Mechanics

- Create an empty function with the desired parameters.

  Give the function an easily searchable name so it can be replaced at the end.

- Fill the body of the new function with a call to the old function, mapping from the new parameters to the old ones.

- Run static checks.

- Adjust each caller to use the new function, testing after each change.

  This may mean that some code that derives the parameter isn't needed, so can fall to Remove Dead Code.

- Once all original callers have been changed, use Inline Function on the original function.

- Change the name of the new function and all its callers.

## Example

Consider a room monitoring system. It compares its daily temperature range with a range in a predefined heating plan.

caller...

```javascript
const low = aRoom.daysTempRange.low;
const high = aRoom.daysTempRange.high;
if (!aPlan.withinRange(low, high))
  alerts.push("room temperature went outside range");
```

class HeatingPlan...

```javascript
withinRange(bottom, top) {
  return (bottom >= this._temperatureRange.low) && (top <= this._temperatureRange.high);
}
```

Instead of unpacking the range information when I pass it in, I can pass in the whole range object.

I begin by stating the interface I want as an empty function.

class HeatingPlan...

```javascript
xxNEWwithinRange(aNumberRange) {
}
```

Since I intend it to replace the existing `withinRange`, I name it the same but with an easily replaceable prefix.

I then add the body of the function, which relies on calling the existing `withinRange`. The body consists of a mapping from the new parameter to the existing ones.

class HeatingPlan...

```javascript
xxNEWwithinRange(aNumberRange) {
  return this.withinRange(aNumberRange.low, aNumberRange.high);
}
```

Now I can begin the serious work, taking the existing function calls and having them call the new function.

caller...

```javascript
const low = aRoom.daysTempRange.low;
const high = aRoom.daysTempRange.high;
if (!aPlan.xxNEWwithinRange(aRoom.daysTempRange))
  alerts.push("room temperature went outside range");
```

When I've changed the calls, I may see that some of the earlier code isn't needed anymore, so I wield Remove Dead Code.

caller...

```javascript
if (!aPlan.xxNEWwithinRange(aRoom.daysTempRange))
  alerts.push("room temperature went outside range");
```

I replace these one at a time, testing after each change.

Once I've replaced them all, I can use Inline Function on the original function.

class HeatingPlan...

```javascript
xxNEWwithinRange(aNumberRange) {
  return (aNumberRange.low >= this._temperatureRange.low) &&
    (aNumberRange.high <= this._temperatureRange.high);
}
```

And I finally remove that ugly prefix from the new function and all its callers. The prefix makes it a simple global replace, even if I don't have robust rename support in my editor.

class HeatingPlan...

```javascript
withinRange(aNumberRange) {
  return (aNumberRange.low >= this._temperatureRange.low) &&
    (aNumberRange.high <= this._temperatureRange.high);
}
```

caller...

```javascript
if (!aPlan.withinRange(aRoom.daysTempRange))
  alerts.push("room temperature went outside range");
```

## Example: A Variation to Create the New Function

In the above example, I wrote the code for the new function directly. Most of the time, that's simple and the easiest way to go. But there is a variation that's occasionally useful -- which can allow composing the new function entirely from refactorings.

I start with a caller of the existing function.

caller...

```javascript
const low = aRoom.daysTempRange.low;
const high = aRoom.daysTempRange.high;
if (!aPlan.withinRange(low, high))
  alerts.push("room temperature went outside range");
```

I want to rearrange the code so I can create the new function by using Extract Function on some existing code. The caller code isn't quite there yet, but I can get there by using Extract Variable a few times. First, I disentangle the call to the old function from the conditional.

caller...

```javascript
const low = aRoom.daysTempRange.low;
const high = aRoom.daysTempRange.high;
const isWithinRange = aPlan.withinRange(low, high);
if (!isWithinRange)
  alerts.push("room temperature went outside range");
```

I then extract the input parameter.

caller...

```javascript
const tempRange = aRoom.daysTempRange;
const low = tempRange.low;
const high = tempRange.high;
const isWithinRange = aPlan.withinRange(low, high);
if (!isWithinRange)
  alerts.push("room temperature went outside range");
```

With that done, I can now use Extract Function to create the new function.

caller...

```javascript
const tempRange = aRoom.daysTempRange;
const isWithinRange = xxNEWwithinRange(aPlan, tempRange);
if (!isWithinRange)
  alerts.push("room temperature went outside range");
```

top level...

```javascript
function xxNEWwithinRange(aPlan, tempRange) {
  const low = tempRange.low;
  const high = tempRange.high;
  const isWithinRange = aPlan.withinRange(low, high);
  return isWithinRange;
}
```

Since the original function is in a different context (the `HeatingPlan` class), I need to use Move Function.

caller...

```javascript
const tempRange = aRoom.daysTempRange;
const isWithinRange = aPlan.xxNEWwithinRange(tempRange);
if (!isWithinRange)
  alerts.push("room temperature went outside range");
```

class HeatingPlan...

```javascript
xxNEWwithinRange(tempRange) {
 const low = tempRange.low;
 const high = tempRange.high;
 const isWithinRange = this.withinRange(low, high);
 return isWithinRange;
}
```

I then continue as before, replacing other callers and inlining the old function into the new one. I would also inline the variables I extracted to provide the clean separation for extracting the new function.

Because this variation is entirely composed of refactorings, it's particularly handy when I have a refactoring tool with robust extract and inline operations.
