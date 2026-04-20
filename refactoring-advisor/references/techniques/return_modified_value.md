# Return Modified Value

```
let totalAscent = 0;
calculateAscent();

function calculateAscent() {
  for (let i = 1; i < points.length; i++) {
    const verticalChange = points[i].elevation - points[i-1].elevation;
    totalAscent += (verticalChange > 0) ? verticalChange : 0;
  }
}
```

```
const totalAscent = calculateAscent();

function calculateAscent() {
  let result = 0;
  for (let i = 1; i < points.length; i++) {
    const verticalChange = points[i].elevation - points[i-1].elevation;
    result += (verticalChange > 0) ? verticalChange : 0;
  }
  return result;
}
```

## Motivation

Changes a function that updates a variable via side effect to instead return the new value, making the data flow explicit in the calling code. Works best with functions that calculate a single value; not effective for functions coordinating multiple updates.

## Mechanics

- Return the modified variable from the function and assign it to that variable in the caller.

- Test.
- Declare the returned value within the modifying function.

If you want to check that things are happening as expected, alter the value of the initialization in the caller -- it will now be ignored.

- Test.
- Combine the declaration with the calculation.

If the language supports it, mark the variable as non-modifiable.

- Test.
- Rename the variable in the sub-function to reflect its new role.

- Test.

## Example

I have some code that calculates various values for a list of GPS trackpoints.

```
let totalAscent = 0;
let totalTime = 0;
let totalDistance = 0;
calculateAscent();
calculateTime();
calculateDistance();
const pace = totalTime / 60 /  totalDistance ;
```

For this refactoring, I'm just going to look at how it calculates the ascent.

```
function calculateAscent() {
  for (let i = 1; i < points.length; i++) {
    const verticalChange = points[i].elevation - points[i-1].elevation;
    totalAscent += (verticalChange > 0) ? verticalChange : 0;
  }
}
```

Burying the update to `totalAscent` within `calculateAscent` hides the linkage between `calculateAscent` and its environment. I prefer the update to be more explicit.

I begin by returning the total and showing the assignment in the caller.

```
let totalAscent = 0;
let totalTime = 0;
let totalDistance = 0;
totalAscent = calculateAscent();
calculateTime();
calculateDistance();
const pace = totalTime / 60 /  totalDistance ;
```

```
function calculateAscent() {
  for (let i = 1; i < points.length; i++) {
    const verticalChange = points[i].elevation - points[i-1].elevation;
    totalAscent += (verticalChange > 0) ? verticalChange : 0;
  }
  return totalAscent;
}
```

I then declare `totalAscent` within `calculateAscent`. This will make a new variable that shadows the one in the parent code.

```
function calculateAscent() {
  let totalAscent = 0;
  for (let i = 1; i < points.length; i++) {
    const verticalChange = points[i].elevation - points[i-1].elevation;
    totalAscent += (verticalChange > 0) ? verticalChange : 0;
  }
  return totalAscent;
}
```

I change its name to match my usual convention.

```
function calculateAscent() {
  let result = 0;
  for (let i = 1; i < points.length; i++) {
    const verticalChange = points[i].elevation - points[i-1].elevation;
    result += (verticalChange > 0) ? verticalChange : 0;
  }
  return result;
}
```

I then fold the calculation into the declaration, and change the declaration to a `const`.

```
const totalAscent = calculateAscent();
let totalTime = 0;
let totalDistance = 0;
calculateTime();
calculateDistance();
const pace = totalTime / 60 /  totalDistance ;
```

If I repeat this with the other functions, the calling code ends up like this:

```
const totalAscent = calculateAscent();
const totalTime = calculateTime();
const totalDistance = calculateDistance();
const pace = totalTime / 60 /  totalDistance ;
```
