# Introduce Parameter Object

```
function amountInvoiced(startDate, endDate) {...}
function amountReceived(startDate, endDate) {...}
function amountOverdue(startDate, endDate) {...}
```

```
function amountInvoiced(aDateRange) {...}
function amountReceived(aDateRange) {...}
function amountOverdue(aDateRange) {...}
```

## Motivation

Replaces a group of parameters that regularly travel together with a single data structure, making the relationship between the items explicit and reducing parameter list size.

## Mechanics

- If there isn't a suitable structure already, create one.

  Prefer a class, as that makes it easier to group behavior later on. Usually make these value objects.

- Test.

- Use Change Function Declaration to add a parameter for the new structure.

- Test.

- Adjust each caller to pass in the correct instance of the new structure. Test after each one.

- For each element of the new structure, replace the use of the original parameter with the element of the structure. Remove the parameter. Test.

## Example

Code that looks at temperature readings and determines whether any fall outside an operating range. The data:

```js
const station = { name: "ZB1",
                  readings: [
                    {temp: 47, time: "2016-11-10 09:10"},
                    {temp: 53, time: "2016-11-10 09:20"},
                    {temp: 58, time: "2016-11-10 09:30"},
                    {temp: 53, time: "2016-11-10 09:40"},
                    {temp: 51, time: "2016-11-10 09:50"},
                  ]
                };
```

A function to find readings outside a temperature range:

```js
function readingsOutsideRange(station, min, max) {
  return station.readings
    .filter(r => r.temp < min || r.temp > max);
}
```

Called from code like this:

caller

```js
alerts = readingsOutsideRange(station,
                              operatingPlan.temperatureFloor,
                              operatingPlan.temperatureCeiling);
```

The calling code pulls two data items as a pair from another object. A range like this is a common case where two separate data items are better combined into a single object. Declare a class for the combined data:

```js
class NumberRange {
  constructor(min, max) {
    this._data = {min: min, max: max};
  }
  get min() {return this._data.min;}
  get max() {return this._data.max;}
}
```

Using a class (rather than a plain object) makes it easier to move behavior later. No update methods -- this will likely be a value object.

Use Change Function Declaration to add the new object as a parameter:

```js
function readingsOutsideRange(station, min, max, range) {
  return station.readings
    .filter(r => r.temp < min || r.temp > max);
}
```

Adjust each caller to pass in the correct range. No behavior changes yet:

caller

```js
const range = new NumberRange(operatingPlan.temperatureFloor, operatingPlan.temperatureCeiling);
alerts = readingsOutsideRange(station,
                              operatingPlan.temperatureFloor,
                              operatingPlan.temperatureCeiling,
                              range);
```

Now start replacing usage of parameters. Start with the maximum:

```js
function readingsOutsideRange(station, min, max, range) {
  return station.readings
    .filter(r => r.temp < min || r.temp > range.max);
}
```

Test, then remove the other parameter:

```js
function readingsOutsideRange(station, min, range) {
  return station.readings
    .filter(r => r.temp < range.min || r.temp > range.max);
}
```

caller

```js
const range = new NumberRange(operatingPlan.temperatureFloor, operatingPlan.temperatureCeiling);
alerts = readingsOutsideRange(station,
                              operatingPlan.temperatureFloor,
                              range);
```

That completes this refactoring. The great benefit of creating a class is that you can then move behavior into it. In this case, add a method that tests if a value falls within the range:

```js
function readingsOutsideRange(station, range) {
  return station.readings
    .filter(r => !range.contains(r.temp));
}
```

class NumberRange...

```js
contains(arg) {return (arg >= this.min && arg <= this.max);}
```

This is the first step to creating a range that can take on a lot of useful behavior. Once identified, you can look for other max/min pairs to replace with a range, and move more useful behavior into the range class. One of the first things to add is a value-based equality method to make it a true value object.
