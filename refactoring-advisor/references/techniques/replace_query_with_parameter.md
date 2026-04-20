# Replace Query with Parameter

## Motivation

Replaces an internal reference (e.g., to a global variable or module-scoped element) with a parameter, shifting resolution responsibility to the caller to remove an unwanted dependency or achieve referential transparency. This complicates callers, so only apply when the decoupling benefit clearly outweighs the cost.

## Mechanics

- Use Extract Variable on the query code to separate it from the rest of the function body.
- Apply Extract Function to the body code that isn't the call to the query.

  Give the new function an easily searchable name, for later renaming.

- Use Inline Variable to get rid of the variable you just created.
- Apply Inline Function to the original function.
- Rename the new function to that of the original.

## Example

Consider a simple control system for temperature. It allows the user to select a temperature on a thermostat -- but only sets the target temperature within a range determined by a heating plan.

class HeatingPlan...

```
get targetTemperature() {
  if      (thermostat.selectedTemperature >  this._max) return this._max;
  else if (thermostat.selectedTemperature <  this._min) return this._min;
  else return thermostat.selectedTemperature;
}
```

caller...

```
if      (thePlan.targetTemperature > thermostat.currentTemperature) setToHeat();
else if (thePlan.targetTemperature < thermostat.currentTemperature) setToCool();
else setOff();
```

The `targetTemperature` function has a dependency on a global thermostat object. I can break this dependency by moving it to a parameter.

My first step is to use Extract Variable on the parameter that I want to have in my function.

class HeatingPlan...

```
get targetTemperature() {
  const selectedTemperature = thermostat.selectedTemperature;
  if      (selectedTemperature >  this._max) return this._max;
  else if (selectedTemperature <  this._min) return this._min;
  else return selectedTemperature;
}
```

That makes it easy to apply Extract Function on the entire body of the function except for the bit that figures out the parameter.

class HeatingPlan...

```
get targetTemperature() {
  const selectedTemperature = thermostat.selectedTemperature;
  return this.xxNEWtargetTemperature(selectedTemperature);
}


xxNEWtargetTemperature(selectedTemperature) {
  if      (selectedTemperature >  this._max) return this._max;
  else if (selectedTemperature <  this._min) return this._min;
  else return selectedTemperature;
}
```

I then inline the variable I just extracted, which leaves the function as a simple call.

class HeatingPlan...

```
get targetTemperature() {
  return this.xxNEWtargetTemperature(thermostat.selectedTemperature);
}
```

I can now use Inline Function on this method.

caller...

```
if      (thePlan.xxNEWtargetTemperature(thermostat.selectedTemperature) >
         thermostat.currentTemperature)
  setToHeat();
else if (thePlan.xxNEWtargetTemperature(thermostat.selectedTemperature) <
         thermostat.currentTemperature)
  setToCool();
else
  setOff();
```

I take advantage of the easily searchable name of the new function to rename it by removing the prefix.

caller...

```
if      (thePlan.targetTemperature(thermostat.selectedTemperature) >
         thermostat.currentTemperature)
  setToHeat();
else if (thePlan.targetTemperature(thermostat.selectedTemperature) <
         thermostat.currentTemperature)
  setToCool();
else
  setOff();
```

class HeatingPlan...

```
targetTemperature(selectedTemperature) {
  if      (selectedTemperature >  this._max) return this._max;
  else if (selectedTemperature <  this._min) return this._min;
  else return selectedTemperature;
}
```

The calling code looks more unwieldy than before. Moving a dependency out of a module pushes the responsibility of dealing with that dependency back to the caller. That's the trade-off for the reduced coupling.

But removing the coupling to the thermostat object isn't the only gain. The `HeatingPlan` class is immutable -- its fields are set in the constructor with no methods to alter them. By moving the thermostat reference out of the function body I've also made `targetTemperature` referentially transparent. Every time I call `targetTemperature` on the same object, with the same argument, I will get the same result. If all methods of the heating plan have referential transparency, that makes the class much easier to test and reason about.
